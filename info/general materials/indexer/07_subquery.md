# SubQuery

## Что это
**SubQuery** — индексатор, чья главная отличительная черта — **широта поддерживаемых сетей**: помимо всех обычных EVM-совместимых сетей, SubQuery поддерживает значительное число **не-EVM** экосистем — в первую очередь **Polkadot/Substrate** (что не случайно: у проекта исторические корни именно в экосистеме Polkadot) и **Cosmos**-совместимые сети, а также Algorand, NEAR, Stellar и другие. Если проект работает в мультичейн-среде, где часть сетей вообще не EVM — SubQuery почти всегда попадает в список первых кандидатов, так как многие конкуренты (Ponder, Envio) сфокусированы именно на EVM.

## Установка
```bash
npm install -g @subql/cli
subql init my-subquery-project
```
При инициализации мастер спросит: какую сеть индексировать (EVM, Substrate/Polkadot, Cosmos и т.д.) — от этого выбора зависит, какой именно шаблон проекта будет сгенерирован.

## Структура проекта (EVM-вариант)
```
my-subquery-project/
├── project.yaml         # манифест: сеть, контракты, обработчики
├── schema.graphql         # сущности (аналогично The Graph)
├── src/
│   └── mappings/
│       └── mappingHandlers.ts
└── abis/
    └── erc20.json
```

## project.yaml — манифест
```yaml
specVersion: 1.0.0
name: my-erc20-subquery
network:
  chainId: "1"
  endpoint: ["https://rpc.ankr.com/eth"]
dataSources:
  - kind: ethereum/Runtime
    startBlock: 18000000
    options:
      abi: erc20
      address: "0xАдресКонтракта"
    assets:
      erc20:
        file: ./abis/erc20.json
    mapping:
      file: ./dist/index.js
      handlers:
        - handler: handleTransfer
          kind: ethereum/LogHandler
          filter:
            topics:
              - Transfer(address indexed from, address indexed to, uint256 value)
```

## schema.graphql
```graphql
type Transfer @entity {
  id: ID!
  from: String!
  to: String!
  value: BigInt!
  blockNumber: BigInt!
}

type Account @entity {
  id: ID!
  balance: BigInt!
}
```

## Обработчик — mappingHandlers.ts (TypeScript)
```typescript
import { Transfer, Account } from "../types";
import { TransferLog } from "../types/abi-interfaces/Erc20";

export async function handleTransfer(log: TransferLog): Promise<void> {
    const { from, to, value } = log.args;

    const transfer = Transfer.create({
        id: `${log.transactionHash}-${log.logIndex}`,
        from,
        to,
        value: value.toBigInt(),
        blockNumber: BigInt(log.blockNumber),
    });
    await transfer.save();

    let fromAccount = await Account.get(from);
    if (!fromAccount) {
        fromAccount = Account.create({ id: from, balance: BigInt(0) });
    }
    fromAccount.balance -= value.toBigInt();
    await fromAccount.save();

    let toAccount = await Account.get(to);
    if (!toAccount) {
        toAccount = Account.create({ id: to, balance: BigInt(0) });
    }
    toAccount.balance += value.toBigInt();
    await toAccount.save();
}
```
Обработчики SubQuery пишутся на обычном TypeScript (не AssemblyScript) — синтаксически похоже на Envio/Ponder, но с собственным набором сгенерированных типов сущностей (`Transfer.create()`/`.save()` вместо `context.db.insert()`).

## Пример обработчика для НЕ-EVM сети (Substrate/Polkadot)
```typescript
import { SubstrateEvent } from "@subql/types";
import { Transfer } from "../types";

export async function handleSubstrateTransfer(event: SubstrateEvent): Promise<void> {
    const { event: { data: [from, to, amount] } } = event;

    const transfer = Transfer.create({
        id: `${event.extrinsic.extrinsic.hash.toString()}`,
        from: from.toString(),
        to: to.toString(),
        value: BigInt(amount.toString()),
    });
    await transfer.save();
}
```
Это ключевая иллюстрация универсальности SubQuery: логика обработчика для событий Substrate концептуально та же самая, что и для EVM-логов, но источник данных совершенно другой (не логи Ethereum, а события Substrate-рантайма) — SubQuery абстрагирует эту разницу под единый интерфейс SDK.

## Запуск
```bash
subql codegen     # генерация типов из schema.graphql
subql build        # компиляция обработчиков

# Self-host через Docker
docker compose up -d
```

## Managed-хостинг: SubQuery Network
Аналогично The Graph, у SubQuery есть собственная децентрализованная сеть хостинга (SubQuery Network), куда можно опубликовать проект без самостоятельного поднятия инфраструктуры — концептуально похоже на выбор "The Graph decentralized network vs Goldsky managed" из предыдущих конспектов.

## Плюсы и минусы SubQuery
| Плюсы | Минусы |
|---|---|
| Самый широкий охват сетей, включая крупные не-EVM экосистемы | Для чисто EVM-проектов может быть избыточно универсальным по сравнению с узкоспециализированными Ponder/Envio |
| TypeScript-обработчики, знакомый синтаксис | Сообщество меньше, чем у The Graph, документация местами менее подробна для экзотических сетей |
| Есть и self-host, и managed decentralized-сеть | Разные "источники данных" под разные экосистемы имеют разный API — придётся разбираться отдельно под каждую |

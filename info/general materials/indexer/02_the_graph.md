# The Graph

## Что это и почему "родоначальник подхода"
**The Graph** — первый широко распространившийся протокол индексации блокчейн-данных, задавший стандарт, на который до сих пор ориентируются почти все остальные инструменты этого раздела: описать события контракта → написать обработчики → получить готовый GraphQL API. Единица индексации в The Graph называется **subgraph (субграф)**.

## Архитектура субграфа
```
subgraph.yaml (манифест)     — какие контракты и события слушать
        │
schema.graphql                — какие сущности и поля хранить
        │
mapping.ts (AssemblyScript)   — что делать при каждом событии
        │
        ▼
   Graph Node (индексирует и хранит данные)
        │
        ▼
   GraphQL API — запросы от фронтенда
```

## Установка инструментов
```bash
npm install -g @graphprotocol/graph-cli
```

## Шаг 1. Инициализация субграфа
```bash
graph init --from-contract 0xАдресКонтракта --network mainnet my-erc20-subgraph
```
Создаёт три ключевых файла: `subgraph.yaml`, `schema.graphql`, `src/mapping.ts` — CLI сам подтягивает ABI контракта, если он верифицирован в блокчейн-эксплорере.

## Шаг 2. Манифест — subgraph.yaml
```yaml
specVersion: 0.0.5
schema:
  file: ./schema.graphql
dataSources:
  - kind: ethereum
    name: ERC20Token
    network: mainnet
    source:
      address: "0xАдресКонтракта"
      abi: ERC20
      startBlock: 18000000
    mapping:
      kind: ethereum/events
      apiVersion: 0.0.7
      language: wasm/assemblyscript
      entities:
        - Transfer
      abis:
        - name: ERC20
          file: ./abis/ERC20.json
      eventHandlers:
        - event: Transfer(indexed address,indexed address,uint256)
          handler: handleTransfer
      file: ./src/mapping.ts
```
- `startBlock` — с какого блока начинать индексацию (важно указать реальный блок деплоя контракта, а не 0 — иначе синхронизация займёт неоправданно много времени).
- `eventHandlers` — связка "какое событие → какая функция-обработчик".

## Шаг 3. Схема данных — schema.graphql
```graphql
type Transfer @entity(immutable: true) {
  id: Bytes!
  from: Bytes!
  to: Bytes!
  value: BigInt!
  blockNumber: BigInt!
  blockTimestamp: BigInt!
  transactionHash: Bytes!
}

type Account @entity {
  id: Bytes!
  balance: BigInt!
  transfersSent: [Transfer!]! @derivedFrom(field: "from")
}
```
`@entity(immutable: true)` — оптимизация: сущность, которая никогда не изменяется после создания (типично для записей о событиях), индексируется быстрее. `@derivedFrom` — обратная связь, вычисляемая автоматически, без дублирования данных.

## Шаг 4. Обработчик события — src/mapping.ts (AssemblyScript)
```typescript
import { Transfer as TransferEvent } from "../generated/ERC20Token/ERC20"
import { Transfer, Account } from "../generated/schema"
import { BigInt } from "@graphprotocol/graph-ts"

export function handleTransfer(event: TransferEvent): void {
    // Создаём новую запись о переводе — id составляется из хэша транзакции и индекса лога
    let transfer = new Transfer(
        event.transaction.hash.concatI32(event.logIndex.toI32())
    )

    transfer.from = event.params.from
    transfer.to = event.params.to
    transfer.value = event.params.value
    transfer.blockNumber = event.block.number
    transfer.blockTimestamp = event.block.timestamp
    transfer.transactionHash = event.transaction.hash
    transfer.save()

    // Обновляем агрегированный баланс отправителя
    let fromAccount = Account.load(event.params.from)
    if (fromAccount == null) {
        fromAccount = new Account(event.params.from)
        fromAccount.balance = BigInt.fromI32(0)
    }
    fromAccount.balance = fromAccount.balance.minus(event.params.value)
    fromAccount.save()

    // Обновляем агрегированный баланс получателя
    let toAccount = Account.load(event.params.to)
    if (toAccount == null) {
        toAccount = new Account(event.params.to)
        toAccount.balance = BigInt.fromI32(0)
    }
    toAccount.balance = toAccount.balance.plus(event.params.value)
    toAccount.save()
}
```
**AssemblyScript** — язык, синтаксически похожий на TypeScript, но компилируется в WebAssembly и исполняется в песочнице (sandbox) Graph Node — это осознанное архитектурное ограничение The Graph ради безопасности и детерминированности, но означает, что нельзя использовать привычные npm-пакеты как в обычном Node.js-коде.

## Шаг 5. Сборка и деплой
```bash
graph codegen     # генерирует TypeScript/AssemblyScript типы из ABI и schema.graphql
graph build        # компилирует mapping.ts в WebAssembly

# Деплой на децентрализованную сеть The Graph
graph deploy --node https://api.thegraph.com/deploy/ my-erc20-subgraph

# Или локально, для разработки — через Docker (graph-node)
graph deploy --node http://localhost:8020 my-erc20-subgraph
```

## Шаг 6. Запрос данных через GraphQL
```graphql
{
  transfers(first: 5, orderBy: blockTimestamp, orderDirection: desc) {
    from
    to
    value
    transactionHash
  }
  account(id: "0xВашАдрес") {
    balance
  }
}
```

## Плюсы и минусы The Graph
| Плюсы | Минусы |
|---|---|
| Индустриальный стандарт — множество готовых, переиспользуемых субграфов | AssemblyScript — отдельный язык, ограниченная экосистема пакетов внутри mapping |
| Децентрализованная сеть индексаторов (можно не поднимать свою инфраструктуру) | Историческая синхронизация больших субграфов может быть медленнее конкурентов (Envio) |
| Огромное сообщество и документация | Оплата запросов к децентрализованной сети идёт в токене GRT |

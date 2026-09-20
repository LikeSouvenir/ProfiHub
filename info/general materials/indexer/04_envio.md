# Envio (HyperIndex / HyperSync)

## Что это
**Envio** — индексатор, который в независимых бенчмарках регулярно показывает самую быструю первичную синхронизацию (backfill) среди всех инструментов этого раздела. Состоит из двух связанных частей:
- **HyperSync** — собственный, крайне быстрый слой доступа к историческим блокчейн-данным (альтернатива обычному RPC, оптимизированная именно под массовое чтение логов событий).
- **HyperIndex** — сам фреймворк индексации: конфигурация + обработчики на TypeScript, использующие HyperSync "под капотом" для скорости.

## Почему это так быстро
Обычный индексатор через стандартный RPC вынужден делать много отдельных запросов (`eth_getLogs` пачками блоков, с ограничениями провайдера на размер диапазона). HyperSync — это отдельная, специально оптимизированная инфраструктура, хранящая уже проиндексированные "сырые" данные блокчейна в удобном для массового чтения формате — HyperIndex читает данные оттуда, а не гоняет тысячи мелких RPC-запросов.

## Установка
```bash
npx envio init
```
Интерактивный мастер спросит: имя проекта, сеть, контракт (по адресу — Envio сам подтянет ABI, если контракт верифицирован) и язык (TypeScript, ReScript или JavaScript — в этом курсе используется TypeScript).

## Структура проекта
```
my-envio-indexer/
├── config.yaml           # какие контракты/события/сети слушать
├── schema.graphql         # сущности (аналогично The Graph)
├── src/
│   └── EventHandlers.ts   # обработчики на обычном TypeScript
└── generated/              # автогенерируемые типы
```

## config.yaml
```yaml
name: my-erc20-indexer
networks:
  - id: 1 # Ethereum mainnet
    start_block: 18000000
    contracts:
      - name: ERC20Token
        address:
          - "0xАдресКонтракта"
        handler: src/EventHandlers.ts
        events:
          - event: "Transfer(address indexed from, address indexed to, uint256 value)"
```

## schema.graphql
```graphql
type Transfer {
  id: ID!
  from: String!
  to: String!
  value: BigInt!
  blockNumber: Int!
  blockTimestamp: Int!
}

type Account {
  id: ID!
  balance: BigInt!
}
```

## Обработчик — обычный TypeScript (не AssemblyScript!)
```typescript
import { ERC20Token, Transfer, Account } from "generated";

ERC20Token.Transfer.handler(async ({ event, context }) => {
    const { from, to, value } = event.params;

    context.Transfer.set({
        id: `${event.transaction.hash}-${event.logIndex}`,
        from,
        to,
        value,
        blockNumber: event.block.number,
        blockTimestamp: event.block.timestamp,
    });

    // Обновляем баланс отправителя
    let fromAccount = await context.Account.get(from);
    if (!fromAccount) {
        fromAccount = { id: from, balance: 0n };
    }
    context.Account.set({
        ...fromAccount,
        balance: fromAccount.balance - value,
    });

    // Обновляем баланс получателя
    let toAccount = await context.Account.get(to);
    if (!toAccount) {
        toAccount = { id: to, balance: 0n };
    }
    context.Account.set({
        ...toAccount,
        balance: toAccount.balance + value,
    });
});
```
Обратите внимание: это обычный `async`/`await` TypeScript, без компиляции в WebAssembly и без ограничений песочницы AssemblyScript — можно использовать привычные npm-пакеты внутри обработчиков (в разумных пределах, не нарушая детерминированность обработки).

## Self-host через Docker
```bash
npx envio codegen     # генерация типов из config.yaml и schema.graphql
npx envio dev          # локальный запуск с hot-reload для разработки
```
```bash
# Продакшн-запуск через Docker Compose (Envio генерирует docker-compose.yaml автоматически)
docker compose up -d
```
Полный self-host — Postgres-база, сам индексатор и Hasura (для автогенерации GraphQL API поверх Postgres) поднимаются локальными контейнерами, без обязательной зависимости от облачного managed-сервиса.

## Запрос данных (через автогенерируемый Hasura GraphQL)
```graphql
{
  Transfer(limit: 5, order_by: {blockTimestamp: desc}) {
    from
    to
    value
  }
}
```

## Плюсы и минусы Envio
| Плюсы | Минусы |
|---|---|
| Самая быстрая историческая синхронизация среди сравниваемых инструментов | Моложе The Graph — меньше готовых примеров/шаблонов "из коробки" |
| Обычный TypeScript вместо AssemblyScript — ниже порог входа | Self-host требует своей инфраструктуры (Docker/Postgres), хотя есть и managed-опция |
| Полностью самостоятельный self-host через Docker без vendor lock-in | HyperSync доступен не для абсолютно всех сетей сразу (хотя список активно растёт) |

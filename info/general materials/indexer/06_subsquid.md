# Subsquid (SQD)

## Что это
**Subsquid** (проект/бренд также называется **SQD**) — индексатор, спроектированный вокруг идеи полноценного **ETL-пайплайна** (Extract, Transform, Load): данные извлекаются из блокчейна батчами, проходят через произвольную, часто довольно "тяжёлую" трансформацию и агрегацию на TypeScript, и только затем сохраняются — с прицелом на серьёзную SQL-аналитику, а не только на простое сопоставление "событие → запись в таблице".

## Архитектура: Archive + Processor
```
Subsquid Archive (историческое хранилище логов/блоков,
                  оптимизированное под пакетное чтение)
        │  батчи сырых данных
        ▼
Processor (ваш TypeScript-код — трансформация, агрегация)
        │
        ▼
   Postgres (через TypeORM) ──► GraphQL API (генерируется автоматически)
```
Аналогично Envio с его HyperSync, Subsquid тоже использует собственный оптимизированный слой чтения истории (Archive) вместо прямого перебора через RPC — это общий тренд среди "новых" индексаторов, ускоряющий именно этап backfill.

## Установка
```bash
npx sqd init my-subsquid-indexer -t evm
cd my-subsquid-indexer
npm install
```

## Структура проекта
```
my-subsquid-indexer/
├── schema.graphql          # сущности (как в The Graph, но здесь дополнительно генерирует TypeORM-модели)
├── src/
│   ├── main.ts               # точка входа процессора
│   ├── processor.ts           # конфигурация: какие события каких контрактов слушать
│   └── model/                 # автогенерируемые TypeORM-модели из schema.graphql
└── db/
    └── migrations/            # SQL-миграции базы данных
```

## schema.graphql
```graphql
type Transfer @entity {
  id: ID!
  from: String! @index
  to: String! @index
  value: BigInt!
  blockNumber: Int! @index
  timestamp: DateTime!
}
```

## processor.ts — конфигурация источника данных
```typescript
import { EvmBatchProcessor } from "@subsquid/evm-processor";

export const processor = new EvmBatchProcessor()
    .setGateway("https://v2.archive.subsquid.io/network/ethereum-mainnet")
    .setRpcEndpoint(process.env.RPC_ENDPOINT)
    .setBlockRange({ from: 18000000 })
    .addLog({
        address: ["0xАдресКонтракта"],
        topic0: [TRANSFER_TOPIC], // хэш сигнатуры события Transfer
    });
```

## main.ts — батчевая обработка (главное отличие от других инструментов)
```typescript
import { processor } from "./processor";
import { Transfer } from "./model";
import { decodeTransferEvent } from "./abi/erc20";

processor.run(new TypeormDatabase(), async (ctx) => {
    const transfers: Transfer[] = [];

    // Обратите внимание: обработка идёт ПАКЕТАМИ блоков (ctx.blocks), а не по одному событию за раз
    for (const block of ctx.blocks) {
        for (const log of block.logs) {
            const { from, to, value } = decodeTransferEvent(log);

            transfers.push(new Transfer({
                id: `${log.transactionHash}-${log.logIndex}`,
                from,
                to,
                value,
                blockNumber: block.header.height,
                timestamp: new Date(block.header.timestamp),
            }));
        }
    }

    // Один массовый insert всех накопленных за батч записей — гораздо быстрее,
    // чем сохранять по одной записи на каждое событие
    await ctx.store.insert(transfers);
});
```
Это ключевое архитектурное отличие Subsquid: обработка построена вокруг **батчей** (`ctx.blocks` — массив сразу нескольких блоков), а сохранение в базу делается одним массовым `insert`, а не отдельной операцией на каждое событие — именно этот подход даёт возможность эффективно делать "тяжёлые" агрегации над большими объёмами данных за один проход.

## Дополнительная SQL-агрегация поверх Postgres
Так как Subsquid хранит данные в обычной, полноценной Postgres-базе через TypeORM, ничто не мешает после (или во время) индексации выполнять сложные SQL-запросы напрямую, в обход GraphQL-слоя:
```sql
SELECT "from", SUM(value) as total_sent
FROM transfer
GROUP BY "from"
ORDER BY total_sent DESC
LIMIT 10;
```

## Запуск
```bash
sqd build       # сборка проекта
sqd migration:generate
sqd migration:apply    # применение SQL-миграций к базе
sqd process              # запуск процессора — начинается индексация
sqd serve                 # запуск GraphQL API поверх заполненной базы
```

## Плюсы и минусы Subsquid
| Плюсы | Минусы |
|---|---|
| Батчевая обработка эффективна для тяжёлой трансформации/агрегации больших объёмов данных | Более сложная ментальная модель по сравнению с "одно событие — один обработчик" (Ponder/Envio) |
| Полноценный Postgres + TypeORM — прямой доступ к SQL без ограничений | Порог входа выше, особенно для новичков без опыта работы с батчевой обработкой |
| Собственная сеть Archive для быстрого чтения истории | Экосистема меньше, чем у The Graph |

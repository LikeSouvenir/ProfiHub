# Ponder

## Что это
**Ponder** — индексатор, спроектированный так, чтобы работа с ним ощущалась максимально похожей на обычную full-stack TypeScript-разработку: никакого AssemblyScript, никакой децентрализованной сети, свой собственный сервер (self-host), конфигурация и обработчики — всё на TypeScript, с горячей перезагрузкой при разработке, как в привычном Node.js-проекте. Именно поэтому Ponder часто называют "любимцем" небольших и средних команд/инди-разработчиков — низкий порог входа и приятный developer experience.

## Установка
```bash
npm create ponder@latest my-ponder-indexer
cd my-ponder-indexer
npm install
```
Мастер инициализации предложит указать сеть, контракт (по адресу — Ponder тоже умеет подтягивать ABI) и стартовый блок.

## Структура проекта
```
my-ponder-indexer/
├── ponder.config.ts       # сети, контракты, RPC
├── ponder.schema.ts        # схема данных (на TypeScript, не GraphQL SDL!)
├── src/
│   └── index.ts             # обработчики событий
└── abis/
    └── ERC20Abi.ts
```

## ponder.config.ts
```typescript
import { createConfig } from "ponder";
import { ERC20Abi } from "./abis/ERC20Abi";

export default createConfig({
    networks: {
        mainnet: {
            chainId: 1,
            transport: http(process.env.PONDER_RPC_URL_1),
        },
    },
    contracts: {
        ERC20Token: {
            network: "mainnet",
            abi: ERC20Abi,
            address: "0xАдресКонтракта",
            startBlock: 18000000,
        },
    },
});
```

## ponder.schema.ts — схема на TypeScript, а не на отдельном языке описания
Ключевая особенность Ponder: схема данных описывается прямо в TS-файле через собственный builder, а не в отдельном `.graphql`-файле:
```typescript
import { onchainTable } from "ponder";

export const transfer = onchainTable("transfer", (t) => ({
    id: t.text().primaryKey(),
    from: t.hex().notNull(),
    to: t.hex().notNull(),
    value: t.bigint().notNull(),
    blockNumber: t.bigint().notNull(),
    blockTimestamp: t.bigint().notNull(),
}));

export const account = onchainTable("account", (t) => ({
    id: t.hex().primaryKey(),
    balance: t.bigint().notNull(),
}));
```

## Обработчик события — src/index.ts
```typescript
import { ponder } from "ponder:registry";
import { transfer, account } from "ponder:schema";

ponder.on("ERC20Token:Transfer", async ({ event, context }) => {
    const { from, to, value } = event.args;

    await context.db.insert(transfer).values({
        id: `${event.transaction.hash}-${event.log.logIndex}`,
        from,
        to,
        value,
        blockNumber: event.block.number,
        blockTimestamp: event.block.timestamp,
    });

    // upsert — создать запись, если нет, или обновить существующую
    await context.db
        .insert(account)
        .values({ id: from, balance: -value })
        .onConflictDoUpdate((row) => ({ balance: row.balance - value }));

    await context.db
        .insert(account)
        .values({ id: to, balance: value })
        .onConflictDoUpdate((row) => ({ balance: row.balance + value }));
});
```
Синтаксис `context.db.insert(...).onConflictDoUpdate(...)` — это, по сути, обёртка над Drizzle ORM: Ponder использует привычные концепции из мира обычной бэкенд-разработки на TypeScript, а не изобретает собственный язык запросов.

## Запуск
```bash
npm run dev      # локальная разработка с hot-reload
npm run start     # продакшн-режим
```
Ponder поднимает собственный HTTP-сервер с GraphQL API (по умолчанию `http://localhost:42069/graphql`) — никакой отдельной ноды индексации (как Graph Node) устанавливать не требуется, весь процесс — это один Node.js-процесс.

## Запрос данных
```graphql
{
  transfers(orderBy: "blockTimestamp", orderDirection: "desc", limit: 5) {
    items {
      from
      to
      value
    }
  }
}
```

## Также доступен прямой SQL/REST через Ponder API
```typescript
import { db } from "ponder:api";

app.get("/top-holders", async (c) => {
    const holders = await db.select().from(account).orderBy(desc(account.balance)).limit(10);
    return c.json(holders);
});
```
Ponder позволяет писать собственные произвольные HTTP-эндпоинты прямо поверх той же базы данных — то есть индексатор одновременно может выступать как небольшой бэкенд-сервер, а не только источник GraphQL-схемы.

## Плюсы и минусы Ponder
| Плюсы | Минусы |
|---|---|
| Полностью TypeScript-native опыт разработки — знакомые паттерны (ORM, hot-reload) | Нет децентрализованной сети хостинга — только self-host |
| Можно писать произвольную бэкенд-логику (не только GraphQL) поверх тех же данных | Моложе The Graph, меньше готовых, "коробочных" интеграций |
| Быстрая, приятная разработка "из коробки" | Требуется собственная инфраструктура для продакшена (сервер + Postgres) |

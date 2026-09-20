# Практика: индексатор для контракта TaskLog

## Задача

Построить индексатор для контракта `TaskLog` (тот же, что использовался в практике модуля 5 направления Frontend), а затем запросить данные из React-приложения.

## Контракт

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract TaskLog {
    struct Entry {
        string title;
        uint256 points;
        address author;
        uint256 timestamp;
    }

    Entry[] private entries;
    mapping(address => uint256) public solvedCount;

    event EntryAdded(address indexed author, string title, uint256 points);

    function addEntry(string calldata _title, uint256 _points) external {
        entries.push(Entry(_title, _points, msg.sender, block.timestamp));
        solvedCount[msg.sender] += 1;
        emit EntryAdded(msg.sender, _title, _points);
    }
}
```

Индексировать будем событие `EntryAdded`. Цель — получить две таблицы: список всех записей и агрегат по авторам.

---

## Шаг 1. Создание проекта

```bash
pnpx envio init
```

Ответы на вопросы:

```
? Choose an initialization option  →  Contract Import
? Block explorer or local abi?     →  Local ABI
? Path to json abi file?           →  ./abis/TaskLog.json
? Which events to index?           →  [x] EntryAdded(address indexed author, string title, uint256 points)
? Choose network                   →  ваша сеть (или Custom Network ID для локальной)
? Name of this contract            →  TaskLog
? Address of the contract          →  0xВашАдрес
```

ABI контракта заранее сохраните в `./abis/TaskLog.json`.

---

## Шаг 2. config.yaml

```yaml
# yaml-language-server: $schema=./node_modules/envio/evm.schema.json
name: tasklog-indexer
description: Индексатор журнала заданий чемпионата

contracts:
  - name: TaskLog
    events:
      - event: "EntryAdded(address indexed author, string title, uint256 points)"
        field_selection:
          # Метка времени блока нужна, чтобы показывать дату записи
          block_fields:
            - timestamp
          transaction_fields:
            - hash

chains:
  - id: ${ENVIO_CHAIN_ID:-11155111}     # Sepolia по умолчанию
    start_block: ${ENVIO_START_BLOCK:-0}
    contracts:
      - name: TaskLog
        address: "${ENVIO_TASKLOG_ADDRESS}"
```

---

## Шаг 3. schema.graphql

```graphql
"Одна запись журнала — одно событие EntryAdded"
type Entry {
    id: ID!
    title: String!
    points: BigInt!
    author: Author!
    timestamp: BigInt!
    blockNumber: BigInt!
    txHash: String!
}

"Агрегат по автору: считается обработчиком, в контракте такого нет"
type Author {
    id: ID!
    address: String!
    entriesCount: Int!
    totalPoints: BigInt!
    firstEntryAt: BigInt!
    lastEntryAt: BigInt!
    entries: [Entry!]!
}

"Глобальная статистика — одна запись с фиксированным id"
type GlobalStats {
    id: ID!
    totalEntries: Int!
    totalPoints: BigInt!
    uniqueAuthors: Int!
}
```

Обратите внимание: `Author` и `GlobalStats` в контракте не существуют. Индексатор их **вычисляет** — в этом и смысл, ончейн такие агрегаты считать дорого.

После правки схемы:

```bash
pnpm codegen
```

---

## Шаг 4. Обработчик

`src/handlers/TaskLog.ts`:

```ts
import { indexer } from "envio";

const GLOBAL_STATS_ID = "global"; // единственная запись статистики

indexer.onEvent(
    {
        contract: "TaskLog",
        event: "EntryAdded",
        fields: { block: ["timestamp"], transaction: ["hash"] },
    },
    async ({ event, context }) => {
        const authorId = event.params.author;
        const timestamp = BigInt(event.block.timestamp);

        // ----- Загрузка существующих сущностей -----
        // Все get вызываются в начале обработчика, чтобы попасть
        // в фазу предзагрузки и уйти в БД одним пакетом
        const author = await context.Author.get(authorId);
        const stats = await context.GlobalStats.get(GLOBAL_STATS_ID);

        // ----- 1. Запись журнала -----
        // id из хеша транзакции и индекса лога уникален глобально
        const entryId = `${event.transaction.hash}-${event.logIndex}`;

        context.Entry.set({
            id: entryId,
            title: event.params.title,
            points: event.params.points,
            author_id: authorId,        // связь задаётся через <поле>_id
            timestamp,
            blockNumber: BigInt(event.block.number),
            txHash: event.transaction.hash,
        });

        // ----- 2. Агрегат по автору -----
        const isNewAuthor = author === undefined;

        context.Author.set({
            id: authorId,
            address: authorId,
            entriesCount: (author?.entriesCount ?? 0) + 1,
            totalPoints: (author?.totalPoints ?? 0n) + event.params.points,
            firstEntryAt: author?.firstEntryAt ?? timestamp,
            lastEntryAt: timestamp,
        });

        // ----- 3. Глобальная статистика -----
        context.GlobalStats.set({
            id: GLOBAL_STATS_ID,
            totalEntries: (stats?.totalEntries ?? 0) + 1,
            totalPoints: (stats?.totalPoints ?? 0n) + event.params.points,
            uniqueAuthors: (stats?.uniqueAuthors ?? 0) + (isNewAuthor ? 1 : 0),
        });
    },
);
```

**Разбор ключевых моментов:**

1. **`?? 0n`, а не `?? 0`.** Поля типа `BigInt` в схеме требуют `BigInt` в коде. Смешение `BigInt` и `Number` в арифметике бросает `TypeError`.
2. **`author_id`, а не `author`.** В связь записывается идентификатор, а не объект. Envio подставит объект при GraphQL-запросе.
3. **Все `get` в начале.** Предзагрузка собирает их в один пакетный запрос — это основной источник производительности.
4. **Детерминированный `id`.** Хеш транзакции плюс индекс лога уникальны и не зависят от порядка обработки, в отличие от счётчика.
5. **Нет `await` у `set`.** Запись идёт в память, на диск попадает пачкой.

---

## Шаг 5. Запуск

```bash
ENVIO_TASKLOG_ADDRESS=0xВашАдрес pnpm dev
```

Откроется панель Hasura, пароль — `testing`. Проверьте в ней, что таблицы наполняются.

---

## Шаг 6. Запросы

```graphql
query Dashboard {
    # Последние 10 записей
    Entry(order_by: { timestamp: desc }, limit: 10) {
        id
        title
        points
        timestamp
        author {
            address
            entriesCount
        }
    }

    # Топ авторов по баллам
    Author(order_by: { totalPoints: desc }, limit: 5) {
        address
        entriesCount
        totalPoints
    }

    # Общая статистика
    GlobalStats {
        totalEntries
        totalPoints
        uniqueAuthors
    }
}
```

---

## Шаг 7. Подключение из React

```jsx
import { useState, useEffect } from "react";

const ENDPOINT = "http://localhost:8080/v1/graphql";

const QUERY = `
    query Leaderboard {
        Author(order_by: { totalPoints: desc }, limit: 10) {
            address
            entriesCount
            totalPoints
        }
    }
`;

function Leaderboard() {
    const [authors, setAuthors] = useState([]);
    const [isLoading, setIsLoading] = useState(true);
    const [error, setError] = useState(null);

    useEffect(() => {
        let cancelled = false;

        async function load() {
            try {
                const response = await fetch(ENDPOINT, {
                    method: "POST",
                    headers: {
                        "Content-Type": "application/json",
                        "x-hasura-admin-secret": "testing", // только для локальной разработки
                    },
                    body: JSON.stringify({ query: QUERY }),
                });

                const json = await response.json();

                // GraphQL отвечает статусом 200 даже на ошибку —
                // проверять нужно поле errors
                if (json.errors) {
                    throw new Error(json.errors[0].message);
                }

                if (!cancelled) setAuthors(json.data.Author);
            } catch (err) {
                if (!cancelled) setError(err.message);
            } finally {
                if (!cancelled) setIsLoading(false);
            }
        }

        load();
        return () => { cancelled = true; };
    }, []);

    if (isLoading) return <p>Загрузка...</p>;
    if (error) return <p>Ошибка: {error}</p>;

    return (
        <table>
            <thead>
                <tr><th>Адрес</th><th>Записей</th><th>Баллов</th></tr>
            </thead>
            <tbody>
                {authors.map((author) => (
                    <tr key={author.address}>
                        <td>{author.address.slice(0, 10)}...</td>
                        <td>{author.entriesCount}</td>
                        <td>{author.totalPoints}</td>
                    </tr>
                ))}
            </tbody>
        </table>
    );
}

export default Leaderboard;
```

Сравните с решением практики модуля 5: там за историей приходилось ходить в контракт вызов за вызовом. Здесь один GraphQL-запрос отдаёт готовый отсортированный рейтинг.

> **Про админ-секрет.** `x-hasura-admin-secret` в коде фронтенда допустим только локально. В продакшене доступ настраивается через роли Hasura или через собственный бэкенд-прокси.

---

## Дополнительные задания

1. **Мультичейн.** Задеплойте `TaskLog` во вторую сеть и добавьте её в `chains`. Обработчик менять не нужно. Не забудьте включить `event.chainId` в `id` сущностей — иначе записи из разных сетей затрут друг друга.
2. **Тесты.** Напишите тест обработчика через `pnpm test`: подайте фиктивное событие и проверьте, что `Author.totalPoints` посчитался верно. Блокчейн при этом не нужен.
3. **Фильтр.** Добавьте второй обработчик того же события с фильтром `where` — например, отдельный учёт записей дороже 30 баллов.
4. **Сравнение.** Соберите тот же индексатор на The Graph и сравните: время первичной синхронизации, объём кода, порог входа (AssemblyScript против TypeScript).

## Критерии приёмки

- `pnpm dev` запускается, таблицы в Hasura наполняются данными.
- Сумма `GlobalStats.totalPoints` совпадает с суммой баллов всех записей.
- `uniqueAuthors` не растёт при повторных записях от одного автора.
- Фронтенд получает рейтинг одним запросом и корректно обрабатывает состояние загрузки и ошибку.
- Арифметика `BigInt` нигде не смешана с `Number`.

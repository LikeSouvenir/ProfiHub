# Bitquery: GraphQL-аналитика и потоковые данные

## Чем отличается от остальных четырёх

Alchemy, Moralis, GoldRush и QuickNode отвечают на вопрос «что лежит на этом адресе». **Bitquery** отвечает на другой класс вопросов: «что происходит в целом на рынке» — сделки на DEX, объёмы торгов, топ трейдеров, движение конкретного токена между биржами. Это аналитический слой, а не просто «баланс по адресу».

Ключевое архитектурное отличие — **GraphQL** вместо REST или JSON-RPC. Один и тот же язык запроса используется и для разового чтения (`query`), и для потоковой подписки на новые события в реальном времени (`subscription`) — меняется одно слово в начале документа.

## Получение доступа

Регистрация на `account.bitquery.io` → создание токена доступа (Access Token). Запросы можно опробовать прямо в браузере через встроенную GraphQL IDE без написания кода — это быстрый способ собрать нужный запрос перед тем, как переносить его в приложение.

## Структура запроса

Схема организована по сетям (`EVM`, `Solana` и другим) и «кубам» данных внутри них (`DEXTrades`, `Transfers`, `Balances` и т.д.):

```graphql
query {
    EVM(network: eth) {
        DEXTrades(limit: { count: 5 }) {
            Trade {
                Buy {
                    Currency { Symbol }
                    Price
                    PriceInUSD
                }
                Dex { ProtocolName }
            }
        }
    }
}
```

## Пример: сделки конкретного кошелька на Uniswap

```graphql
query WalletTrades {
    EVM(network: eth) {
        DEXTrades(
            where: {
                Trade: {
                    Dex: { ProtocolName: { is: "uniswap_v3" } }
                }
                Transaction: { From: { is: "0x9d65...3EFa0" } }
            }
            limit: { count: 20 }
            orderBy: { descending: Block_Time }
        ) {
            Block { Time }
            Trade {
                Buy {
                    Currency { Symbol }
                    Amount
                    AmountInUSD
                }
                Sell {
                    Currency { Symbol }
                    Amount
                }
            }
        }
    }
}
```

Выполнение из кода — обычный POST-запрос с GraphQL-документом в теле:

```js
async function runQuery(query, apiToken) {
    const response = await fetch("https://streaming.bitquery.io/graphql", {
        method: "POST",
        headers: {
            "Content-Type": "application/json",
            Authorization: `Bearer ${apiToken}`,
        },
        body: JSON.stringify({ query }),
    });

    const { data } = await response.json();
    return data;
}
```

## Агрегации без отдельного бэкенда

Там, где для Alchemy или Moralis агрегацию (сумму, топ-N) пришлось бы считать вручную на своём сервере, GraphQL-схема Bitquery умеет отдавать её прямо в запросе:

```graphql
query TopBoughtTokens {
    EVM(network: base) {
        DEXTradeByTokens(
            orderBy: { descendingByField: "buy" }
            limit: { count: 10 }
        ) {
            Trade {
                Currency { Name Symbol SmartContract }
            }
            buy: sum(of: Trade_Side_AmountInUSD, if: { Trade: { Side: { Type: { is: buy } } } })
        }
    }
}
```

Это заменяет отдельный аналитический сервис (наподобие Dune) для несложных агрегаций — сумма, топ по объёму, фильтр по периоду выражаются прямо в GraphQL, без выгрузки сырых данных и обсчёта на своей стороне.

## Потоковые подписки (real-time)

Для «живой ленты сделок» тот же самый запрос меняет корневое слово `query` на `subscription` и открывается через WebSocket:

```js
import { createClient } from "graphql-ws";

const client = createClient({
    url: "wss://streaming.bitquery.io/graphql",
    connectionParams: {
        Authorization: `Bearer ${apiToken}`,
    },
});

const subscriptionQuery = `
    subscription {
        EVM(network: eth) {
            DEXTrades(
                where: { Trade: { Dex: { ProtocolName: { is: "uniswap_v3" } } } }
            ) {
                Trade {
                    Buy { Currency { Symbol } AmountInUSD }
                }
            }
        }
    }
`;

client.subscribe(
    { query: subscriptionQuery },
    {
        next: (data) => console.log("Новая сделка:", data),
        error: (err) => console.error(err),
        complete: () => console.log("Подписка завершена"),
    }
);
```

Практически: интерфейс «лента сделок в реальном времени» или «алерт при крупном свопе» строится без своего WebSocket-сервера и без индексатора — Bitquery берёт эту роль на себя.

## Что важно понимать про модель данных

1. **`DEXTrades` против `DEXTradeByTokens`.** Первый куб — «сделка целиком», с обеими сторонами (`Buy`/`Sell`) в одной записи; удобен для ленты активности. Второй — «запись на каждый токен отдельно», удобен для агрегаций по конкретному токену (объём торгов, топ покупателей).
2. **`dataset: realtime` против архива.** По умолчанию запрос идёт по недавним данным; для полной истории с начала сети указывается другой `dataset`, если он поддерживается сетью (для тестовых сетей архива может не быть вовсе).
3. **`network` — обязательный параметр** корневого поля `EVM(network: ...)` — без него запрос не выполнится; значения — `eth`, `bsc`, `base`, `arc_testnet` и другие.

## Ключевые моменты для практики

1. **GraphQL — это язык запроса, а не эндпоинт.** Форма запроса не меняется от того, вызывается ли он из Python, JavaScript или прямо из IDE в браузере — меняется только транспорт (HTTP для `query`, WebSocket для `subscription`).
2. **Bitquery закрывает нишу, где остальные четыре сервиса бессильны:** не «что у адреса», а «что происходит в рынке в целом» — для дашборда, который показывает объёмы торгов или ленту сделок, это единственный подходящий инструмент из всего раздела.
3. **`selectWhere`, `if`, `orderBy: { descendingByField }`** — GraphQL-схема Bitquery поддерживает довольно богатую агрегацию прямо в запросе; прежде чем писать код агрегации на фронтенде, стоит проверить, не решается ли задача одним полем в запросе.
4. **IDE в браузере — не игрушка, а часть рабочего процесса.** Запрос удобно собирать и отлаживать визуально, а затем копировать готовую GraphQL-строку в код — так делают в документации сами примеры Bitquery.

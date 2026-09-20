# GoldRush (Covalent): Unified API на 100+ сетей

## О названии

Сервис исторически назывался **Covalent**. В 2024 году продукт ребрендировался в **GoldRush**, документация переехала на `goldrush.dev`, но базовый URL самого API остался прежним — `api.covalenthq.com`. В коде, статьях и старых материалах встретятся оба названия; это один и тот же сервис.

## Главная идея: один путь для любой сети

В отличие от Alchemy и Moralis, где сеть указывается параметром запроса, у GoldRush сеть — часть URL-пути. Структура одинакова для 100+ поддерживаемых сетей (Ethereum, Solana, Base, Polygon, BNB Chain, Bitcoin и другие):

```
https://api.covalenthq.com/v1/{chain_name}/{endpoint}/
```

Это упрощает переключение между сетями: код не меняется, меняется только строка в пути.

## Получение ключа и авторизация

Регистрация на `goldrush.dev` → API Keys. Авторизация — Basic Auth, где ключ выступает логином, а пароль пустой:

```js
const headers = {
    Authorization: `Basic ${btoa(`${apiKey}:`)}`,
};
```

## Баланс кошелька

```js
async function getBalances(address, apiKey, chain = "eth-mainnet") {
    const url = `https://api.covalenthq.com/v1/${chain}/address/${address}/balances_v2/`;

    const response = await fetch(url, {
        headers: { Authorization: `Basic ${btoa(`${apiKey}:`)}` },
    });

    const { data } = await response.json();
    return data.items;
}
```

Каждый элемент уже содержит `contract_ticker_symbol`, `contract_decimals`, `quote` (стоимость в USD) — как и у Moralis, данные приходят обогащёнными, без отдельных вызовов за метаданными.

## SDK вместо ручных fetch

GoldRush предоставляет официальный клиент, который удобнее прямых HTTP-вызовов при частом использовании:

```bash
npm install @covalenthq/client-sdk
```

```js
import { CovalentClient } from "@covalenthq/client-sdk";

const client = new CovalentClient("YOUR_API_KEY");

async function getBalances(address) {
    const response = await client.BalanceService.getTokenBalancesForWalletAddress(
        "eth-mainnet",
        address
    );

    if (response.error) {
        throw new Error(response.error_message);
    }

    return response.data.items;
}
```

SDK сам собирает URL, обрабатывает пагинацию и типизирует ответ — в TypeScript-проекте это ощутимо удобнее ручных запросов.

## История транзакций

```js
async function getTransactions(address, apiKey, chain = "eth-mainnet") {
    const url = `https://api.covalenthq.com/v1/${chain}/address/${address}/transactions_v3/`;

    const response = await fetch(url, {
        headers: { Authorization: `Basic ${btoa(`${apiKey}:`)}` },
    });

    const { data } = await response.json();
    return data.items; // с декодированными логами событий и трейсами (по сетям, где доступно)
}
```

## NFT-подходы

```js
async function getNftsForAddress(address, apiKey, chain = "eth-mainnet") {
    const url = `https://api.covalenthq.com/v1/${chain}/address/${address}/balances_nft/`;

    const response = await fetch(url, {
        headers: { Authorization: `Basic ${btoa(`${apiKey}:`)}` },
    });

    const { data } = await response.json();
    return data.items;
}
```

## Проверка статуса всех сетей

Полезный служебный эндпоинт для дашбордов и мониторинга: показывает, какие сети сейчас доступны и насколько свежа индексация.

```js
async function getChainStatuses(apiKey) {
    const response = await fetch("https://api.covalenthq.com/v1/chains/status/", {
        headers: { Authorization: `Basic ${btoa(`${apiKey}:`)}` },
    });

    const { data } = await response.json();
    return data.items; // [{ name, is_testnet, synced_blocks_at_height, ... }]
}
```

## Модель оплаты: кредиты

GoldRush считает использование в **кредитах за вызов**, а не в «безлимитных запросах в секунду», как у некоторых конкурентов. Разные эндпоинты стоят разное число кредитов (например, `get-all-chain-statuses` — 1 кредит за вызов). Это делает стоимость предсказуемой: заранее известно, сколько будет стоить конкретный сценарий использования, что удобно закладывать в расчёт бюджета проекта.

## x402: оплата за отдельный запрос для AI-агентов

Из новых возможностей — поддержка протокола **x402** (HTTP 402 Payment Required): автономные агенты могут платить за каждый конкретный запрос без предварительной регистрации и API-ключа. Для командного проекта на чемпионате это не первостепенная функция, но стоит знать, что она существует — GoldRush явно позиционирует себя как инфраструктуру и для агентных сценариев, не только для классических дашбордов.

## Ключевые моменты для практики

1. **Сеть — часть URL, а не параметр.** `/v1/eth-mainnet/...` против `/v1/base-mainnet/...` — при добавлении новой сети меняется путь, а не тело запроса.
2. **Basic Auth без пароля** — частая ошибка: забыть добавить двоеточие после ключа при кодировании в base64 (`${apiKey}:`, а не просто `${apiKey}`).
3. **SDK экономит время** при работе с несколькими эндпоинтами подряд — меньше ручной сборки URL и разбора пагинации.
4. **Кредитная модель требует прикидки бюджета заранее** — в отличие от «просто лимит запросов в секунду», здесь важно оценить, сколько кредитов съест конкретный сценарий (например, загрузка полной истории транзакций стоит больше, чем один баланс).

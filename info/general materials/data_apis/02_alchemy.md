# Alchemy: Data API и Enhanced JSON-RPC

## Что предлагает Alchemy

Alchemy начинался как провайдер RPC-нод, и это чувствуется в архитектуре: данные отдаются двумя параллельными способами.

1. **Enhanced JSON-RPC методы** (`alchemy_*`) — расширение стандартного `eth_*` протокола. Работают через тот же эндпоинт ноды, что и обычные вызовы.
2. **Data API** (REST, с 2025–2026) — отдельный набор эндпоинтов `POST /data/v1/{apiKey}/...`, задуманный как замена нескольким Enhanced-методам одним унифицированным вызовом.

> **Важно.** `alchemy-sdk-js` архивируется в январе 2026. В новых проектах Alchemy рекомендует прямые HTTP-вызовы к Data API либо Viem/Ethers для обычных вызовов ноды. Примеры ниже используют `fetch` напрямую — это и проще для разбора, и не завязано на устаревающий пакет.

## Получение ключа

Регистрация на `alchemy.com` → создание приложения → выбор сети → получение API-ключа. Базовый URL для JSON-RPC:

```
https://eth-mainnet.g.alchemy.com/v2/{apiKey}
```

## Баланс токенов ERC-20

```js
async function getTokenBalances(address, apiKey) {
    const response = await fetch(`https://eth-mainnet.g.alchemy.com/v2/${apiKey}`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
            jsonrpc: "2.0",
            id: 1,
            method: "alchemy_getTokenBalances",
            // Без списка контрактов вернутся балансы всех токенов,
            // с которыми адрес когда-либо взаимодействовал
            params: [address, "erc20"],
        }),
    });

    const { result } = await response.json();
    return result.tokenBalances;
}
```

Ответ содержит адрес контракта и баланс в hex-формате (в минимальных единицах — как и в самом контракте, без учёта `decimals`):

```json
{
    "address": "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045",
    "tokenBalances": [
        {
            "contractAddress": "0xa0b8...eb48",
            "tokenBalance": "0x0000...03e8"
        }
    ]
}
```

Метаданные (символ, `decimals`, логотип) запрашиваются отдельным вызовом:

```js
async function getTokenMetadata(contractAddress, apiKey) {
    const response = await fetch(`https://eth-mainnet.g.alchemy.com/v2/${apiKey}`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
            jsonrpc: "2.0",
            id: 1,
            method: "alchemy_getTokenMetadata",
            params: [contractAddress],
        }),
    });

    const { result } = await response.json();
    return result; // { name, symbol, decimals, logo }
}
```

**Почему два вызова, а не один.** `getTokenBalances` может вернуть сотни токенов сразу — включать метаданные каждого в тот же ответ было бы избыточно. Метаданные запрашиваются точечно, только для токенов, которые реально нужно показать пользователю.

## История транзакций и переводов

`alchemy_getAssetTransfers` — один из самых полезных методов: отдаёт историю переводов (нативная валюта, ERC-20, ERC-721, ERC-1155) без ручного перебора логов.

```js
async function getAssetTransfers(address, apiKey) {
    const response = await fetch(`https://eth-mainnet.g.alchemy.com/v2/${apiKey}`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
            jsonrpc: "2.0",
            id: 1,
            method: "alchemy_getAssetTransfers",
            params: [{
                fromBlock: "0x0",
                toAddress: address,
                category: ["external", "erc20", "erc721", "erc1155"],
                withMetadata: true,
                maxCount: "0x14", // 20 в hex
            }],
        }),
    });

    const { result } = await response.json();
    return result.transfers;
}
```

Категории (`category`) позволяют выбрать типы переводов: `external` (обычные ETH-транзакции), `internal` (внутренние вызовы контрактов), `erc20`, `erc721`, `erc1155`, `specialnft`.

## Data API (новый унифицированный слой)

Пришедший на смену россыпи отдельных методов REST-эндпоинт для портфеля целиком:

```js
async function getPortfolio(address, apiKey) {
    const response = await fetch(
        `https://api.g.alchemy.com/data/v1/${apiKey}/assets/tokens/by-address`,
        {
            method: "POST",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify({
                addresses: [{ address, networks: ["eth-mainnet", "base-mainnet"] }],
            }),
        }
    );

    const data = await response.json();
    return data; // балансы сразу с ценами в USD, по нескольким сетям одним вызовом
}
```

Отличие от `alchemy_getTokenBalances`: Data API мультичейн по умолчанию (один вызов покрывает несколько сетей) и сразу отдаёт цены — не нужен отдельный запрос к Prices API.

## NFT

```js
async function getNftsForOwner(address, apiKey) {
    const url = new URL(`https://eth-mainnet.g.alchemy.com/nft/v3/${apiKey}/getNFTsForOwner`);
    url.searchParams.set("owner", address);
    url.searchParams.set("withMetadata", "true");

    const response = await fetch(url);
    const data = await response.json();
    return data.ownedNfts;
}
```

## Ключевые моменты для практики

1. **Единый ключ, разные базовые URL.** JSON-RPC живёт на `{network}.g.alchemy.com/v2/{key}`, Data API — на `api.g.alchemy.com/data/v1/{key}/...`, NFT API — `{network}.g.alchemy.com/nft/v3/{key}/...`. Перепутанный URL — самая частая ошибка новичков.
2. **Балансы приходят в hex и без учёта `decimals`.** `0x03e8` нужно перевести в `Number` и поделить на `10 ** decimals`, который берётся из отдельного вызова метаданных.
3. **`alchemy_*` методы — это JSON-RPC**, а значит всегда `POST` с телом `{jsonrpc, id, method, params}`, а не query-параметры.
4. **Пагинация через `pageKey`.** Большие ответы (тысячи токенов, длинная история) режутся на страницы; следующий `pageKey` приходит в предыдущем ответе.

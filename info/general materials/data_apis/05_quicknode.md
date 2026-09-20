# QuickNode: Token and NFT API через marketplace-аддоны

## Отличие от остальных провайдеров

QuickNode начинался и остаётся в первую очередь провайдером **RPC-нод**. Data API у него — не отдельный сервис, а **аддон** (add-on), который подключается к уже существующему RPC-эндпоинту через Marketplace. Практический смысл: тот же URL и тот же ключ, которым вы стучитесь в ноду за `eth_call`, начинает понимать ещё и дополнительные `qn_*` методы.

Это удобно, если в проекте уже используется QuickNode как RPC-провайдер — не нужно заводить второй сервис и второй ключ ради баланса токенов.

## Подключение аддона

1. Зарегистрироваться на `quicknode.com`, создать эндпоинт.
2. В панели эндпоинта открыть вкладку **Add-ons** → **Marketplace**.
3. Найти **Token and NFT API v2 bundle** и установить.

После установки эндпоинт продолжает работать как обычная RPC-нода, но дополнительно принимает методы `qn_*`.

> **Важно про версии.** Есть более старые аддоны **Token API** и **NFT Fetch Tool** (v1) — они считаются устаревшими. Актуальный вариант — **Token and NFT API v2 bundle**: точнее данные, больше поддерживаемых сетей, быстрее индексация. Если в старом материале встретится `qn_getWalletTokenBalance` без пометки v2 — стоит свериться с текущей документацией, сигнатура могла измениться.

## Баланс токенов кошелька

```js
async function getWalletTokenBalance(wallet, endpointUrl) {
    const response = await fetch(endpointUrl, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
            jsonrpc: "2.0",
            id: 1,
            method: "qn_getWalletTokenBalance",
            params: [{ wallet }],
        }),
    });

    const { result } = await response.json();
    return result; // { address, result: [{ address, name, symbol, totalBalance, ... }] }
}
```

Можно сузить список конкретными контрактами:

```js
params: [{
    wallet: "0xd8dA...96045",
    contracts: ["0x95aD...4cE"],
}]
```

## NFT кошелька

```js
async function fetchNfts(wallet, endpointUrl) {
    const response = await fetch(endpointUrl, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
            jsonrpc: "2.0",
            id: 1,
            method: "qn_fetchNFTs",
            params: [{ wallet, perPage: 20, page: 1 }],
        }),
    });

    const { result } = await response.json();
    return result.assets;
}
```

## Метаданные коллекции NFT

```js
async function fetchCollectionDetails(contracts, endpointUrl) {
    const response = await fetch(endpointUrl, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
            jsonrpc: "2.0",
            id: 1,
            method: "qn_fetchNFTCollectionDetails",
            params: [{ contracts }],
        }),
    });

    const { result } = await response.json();
    return result;
}
```

Обратите внимание на требование из документации: контракт должен быть верифицирован на Etherscan, иначе коллекция будет проиндексирована не полностью.

## История переводов токена по кошельку

```js
async function getWalletTokenTransactions(address, contract, endpointUrl) {
    const response = await fetch(endpointUrl, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
            jsonrpc: "2.0",
            id: 1,
            method: "qn_getWalletTokenTransactions",
            params: [{ address, contract, page: 1, perPage: 10 }],
        }),
    });

    const { result } = await response.json();
    return result;
}
```

Поддерживает `fromBlock`/`toBlock` для выборки за исторический диапазон, пагинацию до 1000 результатов.

## Использование через официальный SDK

```js
import { Core } from "@quicknode/sdk";

const core = new Core({
    endpointUrl: "https://ваш-эндпоинт.quiknode.pro/токен/",
    config: {
        addOns: { nftTokenV2: true }, // сообщает SDK, что аддон установлен
    },
});

const nfts = await core.client.qn_fetchNFTs({
    wallet: "0xd8dA...96045",
    perPage: 10,
});
```

SDK даёт автодополнение параметров и типизацию ответа — полезно, если весь остальной стек и так работает через `@quicknode/sdk`.

## Ключевые моменты для практики

1. **Один эндпоинт — и нода, и Data API.** Тот же URL, которым вызывается `eth_getBalance`, принимает и `qn_getWalletTokenBalance`. Не нужен отдельный базовый URL, как у Alchemy (Data API на другом хосте) или GoldRush (`api.covalenthq.com` отдельно от ноды).
2. **Аддон нужно явно подключить** в панели — без установки `Token and NFT API v2 bundle` методы `qn_*` вернут ошибку метода не найден, даже если эндпоинт исправно работает как RPC-нода.
3. **Верификация контракта на Etherscan** — условие для полной индексации NFT-коллекции; для собственного тестового контракта на чемпионате об этом важно не забыть, если он не верифицирован.
4. **Выбор провайдера часто определяется тем, что уже используется для RPC.** Если проект уже вызывает `eth_call` через QuickNode, логично добавить этот же аддон, а не заводить параллельно второй сервис.

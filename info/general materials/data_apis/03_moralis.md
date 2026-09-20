# Moralis: Wallet API одним вызовом

## Философия сервиса

Если Alchemy вырос из инфраструктуры нод, то Moralis изначально строился как **готовый бэкенд для Web3-приложений**: не набор примитивов, а высокоуровневые эндпоинты, которые сразу отдают декодированные, обогащённые данные.

Главное отличие от конкурентов — **Wallet API** старается закрыть весь портрет кошелька одним типом запроса: балансы, NFT, DeFi-позиции, прибыль/убыток (PnL) и полную историю с человекочитаемыми категориями операций.

## Получение ключа

Регистрация на `moralis.com` → Web3 APIs → создание API-ключа. Ключ передаётся заголовком `X-API-Key` в каждом запросе.

## Балансы ERC-20

```js
const BASE_URL = "https://deep-index.moralis.io/api/v2.2";

async function getTokenBalances(address, apiKey) {
    const response = await fetch(
        `${BASE_URL}/wallets/${address}/tokens?chain=eth`,
        {
            headers: { "X-API-Key": apiKey },
        }
    );

    const data = await response.json();
    return data.result;
}
```

Ответ сразу содержит `symbol`, `decimals`, актуальную цену в USD и процент изменения за 24 часа — то, ради чего в Alchemy пришлось бы делать три отдельных вызова (баланс, метаданные, цена).

```json
{
    "result": [
        {
            "token_address": "0xa0b8...eb48",
            "symbol": "USDC",
            "decimals": 6,
            "balance": "1500000000",
            "usd_price": 1.0001,
            "usd_value": 1500.15
        }
    ]
}
```

## NFT кошелька

```js
async function getNfts(address, apiKey) {
    const response = await fetch(
        `${BASE_URL}/${address}/nft?chain=eth&format=decimal&media_items=false`,
        { headers: { "X-API-Key": apiKey } }
    );

    const data = await response.json();
    return data.result;
}
```

`media_items: false` отключает генерацию превью изображений — экономит кредиты, если превью не нужны в интерфейсе.

## Полная история кошелька — Wallet History

Ключевая особенность Moralis: один эндпоинт отдаёт **уже классифицированную** историю с метками (`Send`, `Receive`, `Swap`, `Mint`, `NFT Purchase` и т.д.), а не сырые переводы, которые пришлось бы классифицировать самостоятельно.

```js
async function getWalletHistory(address, apiKey) {
    const response = await fetch(
        `${BASE_URL}/wallets/${address}/history?chain=eth&order=DESC`,
        { headers: { "X-API-Key": apiKey } }
    );

    const data = await response.json();
    return data.result;
}
```

Каждая запись содержит не только сумму и адреса, но и `category` (тип операции) и `summary` — готовое человекочитаемое описание («Sent 0.5 ETH to 0x1234...»). Для интерфейса «лента активности» это экономит весь слой декодирования на фронтенде.

## Чистая стоимость портфеля (Net Worth)

```js
async function getNetWorth(address, apiKey) {
    const response = await fetch(
        `${BASE_URL}/wallets/${address}/net-worth?chains=eth,polygon,base`,
        { headers: { "X-API-Key": apiKey } }
    );

    const data = await response.json();
    return data; // { total_networth_usd, chains: [{ chain, networth_usd }] }
}
```

Считает суммарную стоимость по нескольким сетям одновременно — то, что при работе через низкоуровневые методы пришлось бы агрегировать вручную: получить балансы по каждой сети, умножить на цену, сложить.

## Пример: карточка кошелька для React

```jsx
import { useState, useEffect } from "react";

function WalletCard({ address }) {
    const [data, setData] = useState(null);
    const [isLoading, setIsLoading] = useState(true);

    useEffect(() => {
        let cancelled = false;

        async function load() {
            const [tokensRes, netWorthRes] = await Promise.all([
                fetch(`https://deep-index.moralis.io/api/v2.2/wallets/${address}/tokens?chain=eth`, {
                    headers: { "X-API-Key": import.meta.env.VITE_MORALIS_KEY },
                }),
                fetch(`https://deep-index.moralis.io/api/v2.2/wallets/${address}/net-worth`, {
                    headers: { "X-API-Key": import.meta.env.VITE_MORALIS_KEY },
                }),
            ]);

            const tokens = await tokensRes.json();
            const netWorth = await netWorthRes.json();

            if (!cancelled) {
                setData({ tokens: tokens.result, netWorth: netWorth.total_networth_usd });
                setIsLoading(false);
            }
        }

        load();
        return () => { cancelled = true; };
    }, [address]);

    if (isLoading) return <p>Загрузка портфеля...</p>;

    return (
        <div>
            <h2>${data.netWorth}</h2>
            <ul>
                {data.tokens.map((token) => (
                    <li key={token.token_address}>
                        {token.symbol}: {token.balance / 10 ** token.decimals}
                    </li>
                ))}
            </ul>
        </div>
    );
}
```

> **Важно.** Ключ API никогда не должен уходить в клиентский бандл в реальном проекте — в примере он читается из переменной окружения только ради демонстрации структуры запроса. В продакшене такие вызовы идут через собственный бэкенд-прокси, который прячет ключ.

## Miграция с Sim API (Dune)

Сервис Sim (`api.sim.dune.com`) закрывается 1 августа 2026 года. Если в старых примерах или библиотеках встречается этот домен — его нужно заменить. Основные соответствия:

| Sim API | Moralis |
|---|---|
| `GET /evm/balances/{address}` | `GET /wallets/{address}/tokens` |
| `GET /evm/activity/{address}` | `GET /wallets/{address}/history` |
| `GET /evm/collectibles/{address}` | `GET /{address}/nft` |

## Ключевые моменты для практики

1. **Заголовок, а не query-параметр.** Ключ передаётся через `X-API-Key`, а не в URL — не перепутайте с QuickNode/Alchemy, где ключ обычно часть пути.
2. **Данные приходят уже обогащённые.** Это плюс для скорости разработки и минус для контроля: если нужна именно сырая величина без округлений или конвертации цены, проверяйте, не является ли поле уже посчитанным (`usd_value` — это `balance × usd_price`, посчитанные на сервере Moralis, а не в вашем коде).
3. **`chain` — обязательный параметр** почти во всех эндпоинтах, принимает как имя (`eth`, `polygon`, `base`), так и hex chainId (`0x1`).
4. **Пагинация через `cursor`**, а не через номер страницы: следующий `cursor` приходит в теле ответа.

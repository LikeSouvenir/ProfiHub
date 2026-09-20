# Практика: дашборд кошелька на двух провайдерах

## Задача

Собрать React-компонент, который показывает портфель произвольного адреса (баланс токенов, NFT, последние транзакции), используя **один провайдер по выбору**, а затем сравнить с реализацией через другого провайдера. Цель — не выучить конкретный SDK наизусть, а научиться быстро читать документацию нового Data API и оценивать, какой провайдер подходит под задачу.

## Часть 1. Дашборд на Moralis

### Требования

1. Поле ввода адреса кошелька (или ENS-имени) и выбор сети из выпадающего списка (`eth`, `polygon`, `base`).
2. Три блока: общая стоимость портфеля (Net Worth), список токенов с балансами и ценами, лента последних 10 операций с человекочитаемыми метками (`Send`, `Swap`, `NFT Purchase` и т.п.).
3. Индикатор загрузки и обработка ошибки (неверный адрес, сеть недоступна, превышен лимит запросов).
4. Запросы идут параллельно (`Promise.all`), а не последовательно.

### Структура компонента

```jsx
import { useState } from "react";

const API_KEY = import.meta.env.VITE_MORALIS_KEY;
const BASE_URL = "https://deep-index.moralis.io/api/v2.2";

async function fetchWalletData(address, chain) {
    const headers = { "X-API-Key": API_KEY };

    const [tokensRes, netWorthRes, historyRes] = await Promise.all([
        fetch(`${BASE_URL}/wallets/${address}/tokens?chain=${chain}`, { headers }),
        fetch(`${BASE_URL}/wallets/${address}/net-worth?chains=${chain}`, { headers }),
        fetch(`${BASE_URL}/wallets/${address}/history?chain=${chain}&order=DESC&limit=10`, { headers }),
    ]);

    // Одна из трёх могла вернуть ошибку — проверяем каждую отдельно,
    // чтобы не терять остальные данные из-за одного сбоя
    if (!tokensRes.ok || !netWorthRes.ok || !historyRes.ok) {
        throw new Error("Не удалось загрузить данные кошелька");
    }

    const [tokens, netWorth, history] = await Promise.all([
        tokensRes.json(),
        netWorthRes.json(),
        historyRes.json(),
    ]);

    return {
        tokens: tokens.result,
        netWorthUsd: netWorth.total_networth_usd,
        history: history.result,
    };
}

function WalletDashboard() {
    const [address, setAddress] = useState("");
    const [chain, setChain] = useState("eth");
    const [data, setData] = useState(null);
    const [isLoading, setIsLoading] = useState(false);
    const [error, setError] = useState(null);

    const handleSearch = async (event) => {
        event.preventDefault();
        setIsLoading(true);
        setError(null);

        try {
            const result = await fetchWalletData(address, chain);
            setData(result);
        } catch (err) {
            setError(err.message);
            setData(null);
        } finally {
            setIsLoading(false);
        }
    };

    return (
        <div>
            <form onSubmit={handleSearch}>
                <input
                    value={address}
                    onChange={(e) => setAddress(e.target.value)}
                    placeholder="0x... или vitalik.eth"
                />
                <select value={chain} onChange={(e) => setChain(e.target.value)}>
                    <option value="eth">Ethereum</option>
                    <option value="polygon">Polygon</option>
                    <option value="base">Base</option>
                </select>
                <button type="submit">Показать</button>
            </form>

            {isLoading && <p>Загрузка...</p>}
            {error && <p>Ошибка: {error}</p>}

            {data && (
                <>
                    <h2>${Number(data.netWorthUsd).toLocaleString()}</h2>

                    <h3>Токены</h3>
                    <ul>
                        {data.tokens.map((token) => (
                            <li key={token.token_address}>
                                {token.symbol}: {(token.balance / 10 ** token.decimals).toFixed(4)}
                                {" "}(${token.usd_value?.toFixed(2) ?? "—"})
                            </li>
                        ))}
                    </ul>

                    <h3>Последние операции</h3>
                    <ul>
                        {data.history.map((tx) => (
                            <li key={tx.hash}>
                                {tx.category}: {tx.summary}
                            </li>
                        ))}
                    </ul>
                </>
            )}
        </div>
    );
}

export default WalletDashboard;
```

---

## Часть 2. Тот же дашборд на Alchemy — сравнение подхода

Реализуйте тот же интерфейс через Alchemy и обратите внимание на структурные отличия.

```js
const ALCHEMY_KEY = import.meta.env.VITE_ALCHEMY_KEY;

async function fetchWalletDataAlchemy(address, network = "eth-mainnet") {
    const rpcUrl = `https://${network}.g.alchemy.com/v2/${ALCHEMY_KEY}`;

    // Alchemy не отдаёт net worth одним вызовом — баланс и переводы
    // запрашиваются отдельно через JSON-RPC
    const [balancesRes, transfersRes] = await Promise.all([
        fetch(rpcUrl, {
            method: "POST",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify({
                jsonrpc: "2.0", id: 1,
                method: "alchemy_getTokenBalances",
                params: [address, "erc20"],
            }),
        }),
        fetch(rpcUrl, {
            method: "POST",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify({
                jsonrpc: "2.0", id: 1,
                method: "alchemy_getAssetTransfers",
                params: [{
                    toAddress: address,
                    category: ["erc20", "external"],
                    maxCount: "0xa",
                    withMetadata: true,
                }],
            }),
        }),
    ]);

    const balancesData = await balancesRes.json();
    const transfersData = await transfersRes.json();

    // В отличие от Moralis, здесь нет ни цены, ни символа —
    // балансы приходят "сырыми", метаданные нужно догружать отдельно
    const balances = balancesData.result.tokenBalances;
    const transfers = transfersData.result.transfers;

    return { balances, transfers };
}
```

### Вопросы для сравнения (зафиксируйте ответы в README)

1. Сколько сетевых запросов потребовалось для одного и того же экрана на каждом провайдере?
2. Какому провайдеру потребовались дополнительные вызовы за метаданными (символ, `decimals`, цена)?
3. Какой ответ проще было бы напрямую отрисовать в интерфейсе без промежуточной обработки?
4. Что произойдёт с кодом, если добавить вторую сеть в мультичейн-версию дашборда — сколько строк придётся изменить у каждого провайдера?

---

## Часть 3. Опционально — живая лента через Bitquery

Добавьте на дашборд блок «недавние сделки на DEX по всей сети» (не привязан к конкретному адресу) через Bitquery-подписку:

```jsx
import { useState, useEffect } from "react";
import { createClient } from "graphql-ws";

function LiveDexFeed() {
    const [trades, setTrades] = useState([]);

    useEffect(() => {
        const client = createClient({
            url: "wss://streaming.bitquery.io/graphql",
            connectionParams: { Authorization: `Bearer ${import.meta.env.VITE_BITQUERY_TOKEN}` },
        });

        const unsubscribe = client.subscribe(
            {
                query: `
                    subscription {
                        EVM(network: eth) {
                            DEXTrades(
                                where: { Trade: { Dex: { ProtocolName: { is: "uniswap_v3" } } } }
                            ) {
                                Trade {
                                    Buy { Currency { Symbol } AmountInUSD }
                                    Sell { Currency { Symbol } }
                                }
                            }
                        }
                    }
                `,
            },
            {
                next: (data) => {
                    const trade = data.data.EVM.DEXTrades[0]?.Trade;
                    if (trade) {
                        // Ограничиваем ленту 20 последними сделками,
                        // чтобы список не рос бесконечно
                        setTrades((prev) => [trade, ...prev].slice(0, 20));
                    }
                },
                error: console.error,
                complete: () => {},
            }
        );

        // Закрываем подписку при размонтировании компонента —
        // иначе соединение останется висеть после ухода со страницы
        return () => unsubscribe();
    }, []);

    return (
        <ul>
            {trades.map((trade, i) => (
                <li key={i}>
                    {trade.Sell.Currency.Symbol} → {trade.Buy.Currency.Symbol}
                    {" "}(${Number(trade.Buy.AmountInUSD).toFixed(2)})
                </li>
            ))}
        </ul>
    );
}
```

---

## Критерии приёмки

- Дашборд корректно обрабатывает несуществующий адрес и сетевую ошибку, не роняя приложение.
- Балансы токенов показаны с учётом `decimals` — не «сырые» целые числа из блокчейна.
- Все параллельные запросы к одному провайдеру используют `Promise.all`, а не последовательные `await`.
- README отдела сравнения (часть 2) содержит осмысленные ответы, а не общие фразы.
- API-ключи не закоммичены в репозиторий — читаются из переменных окружения (`.env`, добавленный в `.gitignore`).
- (Опционально) WebSocket-подписка Bitquery корректно закрывается в функции очистки `useEffect`.

## Дополнительное задание

Возьмите контракт `TaskLog` из практики раздела Envio (`01_envio/06_practice.md`) и сравните два подхода к получению его истории:

1. Через собственный индексатор Envio (уже сделан).
2. Через `alchemy_getAssetTransfers` или Moralis `wallets/{address}/history`, отфильтровав по адресу контракта.

Обсудите: почему для кастомного контракта с собственной бизнес-логикой (агрегаты по авторам, подсчёт очков) готовый Data API не подошёл бы напрямую, даже если бы формально умел отдавать сырые события этого контракта.

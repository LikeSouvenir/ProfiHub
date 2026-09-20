# HyperSync: быстрый доступ к данным блокчейна

## Что это такое

**HyperSync** — специализированный слой доступа к ончейн-данным, написанный на Rust. Это альтернатива обычным JSON-RPC эндпоинтам: вместо запроса блоков по одному вы описываете, какие именно данные нужны, и получаете их пачками.

**Ключевые характеристики:**

- Сканирования, которые по RPC занимают часы или дни, выполняются за секунды.
- Поддержка 80+ EVM-совместимых сетей и Fuel; список постоянно пополняется.
- Клиентские библиотеки для Python, Rust, Node.js и Go.
- Гибкая выборка полей: вы запрашиваете ровно те поля, которые нужны, и не платите за передачу лишнего.

Заявленные цифры производительности из документации Envio:

| Задача | Обычный RPC | HyperSync |
|---|---|---|
| Сканирование Arbitrum на редкие логи | часы/дни | ~2 секунды |
| Все события `PoolCreated` Uniswap v3 в Ethereum | часы | секунды |

## Чем HyperSync отличается от RPC

| | JSON-RPC | HyperSync |
|---|---|---|
| Единица запроса | один блок / небольшой диапазон | произвольный диапазон, ответ приходит потоком |
| Фильтрация | по адресу и топикам | по логам, транзакциям, трейсам, блокам |
| Выбор полей | приходит всё | `fieldSelection` — только нужные поля |
| Назначение | универсальный интерфейс ноды | массовое чтение исторических данных |

RPC остаётся нужен для отправки транзакций и вызова `view`-функций. HyperSync его не заменяет в этой роли — он закрывает только чтение больших объёмов данных.

## Установка и токен

```bash
npm install @envio-dev/hypersync-client
```

Для работы требуется API-токен, который передаётся переменной окружения:

```bash
export ENVIO_API_TOKEN="ваш-токен"
```

## Инициализация клиента

Сеть выбирается URL-ом клиента — сам код запроса при этом не меняется.

```js
import { HypersyncClient } from "@envio-dev/hypersync-client";

const client = new HypersyncClient({
    url: "https://eth.hypersync.xyz",       // Ethereum Mainnet
    apiToken: process.env.ENVIO_API_TOKEN,
});

// Другие сети — меняется только URL:
// https://arbitrum.hypersync.xyz
// https://base.hypersync.xyz
```

## Структура запроса

Запрос — обычный объект. В нём описывается, с какого блока начинать, что фильтровать и какие поля возвращать.

```js
const query = {
    fromBlock: 0,          // 0 — с самого генезис-блока
    logs: [
        {
            // topics[0] — хеш сигнатуры события.
            // Внутри массива значения работают как "или"
            topics: [topic0List],
        },
    ],
    fieldSelection: {
        // Запрашиваем только нужные поля — меньше трафика, выше скорость
        log: ["Address", "Topic0", "Topic1", "Topic2", "Topic3", "Data"],
        block: ["Number", "Timestamp"],
    },
};
```

**Виды фильтров:**

- `logs` — события по адресу контракта и сигнатуре
- `transactions` — по отправителю, получателю, сигнатуре метода
- `traces` — внутренние транзакции (поддерживается не во всех сетях)
- диапазон блоков через `fromBlock` / `toBlock`

**Режимы объединения (`join modes`)** определяют, что вернётся вместе с совпадением: `JoinNothing` — только точные совпадения, `JoinTransactions` — плюс транзакции, `JoinAll` — плюс все связанные объекты.

## Получение данных потоком

Ответ приходит частями: клиент возвращает поток, из которого данные читаются в цикле. Поле `nextBlock` в ответе показывает, с какого блока продолжать.

```js
const stream = await client.stream(query, {});

let totalEvents = 0;

while (true) {
    const res = await stream.recv();

    // null означает, что данные закончились
    if (res === null) {
        break;
    }

    if (res.data && res.data.logs) {
        totalEvents += res.data.logs.length;
    }

    // Сдвигаем курсор на следующую порцию
    if (res.nextBlock) {
        query.fromBlock = res.nextBlock;
    }
}

console.log("Всего событий:", totalEvents);
```

## Полный пример: события Uniswap V3

```js
import { keccak256, toHex } from "viem";
import { HypersyncClient } from "@envio-dev/hypersync-client";

// Сигнатуры событий в человекочитаемом виде
const eventSignatures = [
    "PoolCreated(address,address,uint24,int24,address)",
    "Mint(address,address,int24,int24,uint128,uint256,uint256)",
    "Swap(address,address,int256,int256,uint160,uint128,int24)",
];

// topic0 — это keccak256 от сигнатуры события.
// Именно по этому хешу блокчейн отличает одно событие от другого
const topic0List = eventSignatures.map((sig) => keccak256(toHex(sig)));

const client = new HypersyncClient({
    url: "https://eth.hypersync.xyz",
    apiToken: process.env.ENVIO_API_TOKEN,
});

const query = {
    fromBlock: 0,
    logs: [{ topics: [topic0List] }],
    fieldSelection: {
        log: ["Address", "Topic0", "Data"],
    },
};

async function main() {
    const stream = await client.stream(query, {});
    // ...обработка потока
}

main();
```

## Полезные инструменты

- **Query Builder** — `builder.hypersync.xyz`, визуальный конструктор запросов прямо в браузере, без установки.
- **LogTUI** — терминальный просмотрщик событий с готовыми пресетами для популярных протоколов:

  ```bash
  pnpx logtui aave arbitrum       # события Aave в сети Arbitrum
  pnpx logtui uniswap-v4 unichain
  ```

## Когда брать HyperSync, а когда HyperIndex

**HyperSync** — когда нужны сырые данные и собственная логика обработки: аналитика, ETL-конвейер в свою базу, исследование данных, блок-эксплорер.

**HyperIndex** — когда нужен готовый индексатор с БД и GraphQL для приложения. Он и так использует HyperSync под капотом, поэтому скорость вы получаете бесплатно.

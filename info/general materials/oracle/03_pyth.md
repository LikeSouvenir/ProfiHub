# Pyth Network: pull-оракул с обновлением каждые 400 мс

## Идея pull-модели

Pyth принципиально устроен иначе, чем классические push-фиды Chainlink. Вместо того чтобы сеть сама постоянно писала обновления в каждый контракт каждой сети (что дорого и избыточно, если данные никому не нужны прямо сейчас), Pyth **публикует подписанные обновления офчейн**, а на цепь они попадают только тогда, когда кто-то реально запрашивает данные.

**Источник данных** — первого порядка: крупнейшие биржи, маркет-мейкеры и финансовые организации публикуют свои собственные котировки напрямую, без посредника-агрегатора между источником и оракулом.

### Почему это выгодно

| | Push (классический) | Pull (Pyth) |
|---|---|---|
| Кто платит за обновление | Сеть оракула — заранее, независимо от использования | Пользователь — в момент, когда данные реально нужны |
| Частота обновления в источнике | Ограничена расписанием/порогом | Каждые ~400 мс — самая высокая частота среди крупных оракулов |
| Актуальность в контракте | Может отставать до следующего push | Всегда настолько свежая, насколько запрошено |

## Hermes — офчейн-сервис доставки

**Hermes** — промежуточный сервис Pyth, который слушает публикуемые обновления (через Pythnet и Wormhole) и отдаёт их по запросу в виде подписанных байтов, готовых к отправке в блокчейн.

### Получение цены с фронтенда (TypeScript)

```bash
npm install @pythnetwork/hermes-client
```

```ts
import { HermesClient } from "@pythnetwork/hermes-client";

const connection = new HermesClient("https://hermes.pyth.network");

const priceIds = [
    "0xe62df6c8b4a85fe1a67db44dc12de5db330f7ac66b72dc658afedf0f4a415b43", // BTC/USD
    "0xff61491a931112ddf1bd8147cd1b641375f79f5825126d665480874634fd0ace", // ETH/USD
];

async function getLatestPrices() {
    // Возвращает и человекочитаемое значение, и байты для отправки в контракт
    const priceUpdates = await connection.getLatestPriceUpdates(priceIds);
    return priceUpdates;
}
```

Полный список ID фидов — в официальном справочнике Pyth (`docs.pyth.network/price-feeds/price-feeds`); ID — это `bytes32`, единый для всех сетей.

### Установка в контракт (Solidity)

```bash
npm install @pythnetwork/pyth-sdk-solidity
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@pythnetwork/pyth-sdk-solidity/IPyth.sol";
import "@pythnetwork/pyth-sdk-solidity/PythStructs.sol";

contract PythPriceConsumer {
    IPyth pyth;
    bytes32 constant ETH_USD_PRICE_ID =
        0xff61491a931112ddf1bd8147cd1b641375f79f5825126d665480874634fd0ace;

    // Адрес контракта Pyth различается для каждой сети — берётся из документации
    constructor(address pythContractAddress) {
        pyth = IPyth(pythContractAddress);
    }

    /**
     * priceUpdateData приходит с фронтенда — это байты, полученные от Hermes.
     * Функция payable: вызывающий оплачивает небольшую комиссию за обновление.
     */
    function getFreshPrice(
        bytes[] calldata priceUpdateData
    ) public payable returns (PythStructs.Price memory) {
        // 1. Сколько стоит обновление именно этих данных
        uint256 updateFee = pyth.getUpdateFee(priceUpdateData);

        // 2. Публикуем обновление в контракт Pyth, оплачивая комиссию
        pyth.updatePriceFeeds{value: updateFee}(priceUpdateData);

        // 3. Читаем цену не старше 60 секунд — иначе вызов откатится
        return pyth.getPriceNoOlderThan(ETH_USD_PRICE_ID, 60);
    }
}
```

**Три шага — это и есть суть pull-модели**: получить данные с Hermes → оплатить и опубликовать в контракт → прочитать. В push-модели (Chainlink Data Feeds) шагов 1 и 2 просто нет — данные уже лежат в контракте к моменту чтения.

### Получение полного вызова с фронтенда до контракта

```js
import { HermesClient } from "@pythnetwork/hermes-client";
import { ethers } from "ethers";

async function updateAndReadPrice(contract) {
    const connection = new HermesClient("https://hermes.pyth.network");

    const priceUpdates = await connection.getLatestPriceUpdates([
        "0xff61491a931112ddf1bd8147cd1b641375f79f5825126d665480874634fd0ace",
    ]);

    // binary.data — массив байтовых строк, которые контракт ожидает как bytes[]
    const updateData = priceUpdates.binary.data.map((d) => "0x" + d);

    // getUpdateFee — view-функция, читается перед отправкой транзакции
    const fee = await contract.getUpdateFee(updateData);

    const tx = await contract.getFreshPrice(updateData, { value: fee });
    const receipt = await tx.wait();

    return receipt;
}
```

### Важная защита: `getPriceNoOlderThan`

Как и у Chainlink, у Pyth есть риск использовать устаревшую цену. Функция `getPriceNoOlderThan(id, maxAge)` откатывает транзакцию сама, если данные старше указанного порога — не нужно вручную сверять `block.timestamp`, как в push-модели.

### Confidence Interval — доверительный интервал

Особенность структуры `PythStructs.Price` — помимо самой цены и экспоненты, в ней есть поле `conf` (confidence interval): диапазон погрешности вокруг цены, отражающий разброс мнений источников в момент публикации. Для критичных расчётов (например, порог ликвидации) стоит учитывать не только `price`, но и ширину `conf` — резкий рост доверительного интервала сигнализирует о нестабильном/малоликвидном рынке.

## Ключевые моменты для практики

1. **Три обязательных шага pull-модели:** получить updateData с Hermes → `getUpdateFee` → `updatePriceFeeds` с оплатой → прочитать цену. Пропуск любого шага либо не скомпилируется, либо откатится.
2. **`payable` и комиссия — не опция.** В отличие от Chainlink `view`-чтения, вызов Pyth почти всегда требует небольшой суммы ETH на оплату обновления.
3. **`getPriceNoOlderThan` вместо ручной проверки timestamp** — встроенная защита от устаревших данных, специфичная для Pyth.
4. **ID фида одинаков во всех сетях** (в отличие от Chainlink, где у каждой сети свой адрес агрегатора) — меняется только адрес самого контракта Pyth.

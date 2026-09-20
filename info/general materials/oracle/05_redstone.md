# RedStone: push и pull в одном сервисе

## Что делает RedStone особенным

RedStone — единственный оракул из раздела, который **сам предлагает выбор модели доставки** под задачу конкретного проекта, а не жёстко привязан к одной архитектуре:

- **Push Model** — классический подход, совместимый по интерфейсу с Chainlink `AggregatorV3Interface`. Подходит, если проект уже написан под Chainlink и нужен готовый по интерфейсу заменитель (например, для пары, которой у Chainlink ещё нет).
- **Pull Model** — данные не хранятся on-chain вообще, а «впрыскиваются» прямо в `calldata` транзакции фронтендом в момент вызова.

Отдельно RedStone быстро добавляет фиды под **новые классы активов** — LST (liquid staking tokens), LRT (liquid restaking tokens), нестандартные стейблкоины — то есть закрывает нишу «нужен нетиповой актив, которого ещё нет у крупных провайдеров».

## Push Model — совместимость с Chainlink «из коробки»

Если проект уже читает `AggregatorV3Interface`, миграция на RedStone Push не требует переписывать логику контракта — только сменить адрес:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import {AggregatorV3Interface} from "@chainlink/contracts/src/v0.8/shared/interfaces/AggregatorV3Interface.sol";

contract RedStoneDataConsumer {
    AggregatorV3Interface internal dataFeedETH;

    constructor() {
        // Адрес контракта RedStone Push для конкретной пары и сети —
        // из документации RedStone, интерфейс идентичен Chainlink
        dataFeedETH = AggregatorV3Interface(0xАдресRedStonePush);
    }

    function getLatestPrice() public view returns (int256 answer, uint256 updatedAt) {
        (, answer, , updatedAt, ) = dataFeedETH.latestRoundData();
        require(block.timestamp - updatedAt < 1 hours, "Stale price");
    }
}
```

Это работает именно потому, что RedStone Push сознательно реализует тот же самый интерфейс — переключение между Chainlink и RedStone Push сводится к смене одной константы адреса.

## Pull Model — принципиально другая механика

Здесь важен сдвиг мышления, о который часто спотыкаются: **данные не лежат в storage контракта вообще**. Они прилетают внутри `calldata` конкретной транзакции — контракт не может «просто прочитать» цену в любой момент, только в момент выполнения транзакции, к которой фронтенд приложил подписанные данные.

### Контракт-потребитель

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@redstone-finance/evm-connector/contracts/data-services/PrimaryProdDataServiceConsumerBase.sol";

contract ExampleContract is PrimaryProdDataServiceConsumerBase {
    function getPrices() public view returns (uint256[] memory) {
        bytes32[] memory dataFeedIds = new bytes32[](2);
        dataFeedIds[0] = bytes32("BTC");
        dataFeedIds[1] = bytes32("ETH");

        // Значения извлекаются прямо из данных транзакции (calldata),
        // а не из storage — если payload не приложен, вызов откатится
        return getOracleNumericValuesFromTxMsg(dataFeedIds);
    }
}
```

**Формат числа:** RedStone по умолчанию использует 8 знаков после запятой — то же соглашение, что у Chainlink USD-фидов (ETH по $3000 вернётся как `300000000000`, то есть `3000 * 10^8`).

### Обязательный шаг на фронтенде — «обёртка» транзакции

Контракта самого по себе недостаточно: без специальной обёртки вызов откатится с ошибкой `CalldataMustHaveValidPayload`, потому что данные физически отсутствуют в транзакции.

```bash
npm install @redstone-finance/evm-connector
```

```js
import { WrapperBuilder } from "@redstone-finance/evm-connector";
import { ethers } from "ethers";

async function readPricesFromContract(contractAddress, abi, signer) {
    const baseContract = new ethers.Contract(contractAddress, abi, signer);

    // WrapperBuilder оборачивает контракт: каждый вызов автоматически
    // дополняется подписанными данными от указанного data service
    const wrappedContract = WrapperBuilder.wrap(baseContract).usingDataService({
        dataServiceId: "redstone-primary-prod",
        dataPackagesIds: ["BTC", "ETH"],
    });

    // Вызов выглядит как обычный, но фактически несёт payload в calldata
    const prices = await wrappedContract.getPrices();
    return prices;
}
```

**Важное практическое ограничение:** Remix IDE не поддерживает модификацию транзакций так, как это делает `evm-connector` — тестировать pull-контракты RedStone через Remix напрямую не получится, нужен Hardhat/Foundry скрипт или фронтенд с подключённым `evm-connector`.

## Сравнение push и pull внутри самого RedStone

| | Push | Pull |
|---|---|---|
| Совместимость | `AggregatorV3Interface`, как у Chainlink | Собственный `RedstoneConsumerBase` |
| Где хранится значение | В storage, постоянно | Нигде — только в момент вызова |
| Нужна ли обёртка на фронтенде | Нет | Да, обязательно (`WrapperBuilder`) |
| Когда выбирать | Проект уже архитектурно ждёт Chainlink-совместимый интерфейс | Нужна максимальная свежесть данных и меньше storage-операций |

## Ключевые моменты для практики

1. **Pull-модель — это про calldata, не про storage.** Если попытаться прочитать значение вне контекста «обёрнутой» транзакции — в контракте физически нет данных для чтения.
2. **`WrapperBuilder.wrap(...).usingDataService(...)` — обязательный шаг фронтенда**, без которого pull-контракты RedStone не работают вообще, а не просто работают хуже.
3. **Push-модель — самый быстрый путь миграции с Chainlink**, если нужен фид для актива, которого у Chainlink ещё нет: интерфейс идентичен, меняется только адрес.
4. **8 знаков после запятой по умолчанию** — совпадает с конвенцией Chainlink USD-фидов, но стоит явно проверять в документации конкретного фида, если работаете не с USD-парой.
5. **Не тестируйте pull-контракты в Remix** — используйте Hardhat/Foundry скрипт с `evm-connector` или полноценный фронтенд.

# Практика: ценовой порог на push и pull моделях

## Задача

Реализовать один и тот же сценарий — «наградной фонд, который разблокируется только если ETH стоит дороже заданного порога» — двумя разными способами: через push-оракул (Chainlink Data Feeds) и через pull-оракул (Pyth). Цель — на практике прочувствовать разницу в архитектуре, а не просто прочитать про неё.

## Сценарий

Контракт `ThresholdVault` хранит средства чемпионата и позволяет забрать приз только тогда, когда ETH/USD выше определённой отметки — простая, но показательная модель, где от выбора оракула напрямую зависит структура кода и стоимость вызова.

---

## Часть 1. Push-версия (Chainlink Data Feeds)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {AggregatorV3Interface} from "@chainlink/contracts/src/v0.8/shared/interfaces/AggregatorV3Interface.sol";

contract ThresholdVaultPush {
    AggregatorV3Interface internal priceFeed;
    uint256 public immutable thresholdUsd; // порог, с 8 знаками, как у фида
    uint256 public constant MAX_PRICE_AGE = 1 hours;

    mapping(address => uint256) public deposits;

    constructor(address _priceFeedAddress, uint256 _thresholdUsd) {
        priceFeed = AggregatorV3Interface(_priceFeedAddress);
        thresholdUsd = _thresholdUsd;
    }

    function deposit() external payable {
        deposits[msg.sender] += msg.value;
    }

    /**
     * Push-модель: чтение курса не требует данных извне —
     * значение уже лежит в контракте оракула, вызов view и бесплатен
     */
    function withdraw() external {
        uint256 amount = deposits[msg.sender];
        require(amount > 0, "Nothing to withdraw");

        (, int256 answer, , uint256 updatedAt, ) = priceFeed.latestRoundData();
        require(block.timestamp - updatedAt <= MAX_PRICE_AGE, "Price too stale");
        require(uint256(answer) >= thresholdUsd, "ETH price below threshold");

        deposits[msg.sender] = 0;
        payable(msg.sender).transfer(amount);
    }
}
```

**Обратите внимание:** `withdraw()` не принимает никаких дополнительных данных снаружи — вся нужная информация уже доступна на цепи в момент вызова.

---

## Часть 2. Pull-версия (Pyth)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@pythnetwork/pyth-sdk-solidity/IPyth.sol";
import "@pythnetwork/pyth-sdk-solidity/PythStructs.sol";

contract ThresholdVaultPull {
    IPyth pyth;
    bytes32 constant ETH_USD_PRICE_ID =
        0xff61491a931112ddf1bd8147cd1b641375f79f5825126d665480874634fd0ace;
    int64 public immutable thresholdUsd;

    mapping(address => uint256) public deposits;

    constructor(address _pythAddress, int64 _thresholdUsd) {
        pyth = IPyth(_pythAddress);
        thresholdUsd = _thresholdUsd;
    }

    function deposit() external payable {
        deposits[msg.sender] += msg.value;
    }

    /**
     * Pull-модель: вызывающий обязан принести свежие данные и оплатить
     * их публикацию — функция payable, принимает priceUpdateData
     */
    function withdraw(bytes[] calldata priceUpdateData) external payable {
        uint256 amount = deposits[msg.sender];
        require(amount > 0, "Nothing to withdraw");

        uint256 updateFee = pyth.getUpdateFee(priceUpdateData);
        pyth.updatePriceFeeds{value: updateFee}(priceUpdateData);

        PythStructs.Price memory price = pyth.getPriceNoOlderThan(ETH_USD_PRICE_ID, 60);
        require(price.price >= thresholdUsd, "ETH price below threshold");

        deposits[msg.sender] = 0;
        payable(msg.sender).transfer(amount);
    }
}
```

**Обратите внимание:** `withdraw()` теперь принимает `priceUpdateData` и стала `payable` — вызывающий обязан принести данные и оплатить их публикацию. Без фронтенда, который получит `updateData` от Hermes, вызвать эту функцию напрямую (например, из Etherscan) не получится содержательно.

---

## Часть 3. Фронтенд для pull-версии

Push-версия вызывается как любая обычная функция контракта (`contract.withdraw()`), поэтому отдельного кода не требует. Pull-версия нуждается в подготовительном шаге:

```jsx
import { useState } from "react";
import { HermesClient } from "@pythnetwork/hermes-client";
import { ethers } from "ethers";

const ETH_USD_ID = "0xff61491a931112ddf1bd8147cd1b641375f79f5825126d665480874634fd0ace";

function WithdrawButtonPull({ contract }) {
    const [isLoading, setIsLoading] = useState(false);
    const [status, setStatus] = useState(null);

    const handleWithdraw = async () => {
        setIsLoading(true);
        setStatus(null);

        try {
            // 1. Получаем свежие данные с Hermes
            const connection = new HermesClient("https://hermes.pyth.network");
            const priceUpdates = await connection.getLatestPriceUpdates([ETH_USD_ID]);
            const updateData = priceUpdates.binary.data.map((d) => "0x" + d);

            // 2. Узнаём стоимость публикации именно этих данных
            const fee = await contract.getUpdateFee(updateData);

            // 3. Отправляем транзакцию с данными и оплатой прямо внутри вызова
            const tx = await contract.withdraw(updateData, { value: fee });
            await tx.wait();

            setStatus("Успешно снято");
        } catch (err) {
            setStatus(`Ошибка: ${err.message}`);
        } finally {
            setIsLoading(false);
        }
    };

    return (
        <div>
            <button onClick={handleWithdraw} disabled={isLoading}>
                {isLoading ? "Получаем свежую цену..." : "Забрать приз"}
            </button>
            {status && <p>{status}</p>}
        </div>
    );
}
```

---

## Часть 4. Сравнение — заполните таблицу по результатам практики

| Критерий | Push (Chainlink) | Pull (Pyth) |
|---|---|---|
| Строк кода в контракте | | |
| Нужен ли доп. код на фронтенде | | |
| Газ за вызов `withdraw()` (замерить в Remix/Hardhat) | | |
| Что произойдёт, если фид не обновлялся сутки | | |
| Что произойдёт, если фронтенд не приложил свежие данные | | |

## Задания на разбор

1. Разверните обе версии в тестовой сети (Sepolia) с реальными адресами Chainlink и Pyth контрактов.
2. Установите `thresholdUsd` заведомо выше текущей цены ETH и убедитесь, что `withdraw()` откатывается в обеих версиях с понятной причиной.
3. Для push-версии: временно укажите адрес фида несуществующей пары и понаблюдайте за поведением — объясните разницу между «ошибкой чтения» и «устаревшими данными».
4. Для pull-версии: попробуйте вызвать `withdraw` с пустым `priceUpdateData` напрямую через Etherscan (без фронтенда) — объясните в README, почему это не сработает и какая именно проверка внутри `pyth.updatePriceFeeds` останавливает вызов.

## Критерии приёмки

- Обе версии контракта компилируются и разворачиваются без ошибок.
- `withdraw()` в обеих версиях корректно отклоняет попытку снять средства при цене ниже порога.
- Push-версия проверяет свежесть данных через `updatedAt`, pull-версия — через `getPriceNoOlderThan`.
- Фронтенд для pull-версии корректно обрабатывает сетевую ошибку Hermes отдельно от отката транзакции контракта.
- Таблица сравнения в README заполнена реальными измеренными значениями газа, а не предположениями.

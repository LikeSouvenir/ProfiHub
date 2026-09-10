# Модуль 3: Payable, глобальные переменные и ABI — Эталонное решение

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/// @title TimeLockedVault - Хранилище с временной блокировкой и безопасным выводом
/// @notice Демонстрирует работу с payable, block.timestamp, низкоуровневым call и abi.encode/decode
contract TimeLockedVault {
    // 1. Ошибки
    error InsufficientDeposit(uint256 sent, uint256 minRequired);
    error ExceedsMaxLockDuration(uint256 requested, uint256 maxAllowed);
    error LockPeriodActive(uint256 currentTime, uint256 unlockTime);
    error NoFundsToWithdraw();
    error EthTransferFailed();

    // 2. Хранение параметров блокировки
    struct LockBox {
        uint256 balance;
        uint256 unlockTimestamp;
    }

    mapping(address => LockBox) public vaults;

    // Константы (экономят газ, вычисляются на этапе компиляции)
    uint256 public constant MIN_DEPOSIT = 0.01 ether;
    uint256 public constant MAX_DURATION = 365 days;

    // События
    event Deposited(address indexed user, uint256 amount, uint256 unlockTimestamp);
    event Withdrawn(address indexed user, uint256 amount);

    /// @notice Депозит с указанием времени блокировки
    /// @param _lockDuration Длительность блокировки в секундах
    function deposit(uint256 _lockDuration) external payable {
        if (msg.value < MIN_DEPOSIT) {
            revert InsufficientDeposit(msg.value, MIN_DEPOSIT);
        }
        if (_lockDuration > MAX_DURATION) {
            revert ExceedsMaxLockDuration(_lockDuration, MAX_DURATION);
        }

        LockBox storage box = vaults[msg.sender];
        uint256 targetUnlockTime = block.timestamp + _lockDuration;

        // Если уже был установлен более поздний лок, не уменьшаем его
        if (targetUnlockTime > box.unlockTimestamp) {
            box.unlockTimestamp = targetUnlockTime;
        }

        box.balance += msg.value;

        emit Deposited(msg.sender, msg.value, box.unlockTimestamp);
    }

    /// @notice Прием прямого перевода ETH на адрес контракта
    receive() external payable {
        if (msg.value < MIN_DEPOSIT) {
            revert InsufficientDeposit(msg.value, MIN_DEPOSIT);
        }

        LockBox storage box = vaults[msg.sender];
        uint256 targetUnlockTime = block.timestamp + 1 days;

        if (targetUnlockTime > box.unlockTimestamp) {
            box.unlockTimestamp = targetUnlockTime;
        }

        box.balance += msg.value;

        emit Deposited(msg.sender, msg.value, box.unlockTimestamp);
    }

    /// @notice Вывод разблокированных средств
    /// @dev Реализован паттерн Checks-Effects-Interactions (CEI) для предотвращения Reentrancy
    function withdraw() external {
        LockBox storage box = vaults[msg.sender];

        // 1. CHECKS (Проверки условий)
        if (box.balance == 0) {
            revert NoFundsToWithdraw();
        }
        if (block.timestamp < box.unlockTimestamp) {
            revert LockPeriodActive(block.timestamp, box.unlockTimestamp);
        }

        uint256 amountToTransfer = box.balance;

        // 2. EFFECTS (Изменение состояния контракта ДО отправки наружу)
        box.balance = 0;

        // 3. INTERACTIONS (Внешний вызов через низкоуровневый .call)
        (bool success, ) = payable(msg.sender).call{value: amountToTransfer}("");
        if (!success) {
            revert EthTransferFailed();
        }

        emit Withdrawn(msg.sender, amountToTransfer);
    }

    /// @notice Упаковка метаданных депозита в байтовый массив через abi.encode
    function packDepositData(
        address _user,
        uint256 _amount,
        uint256 _releaseTime
    ) external pure returns (bytes memory) {
        return abi.encode(_user, _amount, _releaseTime);
    }

    /// @notice Распаковка байтового массива обратно в строго типизированные переменные
    function unpackDepositData(bytes memory _data)
        external
        pure
        returns (
            address user,
            uint256 amount,
            uint256 releaseTime
        )
    {
        (user, amount, releaseTime) = abi.decode(_data, (address, uint256, uint256));
    }
}
```

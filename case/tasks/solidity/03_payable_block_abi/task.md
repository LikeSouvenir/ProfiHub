# Модуль 3: Payable, глобальные переменные и ABI — Практическое задание

## Проект: Временной депозитарий с защитой от Reentrancy (TimeLockedVault)

### Цель
Освоить прием нативной валюты (`payable`, `receive`), извлечение контекста транзакции (`msg.sender`, `msg.value`), работу с временными метками блокчейна (`block.timestamp`), безопасную отправку эфира через низкоуровневый `.call{value: ...}("")` по паттерну Checks-Effects-Interactions, а также кодирование/декодирование параметров через `abi.encode` и `abi.decode`.

### Техническое задание
Создайте смарт-контракт `TimeLockedVault` со следующими требованиями:

1. **Кастомные ошибки:**
   - `InsufficientDeposit(uint256 sent, uint256 minRequired)`
   - `ExceedsMaxLockDuration(uint256 requested, uint256 maxAllowed)`
   - `LockPeriodActive(uint256 currentTime, uint256 unlockTime)`
   - `NoFundsToWithdraw()`
   - `EthTransferFailed()`

2. **Состояние:**
   - Структура `LockBox`: `uint256 balance` (сумма депозита) и `uint256 unlockTimestamp` (время разблокировки);
   - `mapping(address => LockBox) public vaults`;
   - Константы: `MIN_DEPOSIT = 0.01 ether`, `MAX_DURATION = 365 days`.

3. **Прием депозитов:**
   - `deposit(uint256 _lockDuration) external payable`:
     - Проверяет `msg.value >= MIN_DEPOSIT`.
     - Проверяет `_lockDuration <= MAX_DURATION`.
     - Рассчитывает новое время разблокировки: `block.timestamp + _lockDuration`. Если оно больше текущего `unlockTimestamp`, обновляет его.
     - Увеличивает `vaults[msg.sender].balance` на `msg.value`.
   - `receive() external payable`:
     - Срабатывает при прямой отправке ETH без data.
     - Проверяет минимальный депозит. Блокирует средства на 1 сутки по умолчанию (`block.timestamp + 1 days`).

4. **Вывод средств (`withdraw()`):**
   - Строгое следование паттерну **Checks-Effects-Interactions (CEI)**:
     1. **Checks:** Проверить, что баланс `> 0` и `block.timestamp >= unlockTimestamp`.
     2. **Effects:** Сохранить сумму во временную переменную и обнулить `vaults[msg.sender].balance = 0`.
     3. **Interactions:** Выполнить отправку через `(bool success, ) = payable(msg.sender).call{value: amount}("")`. Если `!success`, выбросить `EthTransferFailed()`.

5. **Работа с ABI:**
   - `packDepositData(address _user, uint256 _amount, uint256 _releaseTime) external pure returns (bytes memory)` — сериализация данных депозита через `abi.encode`.
   - `unpackDepositData(bytes memory _data) external pure returns (address user, uint256 amount, uint256 releaseTime)` — десериализация через `abi.decode`.

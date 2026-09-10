# Модуль 2: Конструктор, модификаторы и ошибки — Практическое задание

## Проект: Ролевой менеджер доступа (RoleAccessHub)

### Цель
Освоить инициализацию состояния в `constructor`, создание кастомных ошибок (`custom errors`) вместо строковых `require`, написание параметризованных `modifier`, использование `_` (merge point) и работу с `revert`.

### Техническое задание
Создайте смарт-контракт `RoleAccessHub` со следующими требованиями:

1. **Кастомные ошибки (Custom Errors):**
   - `NotOwner(address caller)` — вызывающий не является владельцем;
   - `AccountBlacklisted(address account)` — аккаунт находится в черном списке;
   - `ContractIsPaused()` — вызов во время действия паузы;
   - `InvalidAddress()` — передан нулевой адрес;
   - `InsufficientAccessLevel(uint256 current, uint256 required)` — недостаточный уровень доступа;
   - `InvalidLevelValue(uint256 level)` — недопустимое значение уровня доступа (например, выше 5).

2. **Состояние контракта:**
   - `address public owner` — владелец контракта;
   - `bool public isPaused` — флаг экстренной паузы;
   - `mapping(address => bool) public isBlacklisted` — черный список адресов;
   - `mapping(address => uint256) public userAccessLevel` — уровень доступа (1–5).

3. **Конструктор:**
   - Принимает `address _initialOwner`.
   - Если передан `address(0)`, откатывает транзакцию ошибкой `InvalidAddress()`.
   - Назначает `owner = _initialOwner`.

4. **Модификаторы:**
   - `onlyOwner`: разрешает вызов только владельцу контракта (`msg.sender == owner`), иначе `revert NotOwner(msg.sender)`.
   - `notBlacklisted(address account)`: проверяет, что указанный адрес не в бане, иначе `revert AccountBlacklisted(account)`.
   - `whenNotPaused`: проверяет, что контракт активен, иначе `revert ContractIsPaused()`.
   - `requireLevel(uint256 minLevel)`: проверяет, что уровень вызывающего `>= minLevel`.

5. **Функции управления:**
   - `togglePause() external onlyOwner` — переключает паузу (`isPaused = !isPaused`).
   - `setBlacklist(address _account, bool _status) external onlyOwner` — блокирует/разблокирует адрес.
   - `setUserLevel(address _account, uint256 _level) external onlyOwner` — выставляет уровень (1–5).
   - `criticalOperation() external whenNotPaused notBlacklisted(msg.sender) requireLevel(3) returns (bool)` — операция, требующая одновременного выполнения всех условий.

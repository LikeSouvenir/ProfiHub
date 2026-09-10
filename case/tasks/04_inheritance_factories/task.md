# Модуль 4: Наследование, полиморфизм и фабрики контрактов — Практическое задание

## Проект: Фабрика расчетных аккаунтов (AccountFactory)

### Цель
Освоить объектно-ориентированные возможности Solidity: абстрактные контракты (`abstract contract`), виртуальные функции (`virtual`), переопределение (`override`), обращение к родительским методам (`super`), проброс аргументов в базовые конструкторы и фабричный паттерн (деплой контрактов через `new`).

### Техническое задание
Создайте иерархию смарт-контрактов:

1. **Базовый контракт `BaseAccount`:**
   - Поле `address public immutable owner`.
   - Конструктор `constructor(address _owner)` инициализирует владельца.
   - Функция `calculateFee(uint256 amount) public view virtual returns (uint256)`: рассчитывает стандартную комиссию 1.0% (`amount * 100 / 10000`).
   - Функция `accountType() public pure virtual returns (string memory)`: возвращает `"BASE"`.

2. **Дочерний контракт `PremiumAccount is BaseAccount`:**
   - Конструктор передает владельца в базовый: `BaseAccount(_owner)`.
   - Переопределяет `calculateFee`: льготная комиссия 0.2% (`amount * 20 / 10000`).
   - Переопределяет `accountType`: возвращает `"PREMIUM"`.

3. **Дочерний контракт `CashbackAccount is BaseAccount`:**
   - Конструктор передает владельца в базовый.
   - Переопределяет `calculateFee`: вызывает реализацию родителя через `super.calculateFee(amount)` и вычитает кешбэк в размере 0.1% (`amount * 10 / 10000`).
   - Переопределяет `accountType`: возвращает `"CASHBACK"`.

4. **Фабрика `AccountFactory`:**
   - Ошибка `UnknownAccountType()`.
   - Хранилище: `address[] public allAccounts` и `mapping(address => address[]) public userAccounts`.
   - Событие `AccountCreated(address indexed user, address indexed accountAddress, string accountType)`.
   - Функция `createAccount(string memory _accType) external returns (address)`:
     - Сравнивает строки по хешу `keccak256(bytes(_accType))`.
     - При `"PREMIUM"` создает через `new PremiumAccount(msg.sender)`.
     - При `"CASHBACK"` создает через `new CashbackAccount(msg.sender)`.
     - Иначе выбрасывает `UnknownAccountType()`.
     - Сохраняет созданный адрес в реестры и генерирует событие.

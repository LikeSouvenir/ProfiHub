# Модуль 5: Интерфейсы и стандарт токенов ERC-20 — Практическое задание

## Проект: Собственный токен ERC-20 с механикой сжигания и контракт-локер

### Цель
Понять фундаментальную архитектуру стандарта ERC-20: описание абстрактного интерфейса `IERC20`, внутренние переменные балансов и аппрувов (`mapping(address => mapping(address => uint256))`), функции `transfer`, `approve`, `transferFrom`, генерацию событий `Transfer` и `Approval`, методы эмиссии (`_mint`) и сжигания (`_burn`), а также кросс-контрактное взаимодействие через вызовы интерфейса.

### Техническое задание
1. **Интерфейс `IERC20`:**
   - Объявить стандартные функции: `totalSupply`, `balanceOf`, `transfer`, `allowance`, `approve`, `transferFrom`.
   - Объявить события: `Transfer(address indexed from, address indexed to, uint256 value)` и `Approval(address indexed owner, address indexed spender, uint256 value)`.

2. **Контракт токена `MyCustomToken is IERC20`:**
   - Метаданные: `name = "ProfiHub Token"`, `symbol = "PHT"`, `decimals = 18`.
   - Конструктор: принимает `uint256 _initialSupply`, выпускает токены на адрес создателя (`msg.sender * 10 ** decimals`).
   - Функция `transfer`: проверяет баланс и отсутствие нулевого адреса, переводит токены, генерирует `Transfer`.
   - Функция `approve`: устанавливает лимит расходов для `spender`, генерирует `Approval`.
   - Функция `transferFrom`: проверяет и уменьшает `allowance[from][msg.sender]`, переводит средства от `from` к `to`.
   - Функция `burn(uint256 amount)`: позволяет любому держателю уменьшить свой баланс и общий `totalSupply`.

3. **Контракт `TokenLocker`:**
   - Предназначен для блокировки токенов стандарта ERC-20 на смарт-контракте.
   - `mapping(address => mapping(address => uint256)) public lockedBalances` (токен => пользователь => сумма).
   - Функция `lockTokens(address _tokenAddress, uint256 _amount) external`:
     - Принимает адрес токена и приводит его к типу `IERC20(_tokenAddress)`.
     - Забирает токены у `msg.sender` на адрес самого локера `address(this)` через `transferFrom`.
     - Увеличивает заблокированный баланс пользователя.

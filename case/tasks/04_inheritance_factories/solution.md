# Модуль 4: Наследование, полиморфизм и фабрики контрактов — Эталонное решение

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/// @notice Базовый абстрактный контракт расчетного счета
abstract contract BaseAccount {
    // Immutable переменная: значение задается в конструкторе и не может быть изменено
    address public immutable owner;

    event Initialized(address indexed owner);

    constructor(address _owner) {
        owner = _owner;
        emit Initialized(_owner);
    }

    /// @notice Расчет базовой комиссии в 1% (100 базисных пунктов из 10 000)
    function calculateFee(uint256 _amount) public view virtual returns (uint256) {
        return (_amount * 100) / 10000;
    }

    /// @notice Возвращает строковый идентификатор типа аккаунта
    function accountType() public pure virtual returns (string memory) {
        return "BASE";
    }
}

/// @notice Премиальный аккаунт с пониженной фиксированной комиссией
contract PremiumAccount is BaseAccount {
    // Проброс аргумента в конструктор базового контракта
    constructor(address _owner) BaseAccount(_owner) {}

    /// @notice Переопределение комиссии: 0.20% (20 bps)
    function calculateFee(uint256 _amount) public view override returns (uint256) {
        return (_amount * 20) / 10000;
    }

    function accountType() public pure override returns (string memory) {
        return "PREMIUM";
    }
}

/// @notice Аккаунт с кешбэком, использующий super для вызова родительской логики
contract CashbackAccount is BaseAccount {
    constructor(address _owner) BaseAccount(_owner) {}

    /// @notice Вычисляет базовую комиссию предка через super и вычитает кешбэк
    function calculateFee(uint256 _amount) public view override returns (uint256) {
        uint256 baseFee = super.calculateFee(_amount);
        uint256 cashbackDiscount = (_amount * 10) / 10000; // 0.10% скидка

        if (baseFee > cashbackDiscount) {
            return baseFee - cashbackDiscount;
        }
        return 0;
    }

    function accountType() public pure override returns (string memory) {
        return "CASHBACK";
    }
}

/// @notice Фабрика деплоя экземпляров контрактов (Factory Pattern)
contract AccountFactory {
    error UnknownAccountType();

    address[] public allAccounts;
    mapping(address => address[]) public userAccounts;

    event AccountCreated(address indexed user, address indexed accountAddress, string accountType);

    /// @notice Создает новый дочерний аккаунт выбранного типа
    /// @param _accType Тип аккаунта ("PREMIUM" или "CASHBACK")
    /// @return newAccountAddress Адрес развернутого смарт-контракта
    function createAccount(string memory _accType) external returns (address newAccountAddress) {
        bytes32 typeHash = keccak256(bytes(_accType));

        if (typeHash == keccak256(bytes("PREMIUM"))) {
            // Динамическое создание контракта оператором new
            PremiumAccount prem = new PremiumAccount(msg.sender);
            newAccountAddress = address(prem);
        } else if (typeHash == keccak256(bytes("CASHBACK"))) {
            CashbackAccount cash = new CashbackAccount(msg.sender);
            newAccountAddress = address(cash);
        } else {
            revert UnknownAccountType();
        }

        allAccounts.push(newAccountAddress);
        userAccounts[msg.sender].push(newAccountAddress);

        emit AccountCreated(msg.sender, newAccountAddress, _accType);
    }

    /// @notice Получить все аккаунты конкретного пользователя
    function getUserAccounts(address _user) external view returns (address[] memory) {
        return userAccounts[_user];
    }

    /// @notice Общее количество всех созданных через фабрику аккаунтов
    function getTotalAccountsCount() external view returns (uint256) {
        return allAccounts.length;
    }
}
```

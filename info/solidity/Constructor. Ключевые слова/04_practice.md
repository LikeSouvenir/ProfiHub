# Разбор практического задания: Банковская система

В этом задании мы объединим использование конструктора, структур, маппингов, модификаторов и обработчиков ошибок (require, кастомные ошибки) для создания простейшей банковской системы.

## Условие задачи
Создать смарт-контракт банковской системы:
- Пользователи могут зарегистрироваться.
- Просмотр информации профиля.
- Пополнение вклада (положить деньги) и снятие (забрать деньги).
- Структуры: `Profile` (Имя, Идентификатор, Логин, Пароль, баланс), `Deposit` (Адрес, сумма).

## Решение с подробными комментариями

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract BankSystem {
    // ==========================================
    // СТРУКТУРЫ ДАННЫХ
    // ==========================================
    
    struct Profile {
        string name;
        uint256 id;
        string login;
        bytes32 passwordHash; // Хранить пароли в открытом виде нельзя, используем хэш
        uint256 balance;
    }
    
    struct Deposit {
        address userAddress;
        uint256 amount;
    }

    // ==========================================
    // ПЕРЕМЕННЫЕ СОСТОЯНИЯ И МАППИНГИ
    // ==========================================
    
    address public bankOwner;
    uint256 public nextUserId;
    
    // Связываем адрес пользователя с его профилем
    mapping(address => Profile) private profiles;
    
    // Связываем адрес пользователя с информацией о его вкладе
    mapping(address => Deposit) public deposits;
    
    // ==========================================
    // КАСТОМНЫЕ ОШИБКИ И МОДИФИКАТОРЫ
    // ==========================================
    
    // Объявление кастомных ошибок для экономии газа
    error UserAlreadyRegistered();
    error UserNotRegistered();
    error InsufficientFunds();

    // Модификатор: проверка, что пользователь зарегистрирован
    modifier onlyRegistered() {
        // Если длина логина 0, значит профиль пуст (не зарегистрирован)
        if (bytes(profiles[msg.sender].login).length == 0) {
            revert UserNotRegistered();
        }
        _; // Возврат к выполнению тела функции
    }

    // ==========================================
    // КОНСТРУКТОР
    // ==========================================
    
    constructor() {
        bankOwner = msg.sender;
        nextUserId = 1;
    }

    // ==========================================
    // ФУНКЦИИ СИСТЕМЫ
    // ==========================================

    /**
     * @dev Регистрация нового пользователя
     */
    function register(string memory _name, string memory _login, string memory _password) public {
        // Проверка: профиль не должен существовать
        if (bytes(profiles[msg.sender].login).length != 0) {
            revert UserAlreadyRegistered();
        }
        
        // Используем require с юникод-ошибкой для проверки длины логина
        require(bytes(_login).length > 2, unicode"❌ Логин слишком короткий!");
        
        profiles[msg.sender] = Profile({
            name: _name,
            id: nextUserId,
            login: _login,
            // Для безопасности хэшируем пароль встроенной функцией keccak256
            passwordHash: keccak256(abi.encodePacked(_password)),
            balance: 0
        });
        
        nextUserId++;
    }

    /**
     * @dev Просмотр информации профиля (доступно только зарегистрированным)
     */
    function getMyProfile() public view onlyRegistered returns (string memory, uint256, string memory, uint256) {
        Profile memory p = profiles[msg.sender];
        // Пароль не возвращаем в целях безопасности
        return (p.name, p.id, p.login, p.balance);
    }

    /**
     * @dev Положить деньги на вклад (пополнение баланса банка с кошелька)
     * Модификатор payable обязателен, чтобы функция могла принимать эфир.
     */
    function makeDeposit() public payable onlyRegistered {
        require(msg.value > 0, unicode"❌ Сумма вклада должна быть больше 0");
        
        // Обновляем баланс в профиле
        profiles[msg.sender].balance += msg.value;
        
        // Сохраняем информацию о вкладе в структуру Deposit
        deposits[msg.sender].userAddress = msg.sender;
        deposits[msg.sender].amount += msg.value;
    }

    /**
     * @dev Забрать деньги с вклада на свой личный кошелек
     */
    function withdrawDeposit(uint256 _amount) public onlyRegistered {
        require(_amount > 0, unicode"❌ Сумма снятия должна быть больше 0");
        
        uint256 currentBalance = profiles[msg.sender].balance;
        
        // Проверка достаточности средств с использованием кастомной ошибки
        if (currentBalance < _amount) {
            revert InsufficientFunds();
        }
        
        // Обязательно списываем средства ДО отправки (защита от Reentrancy атак)
        profiles[msg.sender].balance -= _amount;
        deposits[msg.sender].amount -= _amount;
        
        // Отправка эфира пользователю (msg.sender)
        payable(msg.sender).transfer(_amount);
    }
}
```

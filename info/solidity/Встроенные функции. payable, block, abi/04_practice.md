# Разбор практического задания: Продвинутый контракт Банка

В этом задании мы закрепим навыки работы с глобальными переменными (msg.sender, msg.value), единицами измерения эфира, а также научимся разделять права доступа между обычными пользователями и владельцем контракта.

## Условие задачи
- При деплое владелец вводит логин и пароль и получает 100 000 бонусного баланса.
- Обычный пользователь при регистрации получает 5 000 бонусного баланса.
- Структура пользователя: ID, Логин, Пароль, Баланс.
- **Функции владельца:** Снять весь ether с контракта, передать владение, посмотреть любого пользователя, найти пользователей по балансу.
- **Функции пользователя:** Регистрация, авторизация по логину и паролю, просмотр своего баланса.

## Решение

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract AdvancedBank {
    // Структура пользователя
    struct User {
        uint256 id;
        string login;
        bytes32 passwordHash; // Всегда хешируем пароли!
        uint256 balance;
        bool isRegistered;
    }

    // Состояние контракта
    address public owner;
    uint256 private nextUserId = 1;

    // Хранилища данных
    mapping(address => User) private users;
    // Вспомогательный массив для поиска пользователей (сохраняем адреса)
    address[] private userAddresses;

    // Модификаторы доступа
    modifier onlyOwner() {
        require(msg.sender == owner, unicode"Доступ запрещен: Вы не владелец");
        _;
    }

    modifier onlyUnregistered() {
        require(!users[msg.sender].isRegistered, unicode"Пользователь уже зарегистрирован");
        _;
    }

    // ==========================================
    // КОНСТРУКТОР
    // ==========================================

    // Деплой контракта с установкой логина и пароля владельца
    constructor(string memory _login, string memory _password) {
        owner = msg.sender;
        
        // Владелец также является пользователем банка с балансом 100 000
        users[msg.sender] = User({
            id: nextUserId,
            login: _login,
            passwordHash: keccak256(abi.encodePacked(_password)),
            balance: 100000,
            isRegistered: true
        });
        userAddresses.push(msg.sender);
        nextUserId++;
    }

    // ==========================================
    // ФУНКЦИИ ПОЛЬЗОВАТЕЛЯ
    // ==========================================

    // Регистрация обычного пользователя (бонус 5000)
    function register(string memory _login, string memory _password) public onlyUnregistered {
        users[msg.sender] = User({
            id: nextUserId,
            login: _login,
            passwordHash: keccak256(abi.encodePacked(_password)),
            balance: 5000,
            isRegistered: true
        });
        userAddresses.push(msg.sender);
        nextUserId++;
    }

    // Авторизация по логину и паролю
    // (Возвращает true в случае успеха, хотя в блокчейне "авторизация" обычно происходит просто по подписи транзакции msg.sender)
    function authorize(string memory _login, string memory _password) public view returns (bool) {
        require(users[msg.sender].isRegistered, unicode"Вы не зарегистрированы");
        
        bytes32 inputHash = keccak256(abi.encodePacked(_password));
        bytes32 storedHash = users[msg.sender].passwordHash;
        
        // Проверяем совпадение логина и пароля
        if (
            keccak256(abi.encodePacked(users[msg.sender].login)) == keccak256(abi.encodePacked(_login)) && 
            inputHash == storedHash
        ) {
            return true;
        }
        
        return false;
    }

    // Посмотреть свой баланс
    function getMyBalance() public view returns (uint256) {
        require(users[msg.sender].isRegistered, unicode"Вы не зарегистрированы");
        return users[msg.sender].balance;
    }

    // ==========================================
    // ФУНКЦИИ ВЛАДЕЛЬЦА (Только для owner)
    // ==========================================

    // Позволяет контракту принимать настоящий эфир
    receive() external payable {}

    // Снятие всего реального эфира с баланса контракта
    function withdrawAllEther() public onlyOwner {
        uint256 contractBalance = address(this).balance;
        require(contractBalance > 0, unicode"Баланс контракта пуст");
        
        // Использование call для перевода эфира
        (bool success, ) = msg.sender.call{value: contractBalance}("");
        require(success, unicode"Ошибка перевода эфира");
    }

    // Передача владения контрактом
    function transferOwnership(address _newOwner) public onlyOwner {
        require(_newOwner != address(0), unicode"Нельзя передать нулевому адресу");
        owner = _newOwner;
    }

    // Посмотреть информацию о любом пользователе (без пароля)
    function getUserInfo(address _userAddress) public view onlyOwner returns (uint256, string memory, uint256) {
        require(users[_userAddress].isRegistered, unicode"Пользователь не найден");
        User memory u = users[_userAddress];
        return (u.id, u.login, u.balance);
    }

    // Искать пользователей, у которых баланс равен или больше указанного
    // Возвращает массив адресов
    function searchUsersByMinBalance(uint256 _minBalance) public view onlyOwner returns (address[] memory) {
        // Сначала считаем, сколько таких пользователей (солидити не умеет делать динамические массивы в memory без фиксированной длины)
        uint256 count = 0;
        for (uint256 i = 0; i < userAddresses.length; i++) {
            if (users[userAddresses[i]].balance >= _minBalance) {
                count++;
            }
        }

        // Создаем массив нужной длины
        address[] memory result = new address[](count);
        uint256 currentIndex = 0;

        // Заполняем массив
        for (uint256 i = 0; i < userAddresses.length; i++) {
            if (users[userAddresses[i]].balance >= _minBalance) {
                result[currentIndex] = userAddresses[i];
                currentIndex++;
            }
        }

        return result;
    }
}
```

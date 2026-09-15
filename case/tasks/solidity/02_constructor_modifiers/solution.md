# Модуль 2: Конструктор, модификаторы и ошибки — Эталонное решение

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/// @title RoleAccessHub - Система контроля доступа на основе ролей и модификаторов
/// @notice Демонстрирует применение Custom Errors, параметризованных модификаторов и конструктора
contract RoleAccessHub {
    // 1. Определение кастомных ошибок (газоэффективная замена строковым require)
    error NotOwner(address caller);
    error AccountBlacklisted(address account);
    error ContractIsPaused();
    error InvalidAddress();
    error InsufficientAccessLevel(uint256 current, uint256 required);
    error InvalidLevelValue(uint256 level);

    // 2. Переменные состояния
    address public owner;
    bool public isPaused;
    mapping(address => bool) public isBlacklisted;
    mapping(address => uint256) public userAccessLevel;

    // 3. События
    event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);
    event PauseToggled(bool isPaused);
    event BlacklistUpdated(address indexed account, bool isBlacklisted);
    event AccessLevelUpdated(address indexed account, uint256 level);
    event CriticalOperationExecuted(address indexed operator, uint256 timestamp);

    // 4. Конструктор контракта
    constructor(address _initialOwner) {
        if (_initialOwner == address(0)) {
            revert InvalidAddress();
        }
        owner = _initialOwner;
        emit OwnershipTransferred(address(0), _initialOwner);
    }

    // 5. Модификаторы доступа
    modifier onlyOwner() {
        if (msg.sender != owner) {
            revert NotOwner(msg.sender);
        }
        _; // Выполнение целевой функции
    }

    modifier notBlacklisted(address _account) {
        if (isBlacklisted[_account]) {
            revert AccountBlacklisted(_account);
        }
        _;
    }

    modifier whenNotPaused() {
        if (isPaused) {
            revert ContractIsPaused();
        }
        _;
    }

    modifier requireLevel(uint256 _minLevel) {
        uint256 currentLevel = userAccessLevel[msg.sender];
        if (currentLevel < _minLevel) {
            revert InsufficientAccessLevel(currentLevel, _minLevel);
        }
        _;
    }

    // 6. Функции администрирования
    function togglePause() external onlyOwner {
        isPaused = !isPaused;
        emit PauseToggled(isPaused);
    }

    function setBlacklist(address _account, bool _status) external onlyOwner {
        if (_account == address(0)) revert InvalidAddress();
        isBlacklisted[_account] = _status;
        emit BlacklistUpdated(_account, _status);
    }

    function setUserLevel(address _account, uint256 _level) external onlyOwner {
        if (_account == address(0)) revert InvalidAddress();
        if (_level > 5) revert InvalidLevelValue(_level);

        userAccessLevel[_account] = _level;
        emit AccessLevelUpdated(_account, _level);
    }

    function transferOwnership(address _newOwner) external onlyOwner {
        if (_newOwner == address(0)) revert InvalidAddress();
        address oldOwner = owner;
        owner = _newOwner;
        emit OwnershipTransferred(oldOwner, _newOwner);
    }

    // 7. Защищенная бизнес-логика: цепочка модификаторов
    function criticalOperation() 
        external 
        whenNotPaused 
        notBlacklisted(msg.sender) 
        requireLevel(3) 
        returns (bool) 
    {
        emit CriticalOperationExecuted(msg.sender, block.timestamp);
        return true;
    }
}
```

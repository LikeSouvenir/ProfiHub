# Разбор практического задания: Система сотрудников компании

В этом задании объединим import, наследование, `virtual`/`override`, взаимодействие конструкторов, `super` и `new` в одном небольшом проекте.

## Условие задачи
Разбить проект на несколько файлов:
- `Employee.sol` — базовый контракт сотрудника (имя, зарплата, функция расчёта бонуса).
- `Manager.sol` — наследник `Employee`, добавляет список подчинённых и переопределяет расчёт бонуса.
- `CompanyFactory.sol` — контракт-фабрика, создающий новые контракты `Manager` через `new`.

## Решение с подробными комментариями

**Employee.sol**
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Employee {
    string public name;
    uint256 public salary;

    event Log(string message);

    // Базовый конструктор — требует имя и зарплату
    constructor(string memory _name, uint256 _salary) {
        name = _name;
        salary = _salary;
        emit Log(unicode"Создан сотрудник (Employee)");
    }

    // virtual — разрешаем переопределение в наследниках
    function calculateBonus() public virtual returns (uint256) {
        // Базовый бонус — 10% от зарплаты
        return salary / 10;
    }
}
```

**Manager.sol**
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

// Импортируем базовый контракт из соседнего файла
import "./Employee.sol";

// Manager наследует Employee
contract Manager is Employee {
    address[] public subordinates;

    // Передаём _name и _salary напрямую в конструктор родителя Employee
    constructor(string memory _name, uint256 _salary) Employee(_name, _salary) {
        emit Log(unicode"Создан менеджер (Manager)");
    }

    function addSubordinate(address _employee) public {
        subordinates.push(_employee);
    }

    // override — переопределяем логику расчёта бонуса для менеджера
    function calculateBonus() public override returns (uint256) {
        // Сначала берём базовый бонус родителя через super
        uint256 baseBonus = super.calculateBonus();

        // Плюс надбавка за каждого подчинённого
        uint256 teamBonus = subordinates.length * 50;

        return baseBonus + teamBonus;
    }
}
```

**CompanyFactory.sol**
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

// Фабрике нужен доступ к контракту Manager, чтобы деплоить его через new
import "./Manager.sol";

contract CompanyFactory {
    // Храним адреса всех созданных менеджеров
    Manager[] public managers;

    error EmptyName();

    /**
     * @dev Создаёт новый контракт Manager и сохраняет его адрес
     */
    function hireManager(string memory _name, uint256 _salary) public returns (address) {
        if (bytes(_name).length == 0) {
            revert EmptyName();
        }

        // new деплоит отдельный, самостоятельный контракт Manager
        // и автоматически вызывает всю цепочку конструкторов: Employee -> Manager
        Manager newManager = new Manager(_name, _salary);

        managers.push(newManager);
        return address(newManager);
    }

    function getManagersCount() public view returns (uint256) {
        return managers.length;
    }

    /**
     * @dev Вызывает calculateBonus() у конкретного менеджера по индексу
     */
    function getManagerBonus(uint256 _index) public returns (uint256) {
        Manager m = managers[_index];
        return m.calculateBonus();
    }
}
```

## Что происходит при вызове `hireManager("Иван", 1000)`
1. `CompanyFactory` вызывает `new Manager("Иван", 1000)`.
2. Так как `Manager is Employee`, сначала выполняется конструктор `Employee("Иван", 1000)` — сохраняет `name` и `salary`, выводит `Log("Создан сотрудник (Employee)")`.
3. Затем выполняется тело конструктора `Manager` — выводит `Log("Создан менеджер (Manager)")`.
4. В блокчейне появляется новый, независимый контракт `Manager` со своим адресом, который сохраняется в массив `managers` фабрики.
5. При вызове `calculateBonus()` у этого менеджера сработает переопределённая версия: `super.calculateBonus()` возьмёт базовые 10% от зарплаты из `Employee`, а сверху добавится бонус за подчинённых.

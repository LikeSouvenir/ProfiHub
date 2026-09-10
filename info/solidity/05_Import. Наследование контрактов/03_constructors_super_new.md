# Конструкторы при наследовании. Ключевые слова super и new

## Взаимодействие конструкторов при наследовании
Если родительский контракт имеет конструктор с аргументами, дочерний контракт **обязан** передать в него эти аргументы. Это можно сделать двумя способами.

### Способ 1: в сигнатуре дочернего конструктора
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Animal {
    string public name;

    constructor(string memory _name) {
        name = _name;
    }
}

contract Dog is Animal {
    string public breed;

    // Сразу после списка аргументов вызываем конструктор родителя Animal(_name)
    constructor(string memory _name, string memory _breed) Animal(_name) {
        breed = _breed;
    }
}
```

### Способ 2: прямо в списке наследования (is)
Используется, когда значение для родителя известно заранее и не зависит от аргументов дочернего конструктора:
```solidity
contract Cat is Animal("Кошка без имени") {
    // Animal("Кошка без имени") вызовется автоматически при деплое Cat
}
```

*Оба способа комбинировать для одного и того же родителя нельзя — нужно выбрать один.*

## Порядок вызова конструкторов
Конструкторы вызываются **от самого базового родителя к самому дочернему**, независимо от того, в каком порядке они физически написаны в коде:
```solidity
contract A {
    constructor() {
        // выполнится первым
    }
}

contract B is A {
    constructor() {
        // выполнится вторым
    }
}

contract C is B {
    constructor() {
        // выполнится последним — при деплое именно C
    }
}
```

## Коллизии имён и их обход
**Коллизия** возникает, когда контракт наследует от нескольких родителей, у которых есть переменные, функции или конструкторы с одинаковыми именами.

### Коллизия конструкторов
Если у нескольких родителей есть конструкторы с аргументами, дочерний контракт должен явно передать аргументы в каждый из них:
```solidity
contract Vehicle {
    string public brand;
    constructor(string memory _brand) {
        brand = _brand;
    }
}

contract Engine {
    uint public horsePower;
    constructor(uint _hp) {
        horsePower = _hp;
    }
}

// Явно вызываем оба родительских конструктора
contract Car is Vehicle, Engine {
    constructor(string memory _brand, uint _hp)
        Vehicle(_brand)
        Engine(_hp)
    {}
}
```

### Коллизия функций
Если два родителя объявляют функцию с одинаковой сигнатурой, дочерний контракт **обязан** переопределить её сам, явно решив, какую логику использовать (см. `override(X, Y)` из предыдущего конспекта). Просто унаследовать "одну из версий" молча — нельзя, компилятор потребует явного разрешения конфликта.

## Ключевое слово super
`super` даёт доступ к функциям **ближайшего родителя** в цепочке наследования — используется, когда в переопределённой функции нужно не просто заменить, а **дополнить** поведение родителя.

```solidity
contract Base {
    event Log(string message);

    function action() public virtual {
        emit Log("Base action");
    }
}

contract Middle is Base {
    function action() public virtual override {
        super.action(); // сначала выполняем логику Base
        emit Log("Middle action");
    }
}

contract Derived is Middle {
    function action() public override {
        super.action(); // вызовет Middle.action(), которая сама вызовет Base.action()
        emit Log("Derived action");
    }
}
```
При вызове `Derived.action()` события выведутся в порядке: `Base action` → `Middle action` → `Derived action`. Важно понимать: `super` вызывает функцию **следующего контракта по линеаризации C3**, а не обязательно "прямого родителя" в списке `is`.

## Ключевое слово new
`new` используется для **создания (деплоя) нового экземпляра контракта прямо из кода другого контракта**. Это отличается от наследования — наследование расширяет один контракт, а `new` создаёт отдельный, независимый контракт в блокчейне со своим собственным адресом.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Wallet {
    address public owner;

    constructor(address _owner) {
        owner = _owner;
    }
}

// Фабрика, которая создаёт новые контракты Wallet
contract WalletFactory {
    Wallet[] public wallets;

    function createWallet() public {
        // new вызывает конструктор Wallet и деплоит новый контракт в блокчейн
        Wallet newWallet = new Wallet(msg.sender);
        wallets.push(newWallet);
    }
}
```

Каждый вызов `createWallet()` создаёт полностью новый, самостоятельный контракт `Wallet` со своим адресом — в отличие от наследования, где дочерний контракт просто "вбирает" в себя код родителя в рамках одного и того же деплоя.

*Также `new` используется для создания массивов динамического размера в памяти, например `new uint[](5)`, но в контексте контрактов чаще всего речь именно про деплой новых экземпляров (шаблон "фабрика").*

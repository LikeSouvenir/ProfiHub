# Наследование контрактов. Ключевые слова virtual и override

Наследование — это механизм ООП, позволяющий одному контракту (наследнику / дочернему) получить все переменные, функции и модификаторы другого контракта (родительского / базового), не переписывая их заново.

## Базовый синтаксис
Для наследования используется ключевое слово `is`:
```solidity
contract Дочерний is Родительский {
    // ...
}
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Animal {
    string public name;

    constructor(string memory _name) {
        name = _name;
    }

    function sound() public pure virtual returns (string memory) {
        return "...";
    }
}

// Dog наследует всё от Animal: переменную name и функцию sound()
contract Dog is Animal {
    constructor(string memory _name) Animal(_name) {}

    function sound() public pure override returns (string memory) {
        return "Woof!";
    }
}
```

## Множественное наследование
Контракт может наследовать сразу от нескольких родителей — они перечисляются через запятую:
```solidity
contract Cat is Animal, Ownable {
    // получает функционал и Animal, и Ownable
}
```
Solidity линеаризует порядок наследования по алгоритму **C3-линеаризации** (как в Python): более "базовые" контракты указываются первыми, самые "производные" — последними в списке `is`.

## Зачем нужны virtual и override

По умолчанию функцию **нельзя переопределить** в контракте-наследнике. Чтобы явно разрешить и явно зафиксировать переопределение, Solidity вводит пару ключевых слов:

- **`virtual`** — ставится на функции **в родительском контракте**. Означает: "эта функция может быть переопределена в дочерних контрактах".
- **`override`** — ставится на функции **в дочернем контракте**. Означает: "эта функция переопределяет одноимённую функцию родителя".

Если убрать `virtual` у родителя, компилятор не даст поставить `override` в наследнике — попытка переопределить обычную функцию завершится ошибкой компиляции.

```solidity
contract Base {
    // Без virtual — переопределить эту функцию НЕЛЬЗЯ
    function fixedValue() public pure returns (uint) {
        return 100;
    }

    // С virtual — переопределить МОЖНО
    function greeting() public pure virtual returns (string memory) {
        return "Hello from Base";
    }
}

contract Child is Base {
    // Обязательно указываем override, иначе ошибка компиляции
    function greeting() public pure override returns (string memory) {
        return "Hello from Child";
    }
}
```

## Цепочка наследования (несколько уровней)
Функция может оставаться переопределяемой и дальше по цепочке, если снова пометить её `virtual` в дочернем контракте:
```solidity
contract A {
    function value() public pure virtual returns (uint) {
        return 1;
    }
}

contract B is A {
    // override — потому что переопределяем A
    // virtual — потому что разрешаем переопределить дальше, в C
    function value() public pure virtual override returns (uint) {
        return 2;
    }
}

contract C is B {
    function value() public pure override returns (uint) {
        return 3;
    }
}
```

## override при множественном наследовании
Если функция с одинаковым именем объявлена сразу в нескольких родителях, в `override` нужно явно перечислить их всех в скобках:
```solidity
contract X {
    function info() public pure virtual returns (string memory) {
        return "X";
    }
}

contract Y {
    function info() public pure virtual returns (string memory) {
        return "Y";
    }
}

// Указываем оба родителя, от которых переопределяем info()
contract Z is X, Y {
    function info() public pure override(X, Y) returns (string memory) {
        return "Z";
    }
}
```

## Переопределение переменных состояния
`virtual`/`override` применимы не только к функциям, но и к публичным переменным состояния (у них автоматически создаётся геттер, который можно переопределить):
```solidity
contract Base {
    uint public virtual price = 100;
}

contract Discounted is Base {
    uint public override price = 80;
}
```

# Абстрактные контракты (abstract contract)

Абстрактный контракт — это что-то среднее между обычным контрактом и интерфейсом. В нём можно смешивать **реализованные функции** (с телом) и **нереализованные** (без тела), которые обязаны доопределить наследники.

## Отличия от интерфейса и обычного контракта

| | Обычный contract | abstract contract | interface |
|---|---|---|---|
| Можно задеплоить напрямую | ✅ Да | ❌ Нет | ❌ Нет |
| Может содержать переменные состояния | ✅ Да | ✅ Да | ❌ Нет |
| Может содержать конструктор | ✅ Да | ✅ Да | ❌ Нет |
| Может содержать реализованные функции | ✅ Да | ✅ Да | ❌ Нет |
| Может содержать нереализованные функции | ❌ Нет | ✅ Да | ✅ Да (все) |

## Когда нужен abstract
Абстрактный контракт используют, когда есть **общая логика**, которую хочется переиспользовать, но при этом одна или несколько функций **обязательно должны быть реализованы по-разному** в каждом наследнике, и деплоить "базовую заготовку" саму по себе не имеет смысла.

## Синтаксис
Контракт объявляется ключевым словом `abstract` перед `contract`. Любая функция без тела внутри него обязательно помечается `virtual`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

abstract contract Shape {
    string public name;

    // Обычный, полностью реализованный конструктор — доступен наследникам
    constructor(string memory _name) {
        name = _name;
    }

    // Реализованная функция — общая для всех фигур
    function describe() public view returns (string memory) {
        return name;
    }

    // Нереализованная функция — каждая фигура считает площадь по-своему
    function area() public view virtual returns (uint256);
}
```

Попытка задеплоить `Shape` напрямую приведёт к ошибке компиляции — у контракта есть функция без реализации, значит он **не может существовать в блокчейне сам по себе**.

## Наследование от абстрактного контракта
Наследник обязан реализовать (`override`) все нереализованные функции родителя — иначе он тоже станет абстрактным:

```solidity
contract Circle is Shape {
    uint256 public radius;

    constructor(uint256 _radius) Shape("Circle") {
        radius = _radius;
    }

    // Обязательная реализация area()
    function area() public view override returns (uint256) {
        // Упрощённо: 3 * r^2 (без плавающей точки Solidity считает целыми числами)
        return 3 * radius * radius;
    }
}

contract Rectangle is Shape {
    uint256 public width;
    uint256 public height;

    constructor(uint256 _w, uint256 _h) Shape("Rectangle") {
        width = _w;
        height = _h;
    }

    function area() public view override returns (uint256) {
        return width * height;
    }
}
```

Оба контракта переиспользуют `name`, конструктор и `describe()` из `Shape`, но каждый по-своему считает `area()`.

## Абстрактный контракт может частично реализовывать интерфейс
Частая практика: контракт наследует интерфейс, реализует часть функций сразу (общую логику), а остальные оставляет `virtual` без тела — тем самым он тоже становится `abstract`:

```solidity
interface IShape {
    function area() external view returns (uint256);
    function perimeter() external view returns (uint256);
}

// Реализует area(), но не реализует perimeter() — значит контракт abstract
abstract contract PartialShape is IShape {
    function area() external view virtual override returns (uint256) {
        return 0; // заглушка по умолчанию
    }
    // perimeter() остаётся не реализованным
}
```

# Интерфейсы (interface)

Интерфейс — это особый тип контракта, который описывает **набор функций без их реализации**. Он отвечает на вопрос "что контракт умеет делать", но не говорит "как именно он это делает".

## Зачем нужны интерфейсы
- **Стандартизация.** Позволяют разным контрактам "договориться" об одинаковом наборе функций (пример — стандарт ERC20, о котором пойдёт речь дальше).
- **Взаимодействие с чужими контрактами.** Чтобы вызвать функцию другого, уже задеплоенного контракта, не обязательно иметь его полный исходный код — достаточно интерфейса с нужными сигнатурами.
- **Экономия места и абстракция.** Компилятору не нужно знать реализацию, чтобы сгенерировать код вызова.

## Правила объявления интерфейса
1. Объявляется ключевым словом `interface` вместо `contract`.
2. Все функции объявляются **без тела** — сразу точка с запятой после сигнатуры.
3. Все функции автоматически считаются `external`.
4. Интерфейс не может содержать переменные состояния, конструктор и реализованный код — только сигнатуры функций, а также события и кастомные ошибки.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IToken {
    // Только сигнатура, без тела функции
    function totalSupply() external view returns (uint256);

    function transfer(address _to, uint256 _amount) external returns (bool);

    // Интерфейс может объявлять события
    event Transfer(address indexed from, address indexed to, uint256 value);
}
```

## Как использовать интерфейс
Интерфейс "натягивается" на адрес уже существующего контракта — так можно вызывать его функции, даже не имея доступа к исходному коду:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./IToken.sol";

contract TokenSpender {
    // Функция принимает адрес любого контракта, реализующего IToken
    function spend(address _tokenAddress, address _to, uint256 _amount) public {
        // "Оборачиваем" адрес в интерфейс — теперь можно вызывать его функции
        IToken token = IToken(_tokenAddress);

        token.transfer(_to, _amount);
    }

    function checkSupply(address _tokenAddress) public view returns (uint256) {
        return IToken(_tokenAddress).totalSupply();
    }
}
```

Главное преимущество: `TokenSpender` умеет работать с **любым** контрактом, который реализует `IToken` — будь то токен твоей команды или чужой контракт из другого проекта, задеплоенный где-то в сети.

## Наследование от интерфейса
Обычный контракт может реализовать интерфейс через `is`, при этом обязан реализовать (с `override`) абсолютно все его функции:

```solidity
contract MyToken is IToken {
    uint256 private _totalSupply;

    function totalSupply() external view override returns (uint256) {
        return _totalSupply;
    }

    function transfer(address _to, uint256 _amount) external override returns (bool) {
        // логика перевода
        emit Transfer(msg.sender, _to, _amount);
        return true;
    }
}
```

*Обратите внимание: `virtual` в самом интерфейсе ставить не нужно — все функции интерфейса переопределяемы по умолчанию.*

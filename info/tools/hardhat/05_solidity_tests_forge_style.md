# Написание тестов на Solidity (встроенный Forge-стиль)

## Зачем тесты на Solidity, если уже есть тесты на JS
Тесты на JS (через ethers.js + Chai) удобны и читаемы, но у них есть накладные расходы: каждый вызов контракта проходит через JSON-RPC, сериализацию данных между JS и EVM. Тесты, написанные **прямо на Solidity**, выполняются внутри самой виртуальной машины Ethereum (EVM) без этого "моста" — за счёт этого работают значительно быстрее, особенно на больших наборах тестов (сотни и тысячи тестов).

Такой подход к тестированию популяризировал фреймворк **Foundry** (инструмент **Forge** — его часть, отвечающая именно за тестирование), а современные версии Hardhat добавили встроенную поддержку такого же стиля тестов на Solidity — без необходимости устанавливать Foundry отдельно.

## Соглашение об именовании
Тесты на Solidity пишутся как обычный контракт, но с рядом соглашений:
- Файл теста называется `ИмяКонтракта.t.sol` (суффикс `.t.` перед расширением).
- Тестовый контракт наследует `Test` из библиотеки `forge-std`.
- Каждая тестовая функция начинается с префикса `test` — именно по этому префиксу фреймворк распознаёт, какие функции запускать как тесты.
- Функция `setUp()` (если объявлена) выполняется перед **каждым** тестом — аналог `beforeEach` из мира JS-тестов.

## Установка forge-std
```bash
npm install --save-dev forge-std
```
```js
// hardhat.config.js
module.exports = {
    solidity: "0.8.24",
    // Дополнительная настройка для поддержки Solidity-тестов зависит от версии Hardhat —
    // актуальные шаги проверяются в официальной документации Hardhat
};
```

## Базовый пример: тест для Storage.sol
```solidity
// contracts/Storage.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract Storage {
    uint256 private value;

    event ValueChanged(uint256 newValue);

    function set(uint256 _value) public {
        value = _value;
        emit ValueChanged(_value);
    }

    function get() public view returns (uint256) {
        return value;
    }
}
```

```solidity
// contracts/Storage.t.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";
import "./Storage.sol";

contract StorageTest is Test {
    Storage public storageContract;

    // Выполняется перед КАЖДЫМ тестом — аналог beforeEach в JS
    function setUp() public {
        storageContract = new Storage();
    }

    // Имя функции обязательно начинается с "test"
    function testInitialValueIsZero() public view {
        assertEq(storageContract.get(), 0);
    }

    function testSetUpdatesValue() public {
        storageContract.set(42);
        assertEq(storageContract.get(), 42);
    }

    function testEmitsValueChangedEvent() public {
        // Указываем, какие именно параметры события ожидаем проверить
        vm.expectEmit(true, true, true, true);
        emit Storage.ValueChanged(42);

        storageContract.set(42);
    }
}
```

## Разбор ключевых элементов

### assertEq и другие ассерты
```solidity
assertEq(a, b);           // a должно быть равно b
assertTrue(condition);    // condition должно быть true
assertFalse(condition);   // condition должно быть false
assertGt(a, b);           // a > b
assertLt(a, b);           // a < b
assertApproxEqAbs(a, b, delta); // a приблизительно равно b с допустимой погрешностью delta
```
Если проверка не проходит, Forge-стиль тестов выводит подробное сообщение с фактическим и ожидаемым значением — так же, как `expect(...).to.equal(...)` в Chai.

### vm — специальный объект для управления тестовым окружением
`vm` (сокращение от "виртуальная машина") даёт доступ к специальным "чит-кодам" (cheatcodes), недоступным в обычных смарт-контрактах, но крайне полезным для тестов:
```solidity
vm.prank(someAddress);       // следующий вызов будет выполнен от имени someAddress
vm.deal(someAddress, 10 ether); // выдать тестовому адресу конкретный баланс ETH
vm.warp(block.timestamp + 30 days); // "перемотать" время вперёд
vm.expectRevert("Только владелец может менять значение"); // ожидаем, что следующий вызов ревертнется с этим текстом
```

### Проверка реверта
```solidity
function testRevertsIfNotOwner() public {
    vm.prank(address(0xBEEF)); // имитируем вызов от чужого адреса
    vm.expectRevert("Только владелец может менять значение");

    storageContract.set(100);
}
```

### Проверка от имени конкретного адреса (prank)
```solidity
function testOwnerCanUpdateValue() public {
    address owner = address(this); // тестовый контракт сам выступает "владельцем" по умолчанию

    vm.prank(owner);
    storageContract.set(99);

    assertEq(storageContract.get(), 99);
}
```

## Запуск Solidity-тестов
```bash
npx hardhat test solidity     # запустить только Solidity-тесты (Forge-стиль)
npx hardhat test              # в современных версиях Hardhat может запускать сразу оба типа тестов — и JS, и Solidity
```

## Когда выбирать JS-тесты, а когда Solidity-тесты
| | Тесты на JS (Mocha + Chai + ethers) | Тесты на Solidity (Forge-стиль) |
|---|---|---|
| Скорость выполнения | Медленнее (через JSON-RPC) | Значительно быстрее (внутри EVM напрямую) |
| Удобство для сложной логики сценариев (интеграция с фронтендом, комплексные пользовательские сценарии) | ✅ Удобнее | Менее удобно |
| Удобство для "фаззинга" (проверки на множестве случайных входных данных) | Ограниченно | ✅ Встроенная поддержка через `testFuzz_...` функции |
| Порог входа для команды без опыта Solidity | Ниже (обычный JS) | Выше (нужно знать Solidity глубже) |

На практике многие проекты используют **оба** подхода одновременно: быстрые, многочисленные модульные тесты на Solidity для логики самого контракта, и JS-тесты — для более сложных, комплексных сценариев, где удобнее оперировать данными в привычном JS-окружении.

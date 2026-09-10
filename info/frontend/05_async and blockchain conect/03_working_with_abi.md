# Работа с ABI

## Что такое ABI
**ABI (Application Binary Interface)** — это JSON-описание того, какие функции, события и параметры есть у смарт-контракта. Смарт-контракт в блокчейне хранится в виде скомпилированного байт-кода — набора машинных инструкций без единого читаемого имени функции. ABI — это своего рода "переводчик" и "инструкция", которая объясняет библиотеке вроде Web3.js: как называется функция, какие аргументы принимает и что возвращает, чтобы правильно закодировать вызов в байт-код и раскодировать ответ обратно.

Аналогия: ABI смарт-контракта — это как **интерфейс** из Solidity (см. соответствующий конспект), только описанный не кодом на Solidity, а в формате JSON, понятном JavaScript-библиотекам.

## Как выглядит ABI
Для простого контракта хранения числа:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Storage {
    uint256 private value;

    function set(uint256 _value) public {
        value = _value;
    }

    function get() public view returns (uint256) {
        return value;
    }
}
```

ABI будет выглядеть примерно так:
```json
[
    {
        "inputs": [{ "internalType": "uint256", "name": "_value", "type": "uint256" }],
        "name": "set",
        "outputs": [],
        "stateMutability": "nonpayable",
        "type": "function"
    },
    {
        "inputs": [],
        "name": "get",
        "outputs": [{ "internalType": "uint256", "name": "", "type": "uint256" }],
        "stateMutability": "view",
        "type": "function"
    }
]
```

## Разбор полей ABI
| Поле | Значение |
|---|---|
| `name` | Имя функции |
| `inputs` | Список входных параметров: имя и тип каждого |
| `outputs` | Список возвращаемых значений: тип каждого |
| `stateMutability` | `view`/`pure` — чтение без изменения состояния; `nonpayable` — изменяет состояние, но не принимает эфир; `payable` — может принимать эфир |
| `type` | `function` (обычная функция), `constructor`, `event` |

По `stateMutability` можно сразу понять, потребует ли вызов функции **транзакции** (и, соответственно, газа) или это просто **бесплатное чтение** данных из блокчейна:
- `view` / `pure` → чтение, вызывается через `.call()`, газ не тратится.
- `nonpayable` / `payable` → изменение состояния, вызывается через `.send()`, требует транзакции и газа.

## Откуда взять ABI
1. **Из компилятора Solidity / Remix.** После компиляции контракта в Remix ABI можно скопировать во вкладке **Compilation Details** (кнопка "ABI").
2. **Из Hardhat/Truffle.** После компиляции ABI автоматически сохраняется в JSON-файл артефакта контракта (папка `artifacts/`).
3. **Из блокчейн-эксплорера** (например, Etherscan) — если контракт верифицирован, там публикуется его исходный код и ABI.

## Использование ABI в Web3.js
```js
const { Web3 } = require("web3");
const web3 = new Web3("http://127.0.0.1:8545");

const contractAddress = "0xВАШ_АДРЕС_КОНТРАКТА";

const abi = [
    {
        inputs: [{ internalType: "uint256", name: "_value", type: "uint256" }],
        name: "set",
        outputs: [],
        stateMutability: "nonpayable",
        type: "function",
    },
    {
        inputs: [],
        name: "get",
        outputs: [{ internalType: "uint256", name: "", type: "uint256" }],
        stateMutability: "view",
        type: "function",
    },
];

// Создаём объект контракта, "натягивая" ABI на конкретный адрес
const storageContract = new web3.eth.Contract(abi, contractAddress);

async function main() {
    const accounts = await web3.eth.getAccounts();

    // Чтение (call) — бесплатно, без транзакции
    const currentValue = await storageContract.methods.get().call();
    console.log("Текущее значение:", currentValue);

    // Запись (send) — требует транзакции и аккаунта-отправителя
    await storageContract.methods.set(42).send({ from: accounts[0] });
    console.log("Значение обновлено на 42");
}

main();
```

## call() vs send() — главное различие
| | `.call()` | `.send()` |
|---|---|---|
| Меняет состояние блокчейна | ❌ Нет | ✅ Да |
| Требует газ | ❌ Нет | ✅ Да |
| Требует аккаунт-отправитель | ❌ Нет | ✅ Да (`from`) |
| Скорость выполнения | Мгновенно | Ждёт включения в блок |
| Используется для | Функций `view`/`pure` | Функций, изменяющих переменные состояния |

Попытка вызвать функцию, изменяющую состояние (например, `set`), через `.call()` не приведёт к ошибке — но и не изменит данные в блокчейне: `.call()` просто симулирует выполнение локально, не отправляя реальную транзакцию в сеть.

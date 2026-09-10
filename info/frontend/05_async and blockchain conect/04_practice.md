# Практика: geth + контракт Storage + запросы через Web3.js

Полный цикл: поднимаем локальную сеть на Geth, разворачиваем простой контракт `Storage`, а затем пишем на Web3.js три запроса — чтение, запись и снова чтение, чтобы убедиться, что значение действительно изменилось в блокчейне.

## Шаг 1. Поднимаем Geth
Повторяем шаги из конспекта «Установка и запуск своей сети на Geth» (папка `geth`): инициализируем `genesis.json`, создаём аккаунт с балансом, и запускаем ноду с HTTP API:

```bash
geth --datadir "" --dev --http --http.api="eth,web3,net" --http.corsdomain "*" --http.port 8545 --networkid 1337 --allow-insecure-unlock --unlock "ВАШ_АДРЕС" console
```

Проверяем, что нода доступна — в новом терминале:
```bash
geth attach http://127.0.0.1:8545
```
```javascript
eth.accounts // должен вывести список доступных аккаунтов
```

## Шаг 2. Контракт Storage
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Storage {
    uint256 private value;

    // Запись данных
    function set(uint256 _value) public {
        value = _value;
    }

    // Получение данных
    function get() public view returns (uint256) {
        return value;
    }
}
```

## Шаг 3. Деплой через Remix
1. Открываем [Remix IDE](https://remix.ethereum.org), вставляем код `Storage.sol`, компилируем.
2. Во вкладке **Deploy & run transactions** выбираем **Environment → Custom - External Http Provider**, указываем `http://127.0.0.1:8545` (наша локальная нода Geth).
3. Нажимаем **Deploy**, подтверждаем транзакцию.
4. Копируем **адрес контракта** и **ABI** (кнопка ABI во вкладке Compilation Details) — они понадобятся в скрипте.

## Шаг 4. Настройка проекта с Web3.js
```bash
mkdir storage-client && cd storage-client
npm init -y
npm install web3
```

**abi.json** (вставляем ABI, скопированный из Remix)
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

## Шаг 5. Скрипт с тремя запросами: get → set → get

**index.js**
```js
const { Web3 } = require("web3");
const abi = require("./abi.json");

// Подключаемся к нашей локальной ноде Geth
const web3 = new Web3("http://127.0.0.1:8545");

// Адрес контракта, полученный после деплоя в Remix
const contractAddress = "0xВСТАВЬТЕ_АДРЕС_ВАШЕГО_КОНТРАКТА";

const storage = new web3.eth.Contract(abi, contractAddress);

async function main() {
    const accounts = await web3.eth.getAccounts();
    const sender = accounts[0];

    // ==== 1. Запрос на получение данных (до записи) ====
    const initialValue = await storage.methods.get().call();
    console.log("1) Значение ДО записи:", initialValue);

    // ==== 2. Запрос на запись данных ====
    console.log("Отправляем транзакцию set(42)...");
    const receipt = await storage.methods.set(42).send({ from: sender });
    console.log("2) Транзакция подтверждена, хэш:", receipt.transactionHash);

    // ==== 3. Запрос на получение данных (после записи) ====
    const updatedValue = await storage.methods.get().call();
    console.log("3) Значение ПОСЛЕ записи:", updatedValue);
}

main().catch(error => {
    console.error("Произошла ошибка:", error);
});
```

Запуск:
```bash
node index.js
```

## Ожидаемый вывод в консоли
```
1) Значение ДО записи: 0
Отправляем транзакцию set(42)...
2) Транзакция подтверждена, хэш: 0xабвг...
3) Значение ПОСЛЕ записи: 42
```

## Разбор решения
1. **Первый `get()`** вызывается через `.call()` — это просто чтение текущего состояния контракта, без транзакции и без газа. Изначально `value` в контракте равно `0` (значение по умолчанию для `uint256` в Solidity).
2. **`set(42)`** вызывается через `.send({ from: sender })` — это уже полноценная транзакция, которая изменяет состояние блокчейна, поэтому обязательно нужно указать отправителя (`from`) и дождаться (`await`) получения чека (`receipt`) о том, что транзакция включена в блок.
3. **Второй `get()`** снова читает значение через `.call()` и показывает уже обновлённое `42` — подтверждая, что транзакция `set` действительно изменила данные, хранящиеся в контракте на блокчейне, а не просто локальную переменную в скрипте.

Такая последовательность (**чтение → запись → чтение**) — стандартный способ проверить, что взаимодействие с контрактом через Web3.js работает корректно от начала до конца.

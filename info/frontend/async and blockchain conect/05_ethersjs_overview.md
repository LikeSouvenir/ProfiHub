# Ethers.js: мини-обзор библиотеки

## Что такое Ethers.js
**Ethers.js** — ещё одна популярная JavaScript-библиотека для взаимодействия с Ethereum и совместимыми сетями, во многом решающая те же задачи, что и Web3.js: чтение данных из блокчейна, отправка транзакций, работа со смарт-контрактами. Отличается более компактным API, продуманной работой с типами (особенно в связке с TypeScript) и меньшим размером итоговой сборки.

На практике выбор между Web3.js и Ethers.js чаще определяется предпочтениями команды и требованиями конкретного проекта (некоторые фреймворки и инструменты "из коробки" ориентированы на одну из библиотек) — концептуально они делают одно и то же.

## Установка
```bash
npm install ethers
```
```js
const { ethers } = require("ethers");
// или: import { ethers } from "ethers";
```

## Провайдер (Provider)
Так же, как и в Web3.js, для подключения к ноде нужен провайдер:
```js
// Подключение к локальной ноде Geth
const provider = new ethers.JsonRpcProvider("http://127.0.0.1:8545");
```

## Получение базовой информации о сети
```js
async function checkNetwork() {
    const network = await provider.getNetwork();
    console.log("Сеть:", network.name, "chainId:", network.chainId);

    const blockNumber = await provider.getBlockNumber();
    console.log("Последний блок:", blockNumber);

    const balanceWei = await provider.getBalance("0xАдресАккаунта");
    console.log("Баланс (wei):", balanceWei);
}
```

## Signer — аккаунт, который подписывает транзакции
Ключевое отличие в архитектуре Ethers.js — явное разделение **Provider** (только чтение данных из сети) и **Signer** (может подписывать и отправлять транзакции от имени конкретного аккаунта):
```js
// Signer на основе приватного ключа
const signer = new ethers.Wallet("0xВАШ_ПРИВАТНЫЙ_КЛЮЧ", provider);

console.log("Адрес отправителя:", signer.address);
```
*Приватный ключ — секретная информация; в реальных проектах он никогда не хранится прямо в коде, а подключается через переменные окружения (`.env`) или менеджеры секретов.*

## Утилиты перевода единиц измерения
```js
console.log(ethers.parseEther("1"));            // 1 ETH -> в wei (BigInt)
console.log(ethers.formatEther(1000000000000000000n)); // wei -> "1.0" (строка в ETH)

console.log(ethers.isAddress("0xAbC123..."));   // проверка валидности адреса
```

## Отправка простой транзакции (перевод эфира)
```js
async function sendEther() {
    const tx = await signer.sendTransaction({
        to: "0xАдресПолучателя",
        value: ethers.parseEther("1"),
    });

    console.log("Транзакция отправлена, хэш:", tx.hash);

    const receipt = await tx.wait(); // ждём, пока транзакция попадёт в блок
    console.log("Транзакция подтверждена в блоке:", receipt.blockNumber);
}
```
Обратите внимание на двухшаговый паттерн: сначала `await tx` — отправка транзакции в сеть (возвращает объект с хэшем практически сразу), затем `await tx.wait()` — отдельное ожидание именно **подтверждения** (включения в блок).

## Работа со смарт-контрактами
```js
const contractAddress = "0xВАШ_АДРЕС_КОНТРАКТА";

const abi = [
    "function get() view returns (uint256)",
    "function set(uint256 _value)",
];

// Контракт для чтения (через provider) — методы вызываются как read-only
const contractRead = new ethers.Contract(contractAddress, abi, provider);

// Контракт для записи (через signer) — методы могут отправлять транзакции
const contractWrite = new ethers.Contract(contractAddress, abi, signer);

async function main() {
    const value = await contractRead.get();
    console.log("Текущее значение:", value);

    const tx = await contractWrite.set(42);
    await tx.wait();
    console.log("Значение обновлено");
}
```
Обратите внимание на удобный, "человекочитаемый" формат ABI — вместо длинного JSON можно передать массив строк с сигнатурами функций (Human-Readable ABI), что делает небольшие интеграции заметно компактнее.

## Сравнение ключевых различий Web3.js и Ethers.js
| | Web3.js | Ethers.js |
|---|---|---|
| Подключение к ноде | `new Web3(url)` | `new ethers.JsonRpcProvider(url)` |
| Чтение / запись | `.call()` / `.send()` | Один и тот же вызов метода — Ethers сам решает, читать или отправлять транзакцию, в зависимости от того, передан ли Signer |
| Работа с аккаунтом-отправителем | `from` в параметрах вызова | Явный объект **Signer** |
| Формат ABI | Только JSON | JSON или компактный Human-Readable ABI (массив строк) |
| Перевод единиц (ETH ↔ wei) | `web3.utils.toWei/fromWei` | `ethers.parseEther/formatEther` |

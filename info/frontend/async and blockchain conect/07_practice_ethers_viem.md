# Практика: тот же сценарий Storage на Ethers.js и Viem

Повторяем практическое задание из конспекта про Geth и Web3.js — тот же самый задеплоенный контракт `Storage` (get → set → get), но теперь на Ethers.js и на Viem, чтобы наглядно сравнить синтаксис всех трёх библиотек.

Предполагается, что контракт уже поднят и задеплоен так, как описано в практике про Geth: нода Geth запущена на `http://127.0.0.1:8545`, контракт `Storage` задеплоен через Remix, известны его **адрес** и **ABI**.

```solidity
// Напоминание кода контракта
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

## Вариант на Ethers.js

```bash
npm install ethers
```

**index.js**
```js
const { ethers } = require("ethers");

const provider = new ethers.JsonRpcProvider("http://127.0.0.1:8545");

// Приватный ключ аккаунта, от имени которого будем отправлять транзакцию
const signer = new ethers.Wallet("0xВАШ_ПРИВАТНЫЙ_КЛЮЧ", provider);

const contractAddress = "0xВСТАВЬТЕ_АДРЕС_ВАШЕГО_КОНТРАКТА";

// Human-Readable ABI — компактная запись сигнатур функций
const abi = [
    "function get() view returns (uint256)",
    "function set(uint256 _value)",
];

async function main() {
    // Контракт для чтения — достаточно provider
    const contractRead = new ethers.Contract(contractAddress, abi, provider);

    // Контракт для записи — нужен signer, чтобы подписать транзакцию
    const contractWrite = new ethers.Contract(contractAddress, abi, signer);

    // ==== 1. Запрос на получение данных (до записи) ====
    const initialValue = await contractRead.get();
    console.log("1) Значение ДО записи:", initialValue);

    // ==== 2. Запрос на запись данных ====
    console.log("Отправляем транзакцию set(42)...");
    const tx = await contractWrite.set(42);
    const receipt = await tx.wait(); // ждём включения транзакции в блок
    console.log("2) Транзакция подтверждена, хэш:", receipt.hash);

    // ==== 3. Запрос на получение данных (после записи) ====
    const updatedValue = await contractRead.get();
    console.log("3) Значение ПОСЛЕ записи:", updatedValue);
}

main().catch(console.error);
```

## Вариант на Viem

```bash
npm install viem
```

**index.js**
```js
import { createPublicClient, createWalletClient, http } from 'viem'
import { privateKeyToAccount } from 'viem/accounts'

const RPC_URL = "http://127.0.0.1:8545";
const contractAddress = "0xВСТАВЬТЕ_АДРЕС_ВАШЕГО_КОНТРАКТА";

const abi = [
    {
        name: "get",
        type: "function",
        stateMutability: "view",
        inputs: [],
        outputs: [{ name: "", type: "uint256" }],
    },
    {
        name: "set",
        type: "function",
        stateMutability: "nonpayable",
        inputs: [{ name: "_value", type: "uint256" }],
        outputs: [],
    },
];

// Клиент для ЧТЕНИЯ данных из блокчейна
const publicClient = createPublicClient({
    transport: http(RPC_URL),
});

// Аккаунт и клиент для ЗАПИСИ (отправки транзакций)
const account = privateKeyToAccount("0xВАШ_ПРИВАТНЫЙ_КЛЮЧ");
const walletClient = createWalletClient({
    account,
    transport: http(RPC_URL),
});

async function main() {
    // ==== 1. Запрос на получение данных (до записи) ====
    const initialValue = await publicClient.readContract({
        address: contractAddress,
        abi,
        functionName: "get",
    });
    console.log("1) Значение ДО записи:", initialValue);

    // ==== 2. Запрос на запись данных ====
    console.log("Отправляем транзакцию set(42)...");
    const hash = await walletClient.writeContract({
        address: contractAddress,
        abi,
        functionName: "set",
        args: [42],
    });

    // Ждём подтверждения транзакции через publicClient
    const receipt = await publicClient.waitForTransactionReceipt({ hash });
    console.log("2) Транзакция подтверждена, хэш:", receipt.transactionHash);

    // ==== 3. Запрос на получение данных (после записи) ====
    const updatedValue = await publicClient.readContract({
        address: contractAddress,
        abi,
        functionName: "get",
    });
    console.log("3) Значение ПОСЛЕ записи:", updatedValue);
}

main().catch(console.error);
```

## Сравнение решений на трёх библиотеках

| Шаг | Web3.js | Ethers.js | Viem |
|---|---|---|---|
| Подключение к ноде | `new Web3(url)` | `new ethers.JsonRpcProvider(url)` | `createPublicClient({ transport: http(url) })` |
| Аккаунт-отправитель | указывается в `.send({ from })` | `new ethers.Wallet(privateKey, provider)` | `privateKeyToAccount(privateKey)` + `createWalletClient` |
| Чтение (`get`) | `contract.methods.get().call()` | `contractRead.get()` | `publicClient.readContract({...})` |
| Запись (`set`) | `contract.methods.set(42).send({ from })` | `contractWrite.set(42)` + `tx.wait()` | `walletClient.writeContract({...})` + `publicClient.waitForTransactionReceipt({ hash })` |
| Ожидание подтверждения | Встроено в `.send()` (промис резолвится после receipt) | Отдельный шаг: `await tx.wait()` | Отдельный шаг: `await publicClient.waitForTransactionReceipt({ hash })` |

## Итог
Все три сценария выполняют абсолютно одну и ту же последовательность действий — **чтение → запись → повторное чтение** — и приводят к одному и тому же результату в блокчейне: значение в контракте `Storage` меняется с `0` на `42`. Разница только в синтаксисе и архитектуре самой библиотеки:
- **Web3.js** — один объект `contract.methods`, отправитель передаётся прямо в вызове.
- **Ethers.js** — чёткое разделение `Provider` (чтение) и `Signer`/`Wallet` (запись), с отдельным шагом `tx.wait()`.
- **Viem** — ещё более явное разделение через два разных клиента (`publicClient`/`walletClient`), с максимальной типобезопасностью при использовании TypeScript.

Понимание одной библиотеки делает переход на любую другую несложным — методология работы с блокчейном (RPC-провайдер + адрес контракта + ABI) остаётся одинаковой.

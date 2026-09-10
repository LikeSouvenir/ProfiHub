# Viem: мини-обзор библиотеки

## Что такое Viem
**Viem** — современная, легковесная TypeScript-библиотека для работы с Ethereum, появившаяся позже Web3.js и Ethers.js как их более быстрая и типобезопасная альтернатива. Активно используется в связке с популярными фреймворками (например, Wagmi для React) и уже стал стандартом де-факто во многих новых Web3-проектах.

**Ключевые особенности Viem:**
- Полностью написан на TypeScript, даёт точные автоматические типы для ABI и вызовов контрактов.
- Меньший размер бандла по сравнению с Web3.js и Ethers.js.
- Явное разделение **клиентов** на "публичный" (чтение) и "кошелёк" (запись/подпись) — концептуально похоже на Provider/Signer в Ethers.js, но реализовано через отдельные типы клиентов.
- Работает по принципу отдельных небольших функций вместо одного большого объекта с методами — это называется модульным (tree-shakable) API.

## Установка
```bash
npm install viem
```

## Публичный клиент (Public Client) — для чтения
```js
import { createPublicClient, http } from 'viem'

const publicClient = createPublicClient({
    transport: http("http://127.0.0.1:8545"), // подключение к локальной ноде Geth
});
```

## Получение базовой информации о сети
```js
async function checkNetwork() {
    const blockNumber = await publicClient.getBlockNumber();
    console.log("Последний блок:", blockNumber);

    const balance = await publicClient.getBalance({
        address: "0xАдресАккаунта",
    });
    console.log("Баланс (wei):", balance);
}
```

## Кошелёк-клиент (Wallet Client) — для отправки транзакций
```js
import { createWalletClient, http } from 'viem'
import { privateKeyToAccount } from 'viem/accounts'

const account = privateKeyToAccount("0xВАШ_ПРИВАТНЫЙ_КЛЮЧ");

const walletClient = createWalletClient({
    account,
    transport: http("http://127.0.0.1:8545"),
});
```

## Утилиты перевода единиц измерения
```js
import { parseEther, formatEther, isAddress } from 'viem'

console.log(parseEther("1"));              // 1 ETH -> wei (BigInt)
console.log(formatEther(1000000000000000000n)); // wei -> "1"

console.log(isAddress("0xAbC123...")); // проверка адреса
```

## Отправка простой транзакции
```js
async function sendEther() {
    const hash = await walletClient.sendTransaction({
        to: "0xАдресПолучателя",
        value: parseEther("1"),
    });

    console.log("Транзакция отправлена, хэш:", hash);

    // Ждём подтверждения через публичный клиент
    const receipt = await publicClient.waitForTransactionReceipt({ hash });
    console.log("Подтверждена в блоке:", receipt.blockNumber);
}
```
Обратите внимание на разделение ответственности: `walletClient` отправляет транзакцию и сразу возвращает её хэш, а ожиданием подтверждения занимается уже `publicClient` — та же логика разделения "запись" / "чтение", что и у Ethers.js, только оформленная через два разных объекта-клиента.

## Работа со смарт-контрактами
```js
const contractAddress = "0xВАШ_АДРЕС_КОНТРАКТА";

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

async function main() {
    // Чтение — через publicClient
    const value = await publicClient.readContract({
        address: contractAddress,
        abi,
        functionName: "get",
    });
    console.log("Текущее значение:", value);

    // Запись — через walletClient
    const hash = await walletClient.writeContract({
        address: contractAddress,
        abi,
        functionName: "set",
        args: [42],
    });

    await publicClient.waitForTransactionReceipt({ hash });
    console.log("Значение обновлено");
}
```

## Сравнение трёх библиотек
| | Web3.js | Ethers.js | Viem |
|---|---|---|---|
| Подключение (чтение) | `new Web3(url)` | `new ethers.JsonRpcProvider(url)` | `createPublicClient({ transport: http(url) })` |
| Подключение (запись) | тот же объект + `from` | отдельный `Signer`/`Wallet` | отдельный `createWalletClient` |
| Чтение контракта | `.methods.x().call()` | `contract.x()` (через provider) | `publicClient.readContract({...})` |
| Запись в контракт | `.methods.x().send({from})` | `contract.x()` (через signer) | `walletClient.writeContract({...})` |
| Типизация (TypeScript) | Базовая | Хорошая | Максимальная, "из коробки" |
| Размер сборки | Больше | Средний | Наименьший |
| Актуальность в новых проектах (2025-2026) | Используется, но чаще в legacy-проектах | Широко используется | Быстро растущий стандарт, особенно с Wagmi/React |

Все три библиотеки решают одну и ту же задачу — связать JavaScript-код с блокчейном через RPC-провайдера и ABI контракта — различаясь синтаксисом, архитектурой клиентов и уровнем поддержки TypeScript. Выбор конкретной библиотеки в реальном проекте обычно продиктован тем, что уже использует остальная кодовая база или фреймворк (например, Wagmi "из коробки" построен поверх Viem).

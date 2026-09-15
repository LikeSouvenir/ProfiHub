# Модуль 5: async/await и подключение к блокчейну — Эталонное решение

## Общий ABI (`abi.json`)

```json
[
    {
        "inputs": [
            { "internalType": "string",  "name": "_title",  "type": "string"  },
            { "internalType": "uint256", "name": "_points", "type": "uint256" }
        ],
        "name": "addEntry",
        "outputs": [],
        "stateMutability": "nonpayable",
        "type": "function"
    },
    {
        "inputs": [{ "internalType": "uint256", "name": "_index", "type": "uint256" }],
        "name": "getEntry",
        "outputs": [
            { "internalType": "string",  "name": "", "type": "string"  },
            { "internalType": "uint256", "name": "", "type": "uint256" },
            { "internalType": "address", "name": "", "type": "address" },
            { "internalType": "uint256", "name": "", "type": "uint256" }
        ],
        "stateMutability": "view",
        "type": "function"
    },
    {
        "inputs": [],
        "name": "getTotalEntries",
        "outputs": [{ "internalType": "uint256", "name": "", "type": "uint256" }],
        "stateMutability": "view",
        "type": "function"
    },
    {
        "inputs": [],
        "name": "getTotalPoints",
        "outputs": [{ "internalType": "uint256", "name": "", "type": "uint256" }],
        "stateMutability": "view",
        "type": "function"
    },
    {
        "anonymous": false,
        "inputs": [
            { "indexed": true,  "internalType": "address", "name": "author", "type": "address" },
            { "indexed": false, "internalType": "string",  "name": "title",  "type": "string"  },
            { "indexed": false, "internalType": "uint256", "name": "points", "type": "uint256" }
        ],
        "name": "EntryAdded",
        "type": "event"
    }
]
```

---

## Вариант 1 — Web3.js (`client-web3.js`)

```js
const { Web3 } = require("web3");
const abi = require("./abi.json");

const RPC_URL = "http://127.0.0.1:8545";
const CONTRACT_ADDRESS = "0xВАШ_АДРЕС_КОНТРАКТА";

// Вся логика в одной async-функции: await работает только внутри async
async function main() {
    try {
        // ===== 1. ПОДКЛЮЧЕНИЕ =====
        const web3 = new Web3(RPC_URL);

        // Три независимых запроса идут параллельно: Promise.all запускает их
        // одновременно, общее время равно самому долгому, а не сумме всех
        const [chainId, blockNumber, accounts] = await Promise.all([
            web3.eth.getChainId(),
            web3.eth.getBlockNumber(),
            web3.eth.getAccounts(),
        ]);

        const sender = accounts[0];
        const balanceWei = await web3.eth.getBalance(sender);

        console.log("=== Подключение (Web3.js) ===");
        console.log("chainId:", chainId);
        console.log("Последний блок:", blockNumber);
        console.log("Аккаунт:", sender);
        // Баланс приходит в wei — минимальной единице. fromWei переводит в ether
        console.log("Баланс:", web3.utils.fromWei(balanceWei, "ether"), "ETH");

        const contract = new web3.eth.Contract(abi, CONTRACT_ADDRESS);

        // ===== 2. ЧТЕНИЕ ДО ЗАПИСИ =====
        // .call() — вызов view-функции: выполняется локально на ноде,
        // не меняет состояние, не создаёт транзакцию и не стоит газа
        const [totalBefore, pointsBefore] = await Promise.all([
            contract.methods.getTotalEntries().call(),
            contract.methods.getTotalPoints().call(),
        ]);

        console.log("\n=== До записи ===");
        console.log("Записей:", totalBefore, "| баллов:", pointsBefore);

        // ===== 3. ЗАПИСЬ =====
        // .send() создаёт транзакцию: она подписывается, попадает в мемпул,
        // майнится и меняет состояние блокчейна — за это платится газ
        console.log("\n=== Отправка транзакции ===");
        const receipt = await contract.methods
            .addEntry("Трекер заданий", 20)
            .send({ from: sender, gas: 300000 });

        console.log("Хеш:", receipt.transactionHash);
        console.log("Блок:", receipt.blockNumber);
        console.log("Газ:", receipt.gasUsed);

        // ===== 6. СОБЫТИЕ =====
        const event = receipt.events?.EntryAdded;
        if (event) {
            console.log("\n=== Событие EntryAdded ===");
            console.log("author:", event.returnValues.author);
            console.log("title: ", event.returnValues.title);
            console.log("points:", event.returnValues.points);
        }

        // ===== 4. ЧТЕНИЕ ПОСЛЕ ЗАПИСИ =====
        const totalAfter = await contract.methods.getTotalEntries().call();
        console.log("\n=== После записи ===");
        console.log(`Записей было ${totalBefore}, стало ${totalAfter}`);

        const lastIndex = Number(totalAfter) - 1;
        const entry = await contract.methods.getEntry(lastIndex).call();

        // Кортеж из Solidity приходит как объект с числовыми ключами
        console.log("Последняя запись:");
        console.log("  title: ", entry[0]);
        console.log("  points:", entry[1]);
        console.log("  author:", entry[2]);
        // timestamp в блокчейне — секунды, в JS Date — миллисекунды
        console.log("  дата:  ", new Date(Number(entry[3]) * 1000).toLocaleString("ru-RU"));

        // ===== 5. ПОСЛЕДОВАТЕЛЬНО ПРОТИВ ПАРАЛЛЕЛЬНО =====
        console.log("\n=== Три записи последовательно ===");
        const writeStart = Date.now();

        // Транзакции с одного аккаунта отправляем строго по очереди:
        // у каждой свой nonce, и параллельный запуск даст конфликт
        // "nonce too low" или "replacement transaction underpriced"
        for (const [title, points] of [["Задание A", 10], ["Задание B", 15], ["Задание C", 25]]) {
            await contract.methods.addEntry(title, points).send({ from: sender, gas: 300000 });
            console.log(`  записано: ${title}`);
        }

        console.log(`Время записи: ${Date.now() - writeStart} мс`);

        console.log("\n=== Три чтения параллельно ===");
        const readStart = Date.now();

        const newTotal = Number(await contract.methods.getTotalEntries().call());

        // Чтение состояние не меняет, поэтому параллелить его безопасно и выгодно
        const lastThree = await Promise.all([
            contract.methods.getEntry(newTotal - 3).call(),
            contract.methods.getEntry(newTotal - 2).call(),
            contract.methods.getEntry(newTotal - 1).call(),
        ]);

        lastThree.forEach((e, i) => console.log(`  [${i}] ${e[0]} — ${e[1]} баллов`));
        console.log(`Время чтения: ${Date.now() - readStart} мс`);

        // ===== 7. ОБРАБОТКА ОШИБКИ КОНТРАКТА =====
        console.log("\n=== Намеренная ошибка ===");
        try {
            await contract.methods.getEntry(9999).call();
        } catch (error) {
            // Контракт откатился через require — скрипт это переживает
            console.error("Контракт отклонил вызов:", error.message.split("\n")[0]);
        }

        console.log("\nСкрипт дошёл до конца, несмотря на ошибку выше.");
    } catch (error) {
        // Сюда попадают сетевые сбои: нода недоступна, неверный адрес контракта
        console.error("Критическая ошибка:", error.message);
    } finally {
        // finally выполняется и после успеха, и после ошибки
        console.log("Сессия завершена");
    }
}

main();
```

---

## Вариант 2 — Ethers.js (`client-ethers.js`)

```js
const { ethers } = require("ethers");
const abi = require("./abi.json");

const RPC_URL = "http://127.0.0.1:8545";
const CONTRACT_ADDRESS = "0xВАШ_АДРЕС_КОНТРАКТА";

async function main() {
    try {
        // ===== 1. ПОДКЛЮЧЕНИЕ =====
        // Provider — только чтение. Signer — умеет подписывать транзакции
        const provider = new ethers.JsonRpcProvider(RPC_URL);
        const signer = await provider.getSigner(0); // нулевой аккаунт dev-ноды

        const [network, blockNumber, address] = await Promise.all([
            provider.getNetwork(),
            provider.getBlockNumber(),
            signer.getAddress(),
        ]);

        const balance = await provider.getBalance(address);

        console.log("=== Подключение (Ethers.js) ===");
        console.log("chainId:", network.chainId); // BigInt
        console.log("Последний блок:", blockNumber);
        console.log("Аккаунт:", address);
        console.log("Баланс:", ethers.formatEther(balance), "ETH");

        // Передав signer вместо provider, получаем контракт, умеющий писать
        const contract = new ethers.Contract(CONTRACT_ADDRESS, abi, signer);

        // ===== 2. ЧТЕНИЕ ДО ЗАПИСИ =====
        // В ethers v6 view-функции вызываются напрямую и возвращают BigInt
        const [totalBefore, pointsBefore] = await Promise.all([
            contract.getTotalEntries(),
            contract.getTotalPoints(),
        ]);

        console.log("\n=== До записи ===");
        console.log("Записей:", totalBefore.toString(), "| баллов:", pointsBefore.toString());

        // ===== 3. ЗАПИСЬ =====
        console.log("\n=== Отправка транзакции ===");
        const tx = await contract.addEntry("Трекер заданий", 20); // транзакция отправлена
        console.log("Хеш:", tx.hash, "— ждём подтверждения...");

        const receipt = await tx.wait(); // ждём включения в блок
        console.log("Блок:", receipt.blockNumber);
        console.log("Газ:", receipt.gasUsed.toString());

        // ===== 6. СОБЫТИЕ =====
        console.log("\n=== Событие EntryAdded ===");
        for (const log of receipt.logs) {
            try {
                const parsed = contract.interface.parseLog(log);
                if (parsed?.name === "EntryAdded") {
                    console.log("author:", parsed.args.author);
                    console.log("title: ", parsed.args.title);
                    console.log("points:", parsed.args.points.toString());
                }
            } catch {
                // Лог от другого контракта — ABI его не распознал, пропускаем
            }
        }

        // ===== 4. ЧТЕНИЕ ПОСЛЕ ЗАПИСИ =====
        const totalAfter = await contract.getTotalEntries();
        console.log("\n=== После записи ===");
        console.log(`Записей было ${totalBefore}, стало ${totalAfter}`);

        // BigInt нельзя смешивать с Number: 1n - 1 бросит TypeError.
        // Вычитаем литерал 1n того же типа
        const [title, points, author, timestamp] = await contract.getEntry(totalAfter - 1n);

        console.log("Последняя запись:");
        console.log("  title: ", title);
        console.log("  points:", points.toString());
        console.log("  author:", author);
        console.log("  дата:  ", new Date(Number(timestamp) * 1000).toLocaleString("ru-RU"));

        // ===== 5. ПОСЛЕДОВАТЕЛЬНО ПРОТИВ ПАРАЛЛЕЛЬНО =====
        console.log("\n=== Три записи последовательно ===");
        const writeStart = Date.now();

        for (const [t, p] of [["Задание A", 10], ["Задание B", 15], ["Задание C", 25]]) {
            const writeTx = await contract.addEntry(t, p);
            await writeTx.wait(); // дожидаемся каждой перед следующей — из-за nonce
            console.log(`  записано: ${t}`);
        }

        console.log(`Время записи: ${Date.now() - writeStart} мс`);

        console.log("\n=== Три чтения параллельно ===");
        const readStart = Date.now();
        const newTotal = await contract.getTotalEntries();

        const lastThree = await Promise.all([
            contract.getEntry(newTotal - 3n),
            contract.getEntry(newTotal - 2n),
            contract.getEntry(newTotal - 1n),
        ]);

        lastThree.forEach((e, i) => console.log(`  [${i}] ${e[0]} — ${e[1].toString()} баллов`));
        console.log(`Время чтения: ${Date.now() - readStart} мс`);

        // ===== 7. ОБРАБОТКА ОШИБКИ =====
        console.log("\n=== Намеренная ошибка ===");
        try {
            await contract.getEntry(9999);
        } catch (error) {
            // ethers распаковывает причину отката в поле reason
            console.error("Контракт отклонил вызов:", error.reason ?? error.shortMessage);
        }

        console.log("\nСкрипт дошёл до конца, несмотря на ошибку выше.");
    } catch (error) {
        console.error("Критическая ошибка:", error.message);
    } finally {
        console.log("Сессия завершена");
    }
}

main();
```

---

## Вариант 3 — Viem (`client-viem.js`)

```js
const {
    createPublicClient,
    createWalletClient,
    http,
    formatEther,
    parseAbi,
} = require("viem");
const abi = require("./abi.json");

const RPC_URL = "http://127.0.0.1:8545";
const CONTRACT_ADDRESS = "0xВАШ_АДРЕС_КОНТРАКТА";

// Описание локальной dev-сети
const localChain = {
    id: 1337,
    name: "Geth Dev",
    nativeCurrency: { name: "Ether", symbol: "ETH", decimals: 18 },
    rpcUrls: { default: { http: [RPC_URL] } },
};

async function main() {
    try {
        // ===== 1. ПОДКЛЮЧЕНИЕ =====
        // Viem разделяет клиентов по назначению: публичный читает, кошелёк пишет
        const publicClient = createPublicClient({ chain: localChain, transport: http(RPC_URL) });
        const walletClient = createWalletClient({ chain: localChain, transport: http(RPC_URL) });

        const [chainId, blockNumber, addresses] = await Promise.all([
            publicClient.getChainId(),
            publicClient.getBlockNumber(),
            walletClient.getAddresses(),
        ]);

        const account = addresses[0];
        const balance = await publicClient.getBalance({ address: account });

        console.log("=== Подключение (Viem) ===");
        console.log("chainId:", chainId);
        console.log("Последний блок:", blockNumber); // BigInt
        console.log("Аккаунт:", account);
        console.log("Баланс:", formatEther(balance), "ETH");

        // Viem не создаёт объект контракта: адрес и ABI передаются в каждый вызов
        const contractConfig = { address: CONTRACT_ADDRESS, abi };

        // ===== 2. ЧТЕНИЕ ДО ЗАПИСИ =====
        // readContract — чтение без транзакции и без газа
        const [totalBefore, pointsBefore] = await Promise.all([
            publicClient.readContract({ ...contractConfig, functionName: "getTotalEntries" }),
            publicClient.readContract({ ...contractConfig, functionName: "getTotalPoints" }),
        ]);

        console.log("\n=== До записи ===");
        console.log("Записей:", totalBefore, "| баллов:", pointsBefore);

        // ===== 3. ЗАПИСЬ =====
        console.log("\n=== Отправка транзакции ===");

        // simulateContract прогоняет вызов на ноде и заранее ловит revert,
        // не тратя газ. Это рекомендованный порядок в viem
        const { request } = await publicClient.simulateContract({
            ...contractConfig,
            functionName: "addEntry",
            args: ["Трекер заданий", 20n],
            account,
        });

        const hash = await walletClient.writeContract(request);
        console.log("Хеш:", hash, "— ждём подтверждения...");

        const receipt = await publicClient.waitForTransactionReceipt({ hash });
        console.log("Блок:", receipt.blockNumber);
        console.log("Газ:", receipt.gasUsed);

        // ===== 6. СОБЫТИЕ =====
        const logs = await publicClient.getContractEvents({
            ...contractConfig,
            eventName: "EntryAdded",
            blockHash: receipt.blockHash,
        });

        console.log("\n=== Событие EntryAdded ===");
        for (const log of logs) {
            console.log("author:", log.args.author);
            console.log("title: ", log.args.title);
            console.log("points:", log.args.points);
        }

        // ===== 4. ЧТЕНИЕ ПОСЛЕ ЗАПИСИ =====
        const totalAfter = await publicClient.readContract({
            ...contractConfig,
            functionName: "getTotalEntries",
        });

        console.log("\n=== После записи ===");
        console.log(`Записей было ${totalBefore}, стало ${totalAfter}`);

        // Все числа в viem — BigInt, поэтому индекс тоже BigInt-литерал
        const [title, points, author, timestamp] = await publicClient.readContract({
            ...contractConfig,
            functionName: "getEntry",
            args: [totalAfter - 1n],
        });

        console.log("Последняя запись:");
        console.log("  title: ", title);
        console.log("  points:", points);
        console.log("  author:", author);
        console.log("  дата:  ", new Date(Number(timestamp) * 1000).toLocaleString("ru-RU"));

        // ===== 5. ПОСЛЕДОВАТЕЛЬНО ПРОТИВ ПАРАЛЛЕЛЬНО =====
        console.log("\n=== Три записи последовательно ===");
        const writeStart = Date.now();

        for (const [t, p] of [["Задание A", 10n], ["Задание B", 15n], ["Задание C", 25n]]) {
            const { request: req } = await publicClient.simulateContract({
                ...contractConfig,
                functionName: "addEntry",
                args: [t, p],
                account,
            });
            const txHash = await walletClient.writeContract(req);
            await publicClient.waitForTransactionReceipt({ hash: txHash });
            console.log(`  записано: ${t}`);
        }

        console.log(`Время записи: ${Date.now() - writeStart} мс`);

        console.log("\n=== Три чтения параллельно ===");
        const readStart = Date.now();

        const newTotal = await publicClient.readContract({
            ...contractConfig,
            functionName: "getTotalEntries",
        });

        const lastThree = await Promise.all([
            publicClient.readContract({ ...contractConfig, functionName: "getEntry", args: [newTotal - 3n] }),
            publicClient.readContract({ ...contractConfig, functionName: "getEntry", args: [newTotal - 2n] }),
            publicClient.readContract({ ...contractConfig, functionName: "getEntry", args: [newTotal - 1n] }),
        ]);

        lastThree.forEach((e, i) => console.log(`  [${i}] ${e[0]} — ${e[1]} баллов`));
        console.log(`Время чтения: ${Date.now() - readStart} мс`);

        // ===== 7. ОБРАБОТКА ОШИБКИ =====
        console.log("\n=== Намеренная ошибка ===");
        try {
            await publicClient.readContract({
                ...contractConfig,
                functionName: "getEntry",
                args: [9999n],
            });
        } catch (error) {
            // shortMessage у viem — короткое человекочитаемое описание
            console.error("Контракт отклонил вызов:", error.shortMessage ?? error.message);
        }

        console.log("\nСкрипт дошёл до конца, несмотря на ошибку выше.");
    } catch (error) {
        console.error("Критическая ошибка:", error.message);
    } finally {
        console.log("Сессия завершена");
    }
}

main();
```

---

## README.md — сравнение библиотек

| Задача | Web3.js | Ethers.js | Viem |
|---|---|---|---|
| Подключение к RPC | `new Web3(url)` | `new ethers.JsonRpcProvider(url)` | `createPublicClient({ transport: http(url) })` |
| Подписант | `{ from: account }` в вызове | `await provider.getSigner()` | `createWalletClient(...)` + `account` |
| Объект контракта | `new web3.eth.Contract(abi, addr)` | `new ethers.Contract(addr, abi, signer)` | объекта нет: `{ address, abi }` в каждый вызов |
| Чтение | `contract.methods.f().call()` | `await contract.f()` | `publicClient.readContract({...})` |
| Запись | `contract.methods.f().send({from})` | `await contract.f()` → `tx.wait()` | `simulateContract` → `writeContract` → `waitForTransactionReceipt` |
| Wei → Ether | `web3.utils.fromWei(v, "ether")` | `ethers.formatEther(v)` | `formatEther(v)` |
| Ether → Wei | `web3.utils.toWei(v, "ether")` | `ethers.parseEther(v)` | `parseEther(v)` |
| Тип чисел | строка или BigInt (зависит от версии) | `BigInt` | `BigInt` везде |
| Размер бандла | самый крупный | средний | самый компактный, tree-shakable |
| Типизация | слабая | хорошая | строгая, выводится из ABI |

---

## Разбор решения

1. **`async/await` вместо цепочек `then`.** Асинхронный код читается как синхронный сверху вниз, а `try/catch` ловит отказы промисов так же, как обычные исключения. Для линейного сценария «подключиться → прочитать → записать → прочитать» это единственный удобный вариант.
2. **`call` против `send` — ключевое различие.** `call` выполняется локально на ноде: состояние не меняется, транзакция не создаётся, газ не тратится, ответ приходит мгновенно. `send` формирует транзакцию, она подписывается ключом, уходит в мемпул, попадает в блок и меняет состояние — за это платится газ, а результат приходит только после подтверждения. Ошибка новичка — ожидать от `send` возвращаемого значения функции: приходит `receipt`, а данные нужно читать отдельным `call` или брать из события.
3. **Почему записи идут последовательно, а чтения — параллельно.** У каждой транзакции с одного адреса свой `nonce`, назначаемый по порядку. Параллельная отправка приведёт к тому, что несколько транзакций получат одинаковый `nonce`, и нода отклонит их с `nonce too low`. Чтение состояние не меняет, поэтому `Promise.all` там безопасен и заметно сокращает время.
4. **wei и ether.** Блокчейн оперирует целыми числами wei (1 ETH = 10^18 wei), потому что чисел с плавающей точкой в EVM нет. Выводить пользователю сырые wei бессмысленно — отсюда `fromWei` / `formatEther`.
5. **`BigInt` в ethers v6 и viem.** Значения `uint256` не помещаются в `Number` без потери точности, поэтому библиотеки возвращают `BigInt`. Смешивать типы нельзя: `total - 1` бросит `TypeError`, нужно `total - 1n`. Для `new Date()` и вывода в консоль `BigInt` приводится через `Number()` — это безопасно для timestamp и индексов, но не для балансов.
6. **`simulateContract` в viem.** Перед записью вызов прогоняется на ноде: если контракт откатится, ошибка придёт сразу и бесплатно, без потери газа на неудачную транзакцию. Это встроенная в API страховка, которой в Web3.js приходится добиваться вручную через `estimateGas`.
7. **Два уровня `try/catch`.** Внешний ловит сетевые сбои (нода не поднята, неверный адрес). Внутренний, вокруг заведомо падающего вызова, изолирует ожидаемый `revert` — скрипт продолжает работу и корректно доходит до `finally`.
8. **Единый ABI для трёх библиотек.** ABI — это описание интерфейса контракта, не зависящее от клиента. Один и тот же `abi.json` обслуживает все три скрипта, что наглядно показывает: библиотеки различаются только обёрткой вокруг одних и тех же JSON-RPC вызовов.

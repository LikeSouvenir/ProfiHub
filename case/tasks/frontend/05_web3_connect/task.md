# Модуль 5: async/await и подключение к блокчейну — Практическое задание

## Проект: Клиент журнала заданий (TaskLogClient)

### Цель

Освоить асинхронное программирование на `async/await` с обработкой ошибок через `try/catch/finally`, последовательное и параллельное выполнение запросов, а также подключение к блокчейну из JavaScript: провайдер, аккаунт-подписант, ABI, различие между вызовом на чтение (`call`) и транзакцией (`send`), конвертацию единиц (`wei` ↔ `ether`), чтение событий. Одна и та же задача решается тремя библиотеками — **Web3.js**, **Ethers.js** и **Viem** — чтобы увидеть разницу в API.

### Подготовка окружения

1. Локальная нода Geth в dev-режиме:

   ```bash
   geth --dev --http --http.api="eth,web3,net,personal" --http.corsdomain "*" \
        --http.port 8545 --networkid 1337 --allow-insecure-unlock console
   ```

2. Проект и зависимости:

   ```bash
   mkdir tasklog-client && cd tasklog-client
   npm init -y
   npm install web3 ethers viem
   ```

3. Контракт `TaskLog.sol` разворачивается через Remix (Environment → Custom - External Http Provider → `http://127.0.0.1:8545`). Адрес и ABI сохраняются в `abi.json` и `.env`.

### Контракт для работы

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract TaskLog {
    struct Entry {
        string title;
        uint256 points;
        address author;
        uint256 timestamp;
    }

    Entry[] private entries;
    mapping(address => uint256) public solvedCount;

    event EntryAdded(address indexed author, string title, uint256 points);

    function addEntry(string calldata _title, uint256 _points) external {
        entries.push(Entry(_title, _points, msg.sender, block.timestamp));
        solvedCount[msg.sender] += 1;
        emit EntryAdded(msg.sender, _title, _points);
    }

    function getEntry(uint256 _index) external view returns (string memory, uint256, address, uint256) {
        require(_index < entries.length, "Index out of range");
        Entry storage e = entries[_index];
        return (e.title, e.points, e.author, e.timestamp);
    }

    function getTotalEntries() external view returns (uint256) {
        return entries.length;
    }

    function getTotalPoints() external view returns (uint256) {
        uint256 sum = 0;
        for (uint256 i = 0; i < entries.length; i++) {
            sum += entries[i].points;
        }
        return sum;
    }
}
```

### Техническое задание

Напишите три скрипта — `client-web3.js`, `client-ethers.js`, `client-viem.js`, — каждый из которых выполняет один и тот же сценарий.

1. **Подключение и проверка сети (async/await + try/catch):**

   - Создайте провайдер/клиент, указывающий на `http://127.0.0.1:8545`.

   - Асинхронно получите: `chainId`, номер последнего блока, список доступных аккаунтов (или адрес подписанта).

   - Выведите баланс первого аккаунта в ether — не в wei (используйте `fromWei` / `formatEther` / `formatEther` из viem).

   - Всё подключение обёрнуто в `try/catch`; в `finally` выводится `"Сессия завершена"`.

2. **Чтение до записи (`call`):**

   - Получите `getTotalEntries()` и `getTotalPoints()`.

   - Оба значения читаются **параллельно** через `Promise.all` (или `Promise.all` с промисами вызовов), а не последовательно.

   - Объясните комментарием, почему эти вызовы бесплатны и не создают транзакцию.

3. **Запись (`send` / транзакция):**

   - Вызовите `addEntry("Трекер заданий", 20)` от первого аккаунта.

   - Дождитесь подтверждения транзакции (`receipt` / `tx.wait()` / `waitForTransactionReceipt`).

   - Выведите хеш транзакции, номер блока и израсходованный газ.

4. **Чтение после записи:**

   - Повторно получите `getTotalEntries()` и убедитесь, что счётчик увеличился на единицу.

   - Получите последнюю запись через `getEntry(total - 1)` и выведите её поля; `timestamp` переведите в читаемую дату через `new Date(Number(ts) * 1000)`.

5. **Последовательные и параллельные операции (демонстрация разницы):**

   - Добавьте **три** записи подряд последовательно (`await` в цикле), замерьте время через `Date.now()`.

   - Прочитайте эти три записи **параллельно** через `Promise.all`, снова замерьте время.

   - Выведите обе длительности и поясните комментарием, почему транзакции нельзя отправлять параллельно с одного аккаунта (конфликт `nonce`).

6. **Событие `EntryAdded`:**

   - Получите событие из логов подтверждённой транзакции (`receipt.logs` / `receipt.events` / декодирование через ABI) и выведите значения `author`, `title`, `points`.

7. **Обработка ошибки:**

   - Вызовите `getEntry(9999)` — контракт откатится с `"Index out of range"`.

   - Поймайте ошибку в `try/catch` и выведите понятное сообщение, не роняя скрипт. Программа должна продолжить работу и корректно завершиться.

8. **Сравнительная таблица:**

   - В файле `README.md` оформите таблицу: что в каждой из трёх библиотек отвечает за провайдер, подписанта, создание объекта контракта, чтение, запись, конвертацию единиц, тип возвращаемых чисел.

### Критерии приёмки

- Ни один скрипт не использует `.then()` для основного потока — только `async/await`.
- Отказ контракта (`revert`) перехвачен и не завершает процесс аварийно.
- Значение баланса выводится в ether, а не «сырыми» wei.
- В ethers/viem учтено, что числа возвращаются как `BigInt` — арифметика не смешивает `BigInt` и `Number`.
- Все три скрипта дают одинаковый результат на одном и том же контракте.

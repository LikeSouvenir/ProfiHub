# Практика: симуляция и алерты для контракта TaskLog

## Задача

Взять контракт `TaskLog` (используется во всех предыдущих разделах: Frontend-модуль 5, Envio-практика) и построить вокруг него слой мониторинга из двух частей:

1. **Симуляция перед отправкой** — прежде чем пользователь подтвердит транзакцию `addEntry` в интерфейсе, приложение должно проверить её через Tenderly и показать точный результат.
2. **Алерт на подозрительную активность** — уведомление в Slack, если кто-то добавляет запись со слишком большим количеством баллов (потенциальная попытка накрутки).

## Напоминание: контракт

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
}
```

---

## Часть 1. Симуляция перед отправкой (Tenderly)

### Функция симуляции

```js
import { ethers } from "ethers";

const TASKLOG_ABI = [
    "function addEntry(string calldata _title, uint256 _points) external",
];

async function simulateAddEntry(userAddress, title, points, contractAddress) {
    const iface = new ethers.Interface(TASKLOG_ABI);
    const data = iface.encodeFunctionData("addEntry", [title, points]);

    const response = await fetch(
        `https://sepolia.gateway.tenderly.co/${import.meta.env.VITE_TENDERLY_NODE_KEY}`,
        {
            method: "POST",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify({
                id: 0,
                jsonrpc: "2.0",
                method: "tenderly_simulateTransaction",
                params: [
                    {
                        from: userAddress,
                        to: contractAddress,
                        data,
                        gas: "0x7a1200",
                    },
                    "latest",
                ],
            }),
        }
    );

    const { result, error } = await response.json();

    if (error) {
        // Отличаем сетевую ошибку от отката транзакции —
        // это два разных случая для интерфейса
        return { success: false, reason: error.message, isNetworkError: true };
    }

    return {
        success: result.status,
        gasUsed: parseInt(result.gasUsed, 16),
        logs: result.logs,
    };
}
```

### Интеграция в форму React

```jsx
import { useState } from "react";

function AddEntryForm({ contractAddress, userAddress, onConfirmed }) {
    const [title, setTitle] = useState("");
    const [points, setPoints] = useState(10);
    const [simulation, setSimulation] = useState(null);
    const [isSimulating, setIsSimulating] = useState(false);

    const handlePreview = async () => {
        setIsSimulating(true);

        const result = await simulateAddEntry(userAddress, title, points, contractAddress);
        setSimulation(result);
        setIsSimulating(false);
    };

    const handleConfirm = async () => {
        // Только после успешной симуляции разрешаем реальную отправку
        await onConfirmed({ title, points });
    };

    return (
        <div>
            <input value={title} onChange={(e) => setTitle(e.target.value)} placeholder="Задание" />
            <input
                type="number"
                value={points}
                onChange={(e) => setPoints(Number(e.target.value))}
            />

            <button onClick={handlePreview} disabled={isSimulating}>
                {isSimulating ? "Проверяем..." : "Проверить транзакцию"}
            </button>

            {simulation && (
                <div>
                    {simulation.success ? (
                        <>
                            <p>✓ Транзакция пройдёт успешно</p>
                            <p>Ожидаемый газ: {simulation.gasUsed}</p>
                            <button onClick={handleConfirm}>Подтвердить и отправить</button>
                        </>
                    ) : (
                        <p>✗ Транзакция отклонится: {simulation.reason}</p>
                    )}
                </div>
            )}
        </div>
    );
}
```

**Почему это лучше, чем просто отправить транзакцию.** Без предварительной симуляции пользователь платит газ за попытку, узнаёт об откате только после подтверждения в блоке — и деньги за неудачную попытку не возвращаются. Симуляция даёт тот же результат бесплатно и мгновенно.

---

## Часть 2. Алерт на подозрительную активность (OpenZeppelin Monitor)

### Установка и структура

```bash
git clone https://github.com/openzeppelin/openzeppelin-monitor
cd openzeppelin-monitor
./setup_and_run.sh
```

### config/networks/sepolia.json

```json
{
    "network_type": "EVM",
    "slug": "sepolia",
    "name": "Sepolia Testnet",
    "rpc_urls": [
        { "type_": "rpc", "url": "${SEPOLIA_RPC_URL}", "weight": 100 }
    ],
    "chain_id": 11155111,
    "block_time_ms": 12000,
    "confirmation_blocks": 2
}
```

> Подставьте сюда реальный RPC-эндпоинт из раздела Data API (Alchemy/QuickNode) — публичный RPC для продакшен-мониторинга не подходит из-за лимитов.

### config/monitors/tasklog_suspicious_points.json

```json
{
    "name": "tasklog_suspicious_points",
    "networks": ["sepolia"],
    "addresses": [
        { "address": "0xВашКонтрактTaskLog" }
    ],
    "match_conditions": {
        "events": [
            {
                "signature": "EntryAdded(address,string,uint256)",
                "expression": "points > 90"
            }
        ]
    },
    "triggers": ["slack_suspicious_entry"]
}
```

### config/triggers/slack_suspicious_entry.json

```json
{
    "slack_suspicious_entry": {
        "name": "Suspicious TaskLog Entry",
        "trigger_type": "slack",
        "config": {
            "slack_url": { "type": "plain", "value": "${SLACK_WEBHOOK_URL}" },
            "message": {
                "title": "⚠️ Подозрительная запись в TaskLog",
                "body": "Адрес ${event.args.author} добавил задание на ${event.args.points} баллов (порог: 90). Транзакция: ${transaction.hash}"
            }
        }
    }
}
```

### Второй монитор: массовое добавление записей одним адресом

Дополнительно отследим паттерн накрутки — один и тот же адрес добавляет много записей за короткое время. Встроенных условий для подсчёта частоты в `expression` нет, поэтому такая логика выносится в скрипт-триггер:

```json
{
    "check_frequency_script": {
        "name": "Check Entry Frequency",
        "trigger_type": "script",
        "config": {
            "script_path": "./scripts/check_frequency.py",
            "language": "python",
            "timeout_ms": 3000
        }
    }
}
```

```python
# scripts/check_frequency.py
import sys
import json
from datetime import datetime, timedelta

# Простое хранилище в файле — для учебного проекта достаточно;
# в продакшене здесь была бы полноценная БД
HISTORY_FILE = "./data/entry_history.json"
WINDOW_MINUTES = 5
MAX_ENTRIES_IN_WINDOW = 3

def main():
    match_data = json.loads(sys.stdin.read())
    author = match_data["event"]["args"]["author"]
    now = datetime.utcnow()

    try:
        with open(HISTORY_FILE, "r") as f:
            history = json.load(f)
    except FileNotFoundError:
        history = {}

    timestamps = [
        datetime.fromisoformat(ts) for ts in history.get(author, [])
        if datetime.fromisoformat(ts) > now - timedelta(minutes=WINDOW_MINUTES)
    ]
    timestamps.append(now)
    history[author] = [ts.isoformat() for ts in timestamps]

    with open(HISTORY_FILE, "w") as f:
        json.dump(history, f)

    # Возвращаем true, если превышен порог — это и определяет,
    # сработает ли уведомление
    is_suspicious = len(timestamps) > MAX_ENTRIES_IN_WINDOW
    print(json.dumps({"trigger": is_suspicious, "count": len(timestamps)}))

if __name__ == "__main__":
    main()
```

### Запуск

```bash
cargo run --release
```

---

## Критерии приёмки

- Форма добавления записи не даёт отправить транзакцию без предварительной успешной симуляции.
- Симуляция различает сетевую ошибку (недоступна нода) и отказ самой транзакции (revert) — интерфейс показывает разные сообщения.
- OpenZeppelin Monitor запущен и корректно слушает тестовую сеть с задеплоенным `TaskLog`.
- Ручное добавление записи с `points > 90` вызывает сообщение в Slack в течение одного блока подтверждения.
- Скрипт частоты корректно считает окно в 5 минут и не даёt ложных срабатываний при обычном использовании (1-2 записи).
- Ни один API-ключ или вебхук не закоммичен в репозиторий — все через переменные окружения.

## Дополнительное задание

Сравните архитектурно три уровня реакции на одно и то же событие «подозрительная запись в TaskLog»:

1. **Tenderly Alert** — уведомление после подтверждения транзакции.
2. **OpenZeppelin Monitor** — то же самое, но self-hosted и с собственным скриптом логики.
3. **Гипотетический Hypernative Transaction Guard** — блокировка ещё до включения в блок.

Обсудите в README: для учебного контракта чемпионата (не с реальными деньгами) какой уровень реакции оправдан, а какой избыточен? Что изменится, если `TaskLog` вместо баллов чемпионата хранил бы реальные финансовые активы?

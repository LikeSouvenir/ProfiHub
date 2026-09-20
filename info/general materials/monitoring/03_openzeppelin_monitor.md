# OpenZeppelin Monitor + Relayer: open-source замена Defender

## Важно: почему не Defender

Если в старом материале, курсе или туториале встречается **OpenZeppelin Defender** (Sentinels, Autotasks, Relayer как облачный сервис) — это устаревшая информация. Регистрация новых пользователей была закрыта 30 июня 2025 года, а 1 июля 2026 года сервис **полностью прекратил работу**.

Вместо него OpenZeppelin развивает два **открытых** (self-hosted) инструмента, написанных на Rust:

- **OpenZeppelin Monitor** — замена Sentinels: следит за событиями и транзакциями по условиям и шлёт уведомления.
- **OpenZeppelin Relayer** — замена облачного Relayer: отправляет транзакции с управлением nonce, газом и подписью ключа.

Оба разворачиваются на своей инфраструктуре (свой сервер или Docker) — то есть требуют немного больше настройки, чем облачная панель, но не имеют vendor lock-in и бесплатны.

## OpenZeppelin Monitor: установка

```bash
git clone https://github.com/openzeppelin/openzeppelin-monitor
cd openzeppelin-monitor
chmod +x setup_and_run.sh
./setup_and_run.sh
```

Скрипт автоматической установки:
- собирает приложение из исходников (нужен Rust, либо Docker для контейнерного запуска);
- копирует примеры конфигурации из `examples/` в рабочую папку `config/`;
- проверяет корректность конфигурации;
- по желанию сразу запускает монитор.

## Структура проекта

```
config/
├── networks/    # какие сети и RPC-эндпоинты слушать
├── monitors/    # что именно отслеживать (события, функции, условия)
└── triggers/    # куда слать уведомление при срабатывании
```

Все три файла — обычный JSON, редактируются вручную или копируются из `examples/config/` как отправная точка.

## Конфигурация сети

```json
{
    "network_type": "EVM",
    "slug": "sepolia",
    "name": "Sepolia Testnet",
    "rpc_urls": [
        { "type_": "rpc", "url": "https://rpc.sepolia.org", "weight": 100 }
    ],
    "chain_id": 11155111,
    "block_time_ms": 12000,
    "confirmation_blocks": 1
}
```

> Для продакшена документация прямо рекомендует не использовать публичный RPC — брать выделенный от Alchemy, QuickNode или другого провайдера из раздела Data API, чтобы не упереться в rate limit.

## Конфигурация монитора: пример на TaskLog

Отслеживаем событие `EntryAdded` контракта `TaskLog` из практики предыдущих разделов и шлём уведомление при добавлении записи с крупными баллами:

```json
{
    "name": "tasklog_large_entry",
    "networks": ["sepolia"],
    "addresses": [
        { "address": "0xВашКонтрактTaskLog" }
    ],
    "match_conditions": {
        "events": [
            {
                "signature": "EntryAdded(address,string,uint256)",
                "expression": "points > 50"
            }
        ]
    },
    "trigger_conditions": [],
    "triggers": ["slack_notification"]
}
```

Поле `expression` — это то, что в Defender называлось «autotask condition»: произвольное условие по параметрам события, вычисляемое без написания отдельного скрипта.

## Конфигурация триггера — уведомление в Slack

```json
{
    "slack_notification": {
        "name": "Slack Alert",
        "trigger_type": "slack",
        "config": {
            "slack_url": {
                "type": "plain",
                "value": "https://hooks.slack.com/services/A/B/C"
            },
            "message": {
                "title": "Новая запись в TaskLog",
                "body": "Задание на ${event.args.points} баллов добавлено адресом ${event.args.author}. Транзакция: ${transaction.hash}"
            }
        }
    }
}
```

Шаблонные переменные (`${event.args.points}`, `${transaction.hash}`) подставляются из данных сработавшего события — так же, как переменные шаблонного литерала в JS, только на уровне конфигурации.

Поддерживаемые каналы: email, Slack, Discord, Telegram, вебхук, а также кастомный скрипт (Python, JavaScript или Bash), которому передаются данные о совпадении в JSON.

## Запуск

```bash
cargo run --release
# либо через Docker:
docker compose up
```

Монитор работает постоянно, опрашивая сеть с интервалом `block_time_ms` и сверяя новые блоки с условиями всех активных мониторов.

## Пользовательский скрипт как триггер

Когда встроенных типов уведомлений недостаточно — например, нужно записать событие в свою базу данных, а не просто отправить сообщение:

```json
{
    "custom_db_write": {
        "name": "Write to Database",
        "trigger_type": "script",
        "config": {
            "script_path": "./scripts/save_entry.py",
            "language": "python",
            "timeout_ms": 5000
        }
    }
}
```

Скрипт получает данные о совпадении через `stdin` в формате JSON и должен уложиться в `timeout_ms`, иначе будет принудительно завершён.

## OpenZeppelin Relayer: отправка транзакций из кода

Вторая часть связки — когда монитор должен не просто уведомить, а **автоматически отправить транзакцию** (например, поставить контракт на паузу при обнаружении аномалии).

```bash
git clone https://github.com/openzeppelin/openzeppelin-relayer
cd openzeppelin-relayer
docker compose up
```

Relayer поднимает HTTP API, через который можно отправлять транзакции, не храня приватный ключ в коде приложения — ключ живёт в конфигурации самого Relayer-сервиса:

```js
async function sendPauseTransaction(relayerUrl, apiKey) {
    const response = await fetch(`${relayerUrl}/api/v1/relayers/sepolia/transactions`, {
        method: "POST",
        headers: {
            "Content-Type": "application/json",
            Authorization: `Bearer ${apiKey}`,
        },
        body: JSON.stringify({
            to: "0xВашКонтрактTaskLog",
            data: "0x8456cb59", // селектор функции pause()
            gas_limit: 100000,
        }),
    });

    return response.json();
}
```

Relayer сам берёт на себя управление `nonce`, оценку газа и повторную отправку при застревании транзакции — те же задачи, которые в разделе про Web3 (модуль 5 Frontend) решались вручную через `await tx.wait()`.

## Сравнение: Defender (закрыт) vs открытые инструменты

| | Defender (закрыт) | OpenZeppelin Monitor + Relayer |
|---|---|---|
| Хостинг | облако OpenZeppelin | self-hosted (свой сервер/Docker) |
| Конфигурация | веб-панель | JSON-файлы |
| Стоимость | подписка | бесплатно (платите только за инфраструктуру) |
| Быстрый старт | быстрее — не нужен свой сервер | требует поднять и поддерживать сервис |
| Управляемый вариант | — | OpenZeppelin предлагает Managed Service по запросу, если не хочется самостоятельно администрировать |

## Ключевые моменты для практики

1. **Всегда проверяйте дату материала по OpenZeppelin Defender.** Любой пример с `defender-sdk`, `SentinelClient`, `AutotaskClient` — устаревший код для закрытого сервиса.
2. **Конфигурация через JSON — весь смысл файлов `networks/monitors/triggers`.** Это ближе к тому, как настраивается `config.yaml` у Envio, чем к SaaS-панели — тот же принцип «инфраструктура как код».
3. **`expression` в мониторе — это условие без написания скрипта.** Полноценный скрипт (Python/JS/Bash) нужен только для нестандартной логики, которую нельзя выразить простым выражением.
4. **Monitor и Relayer — разные инструменты для разных задач.** Monitor только наблюдает и уведомляет; для автоматической реакции транзакцией нужен отдельно поднятый Relayer.

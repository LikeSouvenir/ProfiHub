# Forta: децентрализованная сеть детекции аномалий

## Чем отличается от Tenderly и OpenZeppelin Monitor

Tenderly и OpenZeppelin Monitor реагируют на **заранее известные условия**: «если событие X содержит значение больше Y — уведомить». Это отлично работает, когда заранее понятно, что именно искать.

**Forta** решает другую задачу — обнаружение атак, которые заранее не были описаны условием. Вместо статичных правил используются **детекторные боты**: небольшие программы, анализирующие паттерны поведения (аномальные последовательности вызовов, подозрительные графы переводов, нетипичное для адреса поведение) и вычисляющие вероятность того, что происходит атака.

## Архитектура сети

```
Блокчейн → Scan Node → Detection Bots → Alert → Forta App / Public API / подписчики
```

- **Scan Node** — узел сети, который получает каждый новый блок и транзакцию и передаёт их ботам-детекторам.
- **Detection Bot** — код (на JS/TS/Python через готовые SDK), который анализирует переданные данные и решает, генерировать ли алерт.
- **Alert** — результат срабатывания бота, публикуется в публичный реестр (если бот не помечен как приватный).

Сеть децентрализована: любой может поднять scan node, застейкав токен FORT, и любой разработчик может опубликовать своего бота. Часть ботов, признанных сообществом особо ценными, объединена в **Premium Feeds** — например, готовый детектор скама.

## Готовые боты вместо написания своего

Для большинства типовых угроз (фишинг, rug pull, аномалии конкретных протоколов вроде Lido или Compound) уже существуют опубликованные боты — их не нужно писать самостоятельно, достаточно подписаться на алерты.

## Написание собственного детекторного бота

Установка CLI и инициализация проекта:

```bash
npm install -g forta-agent
forta-agent init --typescript
```

Базовая структура бота — обработчик транзакции, возвращающий массив findings (находок):

```ts
import { Finding, FindingSeverity, FindingType, HandleTransaction, TransactionEvent } from "forta-agent";

const LARGE_TRANSFER_THRESHOLD = "50000000000000000000"; // 50 токенов, с учётом decimals

const handleTransaction: HandleTransaction = async (txEvent: TransactionEvent) => {
    const findings: Finding[] = [];

    // Перебираем все события Transfer в транзакции
    const transferEvents = txEvent.filterLog(
        "event Transfer(address indexed from, address indexed to, uint256 value)"
    );

    for (const transferEvent of transferEvents) {
        const { value } = transferEvent.args;

        if (value.gte(LARGE_TRANSFER_THRESHOLD)) {
            findings.push(
                Finding.fromObject({
                    name: "Крупный перевод токена",
                    description: `Перевод на сумму ${value.toString()} обнаружен в транзакции ${txEvent.hash}`,
                    alertId: "LARGE-TRANSFER-1",
                    severity: FindingSeverity.Medium,
                    type: FindingType.Info,
                    metadata: {
                        from: transferEvent.args.from,
                        to: transferEvent.args.to,
                        value: value.toString(),
                    },
                })
            );
        }
    }

    return findings;
};

export default { handleTransaction };
```

**Ключевая идея:** бот не пишет напрямую в базу и не шлёт уведомления сам — он лишь возвращает `Finding`. Дальше сеть сама разносит находку в публичный реестр и подписчикам.

## Локальное тестирование бота

```bash
forta-agent run --tx 0xХешТранзакцииДляПроверки
```

Прогоняет обработчик на реальной исторической транзакции без деплоя в сеть — быстрый способ проверить логику перед публикацией.

## Публикация бота в сеть

```bash
forta-agent publish
```

Бот упаковывается в Docker-образ и публикуется в открытый реестр — код бота становится общедоступным (приватные секреты в боте хранить нельзя).

## Получение алертов из своего приложения

Алерты можно получать без написания собственного бота — просто подписавшись на публичный GraphQL API и фильтруя по интересующему боту или адресу:

```js
async function getRecentAlerts(botId) {
    const response = await fetch("https://api.forta.network/graphql", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
            query: `
                query {
                    alerts(input: { bots: ["${botId}"], first: 10 }) {
                        alerts {
                            alertId
                            name
                            description
                            severity
                            createdAt
                        }
                    }
                }
            `,
        }),
    });

    const { data } = await response.json();
    return data.alerts.alerts;
}
```

## Ключевые моменты для практики

1. **Forta — не замена Tenderly/Monitor, а дополнение.** Условные алерты (Tenderly, OZ Monitor) хороши для известных сценариев; Forta полезна там, где заранее неизвестно, что именно искать — общие паттерны аномального поведения.
2. **Бот возвращает `Finding`, а не сам решает, как уведомлять.** Логика доставки (email, Slack, дашборд) — забота сети и подписчиков, а не бота.
3. **Локальный прогон на исторической транзакции — обязательный шаг перед публикацией.** `forta-agent run --tx` позволяет проверить логику на реальных данных без риска ложных срабатываний в сети.
4. **Публичный бот — публичный код.** Нельзя хранить в боте секреты или проприетарную логику детекции, которую не хочется раскрывать конкурентам — для таких случаев есть режим приватного бота, но и в нём стоит быть осторожным.

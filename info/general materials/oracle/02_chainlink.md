# Chainlink: Data Feeds и Runtime Environment (CRE)

## Что осталось, а что заменилось

Chainlink — крупнейшая и старейшая сеть оракулов. Исторически продуктовая линейка была разбита на отдельные сервисы: Data Feeds (курсы активов), Functions (произвольный внешний код), Automation (автоматический вызов функций), VRF (случайность), CCIP (межсетевые сообщения).

**В 2026 году архитектура изменилась.** Functions и Automation как самостоятельные продукты свёрнуты, их функциональность объединена в новый продукт — **Chainlink Runtime Environment (CRE)**. Data Feeds при этом остаются отдельным, самым зрелым и никуда не девшимся продуктом — именно на них строится подавляющее большинство DeFi-протоколов.

| Продукт | Статус | Что делать |
|---|---|---|
| **Data Feeds** | Активен | Используется напрямую, как и раньше |
| ~~Functions~~ | Свёрнут 1 сентября 2026 | Мигрировать на CRE (HTTP-триггер + `runtime.http`) |
| ~~Automation~~ | Свёрнут (v1.x — 30 июня, v2.1 — 31 июля 2026) | Мигрировать на CRE (Cron/Log-триггер) |
| **CRE** | Активен | Новая точка входа для произвольной логики и автоматизации |

---

## Часть 1. Data Feeds — чтение курса актива

### Принцип

Data Feeds — классическая push-модель: децентрализованная сеть нод (DON) постоянно публикует агрегированное значение в контракт-агрегатор, который лежит по фиксированному адресу в каждой сети. Контракту достаточно вызвать `view`-функцию — без газа, без ожидания.

### Чтение курса в Solidity

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {AggregatorV3Interface} from "@chainlink/contracts/src/v0.8/shared/interfaces/AggregatorV3Interface.sol";

contract PriceConsumer {
    // Адрес контракта-агрегатора ETH/USD в сети Sepolia.
    // Для каждой сети и каждой пары свой адрес — список в документации Chainlink
    AggregatorV3Interface internal priceFeed;

    constructor() {
        priceFeed = AggregatorV3Interface(0x694AA1769357215DE4FAC081bf1f309aDC325306);
    }

    function getLatestPrice() public view returns (int256) {
        (
            /* uint80 roundId */,
            int256 answer,
            /* uint256 startedAt */,
            uint256 updatedAt,
            /* uint80 answeredInRound */
        ) = priceFeed.latestRoundData();

        // Курс без проверки свежести — плохая практика для реального протокола.
        // Ниже показано, как правильно проверять updatedAt
        return answer;
    }
}
```

### Обязательная проверка свежести данных

Это главная ошибка новичков: `latestRoundData()` вернёт **последнее известное значение**, даже если фид давно не обновлялся (например, из-за сбоя сети). Использовать устаревшую цену для расчёта ликвидации — прямой путь к эксплойту.

```solidity
function getSafePrice(uint256 maxAgeSeconds) public view returns (int256) {
    (, int256 answer, , uint256 updatedAt, ) = priceFeed.latestRoundData();

    require(answer > 0, "Invalid price");
    require(block.timestamp - updatedAt <= maxAgeSeconds, "Price is stale");

    return answer;
}
```

### Количество знаков после запятой

Курс приходит как целое число с фиксированным числом десятичных знаков, которое **отличается между фидами** — не предполагайте 18, как в большинстве ERC-20:

```solidity
function getDecimals() public view returns (uint8) {
    return priceFeed.decimals(); // для большинства USD-фидов — 8
}
```

### Ключевые моменты

1. **`view`-вызов, без газа за чтение.** Данные уже лежат в storage — в этом суть push-модели.
2. **Всегда проверяйте `updatedAt`.** Отсутствие этой проверки — самая частая уязвимость в учебных и даже реальных проектах.
3. **`decimals()` разный у разных фидов** — не хардкодьте 18.
4. **Адрес фида уникален для каждой пары и каждой сети** — берётся из официального списка адресов в документации, никогда не угадывается.

---

## Часть 2. От Functions/Automation к CRE

### Модель CRE: Workflow

CRE вводит единое понятие — **Workflow**: TypeScript- или Go-проект, скомпилированный в WebAssembly и развёрнутый в децентрализованной сети (DON). Workflow строится по схеме «триггер → обработчик»:

- **Trigger** — что запускает выполнение: расписание (Cron), HTTP-запрос, или событие в блокчейне (EVM Log).
- **Callback** — функция с вашей бизнес-логикой, вызывается при срабатывании триггера.
- Внутри callback доступны клиенты для HTTP-запросов, чтения/записи в блокчейн, секретов.

Это закрывает разом обе прежние задачи: то, что раньше делал Functions (принести внешние данные), и то, что делал Automation (вызвать функцию по условию) — теперь один и тот же механизм, отличается только тип триггера.

### Установка

```bash
npm install -g @chainlink/cre-cli
# либо через bun:
bun add @chainlink/cre-sdk
```

Аккаунт создаётся на `app.chain.link/cre/discover`. Симуляция workflow доступна сразу, без подтверждения доступа; **развёртывание на DON** пока в статусе Early Access.

### Пример: Cron-триггер + HTTP-запрос + запись в контракт

Аналог того, что раньше делал Chainlink Functions — получить внешние данные и записать их on-chain:

```ts
import { cre } from "@chainlink/cre-sdk";
import { encodeAbiParameters } from "viem";

export async function main() {
    // Триггер: запуск каждые 5 минут по cron-расписанию
    const trigger = cre.capabilities.cron.trigger({ schedule: "*/5 * * * *" });

    cre.handler(trigger, async (runtime) => {
        // Секреты не хардкодятся в коде — запрашиваются через runtime
        const apiKey = await runtime.getSecret({ id: "API_KEY" }).result();

        // HTTP-запрос выполняется независимо каждой нодой DON,
        // а затем результаты сверяются между собой (консенсус)
        const response = await runtime.http
            .sendRequest({
                url: "https://api.example.com/price",
                headers: { Authorization: `Bearer ${apiKey}` },
            })
            .result();

        const price = BigInt(Math.round(response.json().price * 100));

        // Кодируем значение для передачи в контракт
        const payload = encodeAbiParameters([{ type: "uint256" }], [price]);

        // report() формирует подписанный сетью DON пакет данных,
        // который любой контракт, реализующий IReceiver, может принять
        const report = await runtime.report(payload).result();

        return report;
    });
}
```

Контракт-получатель реализует интерфейс `IReceiver` — по аналогии с тем, как раньше `fulfillRequest()` принимал ответ от Functions.

### Пример: EVM Log-триггер (аналог Automation по событию)

```ts
import { cre } from "@chainlink/cre-sdk";

export async function main() {
    const trigger = cre.capabilities.evmLog.trigger({
        chainSelector: "ethereum-testnet-sepolia",
        address: "0xВашКонтракт",
        eventSignature: "EntryAdded(address,string,uint256)",
    });

    cre.handler(trigger, async (runtime, payload) => {
        // payload содержит данные события, включая параметры
        runtime.log(`Обнаружено событие в транзакции ${payload.transactionHash}`);

        // Дальше — любая бизнес-логика: HTTP-запрос, запись в другой контракт
    });
}
```

### Локальная симуляция

```bash
cre workflow simulate --target staging-settings --config config.staging.json main.ts

# Для симуляций, которые пишут в блокчейн, нужен флаг --broadcast
cre workflow simulate --broadcast --config config.staging.json main.ts
```

Симуляция компилирует workflow в WASM и исполняет **с реальными вызовами** к живым API и публичным EVM-сетям — то есть это не заглушка, а полноценная проверка перед выкладкой на DON.

### Ключевые моменты

1. **CRE — надмножество, а не аналог.** Всё, что делал Functions, делает и CRE, плюс многошаговые сценарии, которые раньше требовали связки Functions + Automation + собственный релеер, теперь укладываются в один workflow.
2. **Триггер определяет тип задачи, а не отдельный продукт.** Cron — то, что раньше называлось Automation (time-based), EVM Log — то, что раньше называлось Automation (log trigger) или совмещалось с Functions, HTTP — новый тип, которого не было ни у одного из старых продуктов.
3. **Секреты через `runtime.getSecret`, никогда не в коде.** Workflow компилируется в WASM и разворачивается в сети — секрет, зашитый в код, был бы виден при распространении.
4. **Развёртывание — Early Access.** Для учебного проекта на чемпионате разумно ограничиться локальной симуляцией (`cre workflow simulate`), которая уже показывает всю логику работы без необходимости в полном доступе к продакшен-сети.

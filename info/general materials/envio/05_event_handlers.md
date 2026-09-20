# Обработчики событий (src/handlers)

## Что такое обработчик

**Обработчик** — функция, которая получает данные события из блокчейна, обрабатывает их и записывает в базу. Это единственное место, где вы пишете бизнес-логику индексатора.

По умолчанию обработчики ищутся в файлах папки `src/handlers`.

## Регистрация

```ts
import { indexer } from "envio";

indexer.onEvent(
    { contract: "ИМЯ_КОНТРАКТА", event: "ИМЯ_СОБЫТИЯ" },
    async ({ event, context }) => {
        // логика здесь
    },
);
```

Имена контракта и события берутся из `config.yaml`. Модуль `envio` экспортирует объект `indexer` и типы, сгенерированные из вашего конфига и схемы — после правки этих файлов запускайте `pnpm codegen`.

## Простой пример

Событие `NewGreeting(address user, string greeting)` обновляет сущность `User`:

```ts
import { indexer, type User } from "envio";

indexer.onEvent(
    { contract: "Greeter", event: "NewGreeting" },
    async ({ event, context }) => {
        const userId = event.params.user;
        const latestGreeting = event.params.greeting;

        // get вернёт сущность или undefined, если её ещё нет
        const currentUser = await context.User.get(userId);

        const userEntity: User = currentUser
            ? {
                  id: userId,
                  latestGreeting,
                  numberOfGreetings: currentUser.numberOfGreetings + 1,
                  greetings: [...currentUser.greetings, latestGreeting],
              }
            : {
                  id: userId,
                  latestGreeting,
                  numberOfGreetings: 1,
                  greetings: [latestGreeting],
              };

        // set создаёт или перезаписывает сущность
        context.User.set(userEntity);
    },
);
```

## Объект event

Параметры события доступны через `event.params`:

```ts
const sender = event.params.from;
const amount = event.params.value;
```

Дополнительные поля:

| Поле | Содержимое |
|---|---|
| `event.chainId` | id сети, в которой произошло событие |
| `event.srcAddress` | адрес контракта, эмитировавшего событие |
| `event.logIndex` | индекс лога внутри блока |
| `event.block` | поля блока (по умолчанию `number`, `timestamp`, `hash`) |
| `event.transaction` | поля транзакции (по умолчанию пусто) |

Все адреса по умолчанию приводятся к формату EIP-55 (checksum, смешанный регистр). Если нужен нижний регистр для всех адресов сразу, в конфиге задаётся `address_format: lowercase`.

### Запрос дополнительных полей

Поля транзакции по умолчанию не загружаются. Нужные перечисляются в опции `fields` при регистрации:

```ts
indexer.onEvent(
    {
        contract: "MyContract",
        event: "Transfer",
        fields: { transaction: ["hash", "from"], block: ["timestamp"] },
    },
    async ({ event, context }) => {
        event.transaction.hash;     // string
        event.block.timestamp;      // number
        event.transaction.gasUsed;  // ошибка типизации: поле не запрошено
    },
);
```

Обращение к незапрошенному полю — ошибка компиляции, поэтому список полей не разъезжается с кодом. `event.block.number` доступен всегда, его перечислять не нужно.

## Объект context — работа с БД

### Чтение

```ts
// Вернёт сущность или undefined
const user = await context.User.get(userId);

// Бросит ошибку, если сущности нет — когда её отсутствие означает баг
const pool = await context.Pool.getOrThrow(poolId);
const pool = await context.Pool.getOrThrow(poolId, `Пул ${poolId} должен существовать`);

// Вернёт существующую или создаст новую со значениями по умолчанию
const account = await context.Account.getOrCreate({
    id: userId,
    balance: 0n,
});
```

### Поиск по полю

```ts
const approvals = await context.Approval.getWhere({
    owner_id: { _eq: event.params.owner },
});

// Несколько условий объединяются по И
const accounts = await context.Account.getWhere({
    id: { _eq: event.params.account },
    balance: { _gte: 1_000_000n, _lte: 10_000_000n },
});
```

Кроме `_eq` доступны `_gt`, `_gte`, `_lt`, `_lte`, `_in`.

Ставьте вызов `getWhere` ближе к началу обработчика — так он попадёт в фазу предзагрузки (см. ниже). Очень большие выборки могут переполнить память.

### Запись

```ts
context.Entity.set({
    id: entityId,
    // остальные поля
});
```

`set` и `deleteUnsafe` работают через хранилище в памяти и **не требуют `await`** — в отличие от `get`.

### Обновление отдельных полей

Частичного обновления нет: сущность читается целиком, изменяется и записывается обратно.

```ts
const pool = await context.Pool.get(poolId);

if (pool) {
    context.Pool.set({
        ...pool,                                  // копируем все поля
        totalValueLocked: pool.totalValueLocked + newDeposit,
    });
}
```

### Удаление

```ts
context.Entity.deleteUnsafe(entityId);
```

Метод помечен как экспериментальный и небезопасный: ссылки на удалённую сущность придётся чинить вручную, иначе база станет несогласованной.

### Логирование

```ts
context.log.info("Обработан перевод", { amount: event.params.value });
```

В отличие от `console.log`, эти сообщения видны в логах Envio Cloud.

## Предзагрузка: обработчики выполняются дважды

Это главная особенность, о которую спотыкаются новички.

В HyperIndex V3 **всегда включена оптимизация предзагрузки** (preload optimization), отключить её нельзя. Смысл в следующем: индексатор прогоняет обработчики первый раз, чтобы понять, какие сущности им понадобятся, загружает их все одним пакетным запросом к БД, и только потом выполняет обработчики повторно — уже с готовыми данными.

Благодаря этому вместо тысяч мелких запросов к базе выполняется несколько больших.

**Практическое следствие:** любой код в обработчике выполняется два раза. Для чтения и записи сущностей это безопасно. Но если внутри есть тяжёлые вычисления или побочные эффекты, их нужно пропускать в фазе предзагрузки:

```ts
indexer.onEvent(
    { contract: "ERC20", event: "Transfer" },
    async ({ event, context }) => {
        // Загрузка данных — выполняется в обе фазы, это нормально
        const [sender, receiver] = await Promise.all([
            context.Account.getOrThrow(event.params.from),
            context.Account.getOrThrow(event.params.to),
        ]);

        // Пропускаем дорогие операции в фазе предзагрузки
        if (context.isPreload) {
            return;
        }

        const result = performExpensiveOperation(event.params.value);

        context.Account.set({
            id: event.params.from,
            balance: sender.balance - event.params.value,
            computedValue: result,
        });
    },
);
```

## Внешние вызовы: Effect API

Индексатор работает в Node.js, поэтому в обработчике доступны `fetch`, `viem` и любые другие библиотеки. Но из-за двойного выполнения наивный `fetch` отправит запрос дважды.

Правильный способ — **Effect API**: он автоматически батчит вызовы, запоминает результаты и умеет ограничивать частоту запросов.

```ts
import { indexer, createEffect, S } from "envio";

const getMetadata = createEffect(
    {
        name: "getMetadata",
        input: S.string,
        output: {
            description: S.string,
            value: S.bigint,
        },
        rateLimit: { calls: 5, per: "second" },  // ограничение частоты
        cache: true,                              // сохранять результаты в БД
    },
    async ({ input }) => {
        const response = await fetch(`https://api.example.com/metadata/${input}`);
        const data = await response.json();
        return { description: data.description, value: data.value };
    },
);

indexer.onEvent(
    { contract: "ERC20", event: "Transfer" },
    async ({ event, context }) => {
        // Вызов выполнится параллельно для всех событий пачки
        // и автоматически мемоизируется — дубликатов не будет
        const metadata = await context.effect(getMetadata, event.params.from);
    },
);
```

## Несколько обработчиков на одно событие

Начиная с v3.4 на одно событие можно зарегистрировать сколько угодно обработчиков, каждый со своим фильтром `where`. Это позволяет не городить ветвление внутри одного обработчика:

```ts
const ZERO_ADDRESS = "0x0000000000000000000000000000000000000000";

// Общий учёт всех переводов
indexer.onEvent(
    { contract: "ERC20", event: "Transfer", wildcard: true },
    async ({ event, context }) => {
        // ...
    },
);

// Отдельная логика только для эмиссии (перевод с нулевого адреса)
indexer.onEvent(
    {
        contract: "ERC20",
        event: "Transfer",
        wildcard: true,
        where: () => ({ params: [{ from: ZERO_ADDRESS }] }),
    },
    async ({ event, context }) => {
        // ...
    },
);
```

Событие, подходящее под несколько регистраций, попадёт в каждую из них — в порядке регистрации.

## Доступ к конфигурации из обработчика

```ts
import { indexer } from "envio";

indexer.onEvent(
    { contract: "Greeter", event: "NewGreeting" },
    async ({ event, context }) => {
        const chain = indexer.chains[event.chainId];

        chain.id;                 // id сети
        chain.startBlock;         // стартовый блок из конфига
        chain.isRealtime;         // достигнута ли голова цепи
        chain.Greeter.addresses;  // адреса контракта, включая динамические
    },
);
```

## Продвинутые возможности

Кратко, для ориентира — подробности в документации:

- **Dynamic Contract Registration** — индексация контрактов, которые создаются фабрикой во время работы (пулы Uniswap, например)
- **Wildcard Indexing** — индексация всех контрактов определённого типа без перечисления адресов (все переводы ERC-20 в сети)
- **Topic Filtering** — отсев ненужных событий на уровне источника данных
- **Contract State** — прямой вызов `view`-функций контракта из обработчика
- **Block Handlers** (`indexer.onBlock`) — обработка блоков без привязки к контрактам

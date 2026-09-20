# config.yaml и schema.graphql

## config.yaml — что индексировать

Файл описывает контракты, их события и сети, на которых эти контракты развёрнуты.

### Базовый пример

```yaml
# Строка ниже подключает автодополнение и проверку в IDE
# yaml-language-server: $schema=./node_modules/envio/evm.schema.json
name: erc20-indexer
description: Индексатор переводов ERC-20

# 1. Контракты описываются один раз: имя + список событий
contracts:
  - name: ERC20
    events:
      - event: "Approval(address indexed owner, address indexed spender, uint256 value)"
      - event: "Transfer(address indexed from, address indexed to, uint256 value)"

# 2. Сети ссылаются на контракты по имени и задают адрес
chains:
  - id: 1                    # Ethereum Mainnet
    start_block: 0
    contracts:
      - name: ERC20
        address: "0x1f9840a85d5aF5bf1D1762F925BDADdC4201F984"  # UNI

  - id: 100                  # Gnosis
    start_block: 0
    contracts:
      - name: ERC20
        address: "0x4537e328Bf7e4eFA29D05CAeA260D7fE26af9D74"
```

Обратите внимание на структуру: **контракт описывается один раз**, а сети только подставляют свои адреса. Так работает мультичейн-индексация — один и тот же обработчик обслуживает все сети сразу.

После любого изменения конфига или схемы нужно перегенерировать типы:

```bash
pnpm codegen
```

### Объявление событий

Рекомендуемый способ — человекочитаемая сигнатура прямо в конфиге. Индексируются **только перечисленные** события; чтобы перестать индексировать событие, достаточно убрать строку.

```yaml
contracts:
  - name: Greeter
    events:
      - event: "NewGreeting(address user, string greeting)"
      - event: "ClearGreeting(address user)"
```

**Через файл ABI** — если ABI уже есть и нужно выбрать из него часть событий:

```yaml
contracts:
  - name: Greeter
    abi_file_path: ./abis/greeter.json
    events:
      - event: NewGreeting   # сигнатура берётся из ABI
```

**Собственное имя события** — нужно, когда два события называются одинаково, но имеют разные сигнатуры:

```yaml
events:
  - event: Assigned(address indexed recipientId, uint256 amount, address token)
  - event: Assigned(address indexed recipientId, uint256 amount, address token, address sender)
    name: AssignedWithSender   # чтобы различать их в обработчиках
```

### Стартовый и конечный блок

```yaml
chains:
  - id: 1
    start_block: 0      # для HyperSync безопасное значение: он сам перемотает
                        # к первому блоку, где есть данные вашего контракта
    end_block: 19000000 # необязательно: остановиться на этом блоке
```

Можно задать стартовый блок **отдельно для каждого контракта** — удобно, когда контракты задеплоены в разное время:

```yaml
chains:
  - id: 1
    start_block: 18000000        # значение по умолчанию для сети
    contracts:
      - name: ERC20
        address: "0x1111..."
        start_block: 18500000    # переопределение для этого контракта
      - name: Greeter
        address: "0x9D02..."     # использует 18000000
```

### Адреса

```yaml
# Один адрес
contracts:
  - name: MyContract
    address: "0xКонтракт"

# Несколько адресов одного контракта
contracts:
  - name: MyContract
    address:
      - "0xАдрес1"
      - "0xАдрес2"
```

Адреса принимаются и в checksum-формате, и в нижнем регистре — Envio нормализует их сам.

### RPC

Для сетей, поддерживаемых HyperSync, указывать RPC **не нужно** — HyperSync используется как основной источник данных. RPC можно добавить как запасной вариант. Для сетей без поддержки HyperSync RPC обязателен.

```yaml
chains:
  - id: 1
    rpc:
      - url: https://eth-mainnet.your-provider.com
        for: sync        # историческая синхронизация
      - url: wss://eth-mainnet.your-provider.com
        for: realtime    # отслеживание головы цепи
      - url: https://fallback.example.com
        for: fallback    # запасной источник
```

### Выбор полей блока и транзакции

По умолчанию в событии доступен только базовый набор полей (`block.number`, `block.timestamp`, `block.hash`). Остальное запрашивается явно — это экономит трафик и ускоряет индексацию:

```yaml
events:
  - event: "Assigned(address indexed user, uint256 amount)"
    field_selection:
      transaction_fields:
        - hash
        - gasUsed
      block_fields:
        - timestamp
```

### Переменные окружения

Подстановка работает в любом месте конфига — удобно, чтобы не коммитить адреса и ключи:

```yaml
chains:
  - id: ${ENVIO_CHAIN_ID:-1}          # значение по умолчанию после :-
    contracts:
      - name: Greeter
        address: "${ENVIO_GREETER_ADDRESS}"
```

```bash
ENVIO_GREETER_ADDRESS=0xВашАдрес pnpm dev
```

### Сырые события для отладки

По умолчанию исходные события в БД не сохраняются — ради скорости. Для отладки это можно включить:

```yaml
raw_events: true   # события попадут в таблицу raw_events, видную в Hasura
```

---

## schema.graphql — что получится на выходе

Схема описывает **сущности** — таблицы, которые появятся в базе и станут доступны через GraphQL.

### Базовый пример

```graphql
type User {
    id: ID!                    # обязательное поле, первичный ключ
    address: String!
    totalTransfers: Int!
    balance: BigInt!
    lastSeenAt: BigInt!
}

type Transfer {
    id: ID!
    from: String!
    to: String!
    value: BigInt!
    blockNumber: BigInt!
    timestamp: BigInt!
}
```

Поле `id` обязательно у каждой сущности: по нему запись читается и перезаписывается. Восклицательный знак означает «поле не может быть null».

### Основные типы

| Тип | Назначение |
|---|---|
| `ID!` | идентификатор сущности |
| `String` | строки, адреса, хеши |
| `Int` | небольшие целые |
| `BigInt` | `uint256` и другие большие числа |
| `Float` | дробные |
| `Boolean` | флаги |
| `Bytes` | байтовые данные |
| `[String!]!` | массив строк |

Для значений `uint256` используйте `BigInt` — в `Int` они не помещаются.

### Связи между сущностями

```graphql
type Task {
    id: ID!
    title: String!
    points: BigInt!
    author: User!          # связь с сущностью User
}

type User {
    id: ID!
    address: String!
    tasks: [Task!]!        # обратная связь
}
```

В обработчике связь задаётся через поле `<имя>_id`, куда кладётся идентификатор связанной сущности — не сам объект:

```ts
context.Task.set({
    id: taskId,
    title: "Реестр студентов",
    points: 10n,
    author_id: userAddress,   // сохраняем ID, а не объект User
});
```

Envio сам подставит объект `author` при запросе через GraphQL.

### Хорошие практики выбора id

Идентификатор должен быть уникальным и детерминированным — то есть вычисляться из самого события, а не через счётчик:

```ts
// Для события: связка хеша транзакции и индекса лога уникальна глобально
const id = `${event.transaction.hash}-${event.logIndex}`;

// Для сущности-агрегата: естественный ключ
const id = event.params.user;                          // адрес пользователя
const id = `${event.chainId}-${event.params.poolId}`;  // сеть + id пула
```

Второй вариант с указанием `chainId` важен для мультичейн-индексаторов: без него записи из разных сетей затрут друг друга.

### GraphQL-запрос к результату

После синхронизации данные доступны через Hasura:

```graphql
query TopUsers {
    User(order_by: { totalTransfers: desc }, limit: 10) {
        id
        address
        totalTransfers
        balance
    }
}
```

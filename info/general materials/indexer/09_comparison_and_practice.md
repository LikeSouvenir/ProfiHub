# Итоговое сравнение и практика

## Единая сводная таблица по всем 8 инструментам раздела
| Инструмент | Язык обработчиков | Хостинг | Скорость backfill | Сильная сторона |
|---|---|---|---|---|
| The Graph | AssemblyScript | Decentralized / hosted | Средняя | Стандарт индустрии, готовые субграфы |
| Goldsky | AssemblyScript (совместим с The Graph) | Managed | Высокая | Managed-хостинг + Mirror (сырые данные в Postgres) |
| Envio | TypeScript | Self-host (Docker) / hosted | Самая высокая (по бенчмаркам) | Скорость + обычный TS |
| Ponder | TypeScript | Self-host | Высокая | Максимально TS-native DX, свой сервер |
| Subsquid (SQD) | TypeScript | Self-host / hosted | Высокая | Батчевый ETL + тяжёлая SQL-агрегация |
| SubQuery | TypeScript/AssemblyScript | Self-host / managed | Средняя | Больше всего сетей, включая не-EVM |
| Sentio | TypeScript | Managed SaaS | — | Профилирование газа + дашборды/алерты |
| Ormi | Часто без кода | Managed SaaS | — | Готовые API под типовые задачи |
| Chainbase | SQL | Managed SaaS | — | SQL к готовым датасетам + ИИ-запросы |

## Практика: индексация события Transfer одного и того же ERC20-токена — сравнение подходов

### Задача
Проиндексировать событие `Transfer(address indexed from, address indexed to, uint256 value)` конкретного ERC20-токена, сохранить каждый перевод и агрегированный баланс по адресам, получить возможность запросить последние 5 переводов.

### Реализация 1: The Graph (AssemblyScript, WASM-песочница)
```typescript
export function handleTransfer(event: TransferEvent): void {
    let transfer = new Transfer(event.transaction.hash.concatI32(event.logIndex.toI32()))
    transfer.from = event.params.from
    transfer.to = event.params.to
    transfer.value = event.params.value
    transfer.save()
}
```
**Особенность:** компилируется в WebAssembly, исполняется в изолированной песочнице Graph Node — нельзя подключить произвольные npm-пакеты.

### Реализация 2: Envio (обычный TypeScript, async/await)
```typescript
ERC20Token.Transfer.handler(async ({ event, context }) => {
    context.Transfer.set({
        id: `${event.transaction.hash}-${event.logIndex}`,
        from: event.params.from,
        to: event.params.to,
        value: event.params.value,
    });
});
```
**Особенность:** обычная асинхронная функция, никакой компиляции в WASM, доступ к HyperSync для быстрого чтения истории.

### Реализация 3: Ponder (TypeScript + Drizzle-подобный ORM)
```typescript
ponder.on("ERC20Token:Transfer", async ({ event, context }) => {
    await context.db.insert(transfer).values({
        id: `${event.transaction.hash}-${event.log.logIndex}`,
        from: event.args.from,
        to: event.args.to,
        value: event.args.value,
    });
});
```
**Особенность:** схема описывается в TS-файле (не в отдельном GraphQL SDL), можно писать и произвольные REST-эндпоинты поверх той же базы.

### Реализация 4: Subsquid (батчевая обработка)
```typescript
processor.run(new TypeormDatabase(), async (ctx) => {
    const transfers: Transfer[] = [];
    for (const block of ctx.blocks) {
        for (const log of block.logs) {
            const { from, to, value } = decodeTransferEvent(log);
            transfers.push(new Transfer({ id: `${log.transactionHash}-${log.logIndex}`, from, to, value }));
        }
    }
    await ctx.store.insert(transfers); // один массовый insert на весь батч
});
```
**Особенность:** обрабатывается сразу пачка блоков за раз, а не одно событие — эффективнее для тяжёлой агрегации на больших объёмах.

### Реализация 5: SubQuery (TypeScript, универсальный SDK)
```typescript
export async function handleTransfer(log: TransferLog): Promise<void> {
    const transfer = Transfer.create({
        id: `${log.transactionHash}-${log.logIndex}`,
        from: log.args.from,
        to: log.args.to,
        value: log.args.value.toBigInt(),
    });
    await transfer.save();
}
```
**Особенность:** тот же самый паттерн `Entity.create()/.save()` можно применить и к событиям Substrate/Cosmos, просто заменив тип входного параметра.

### Реализация 6: Chainbase (вообще без индексатора — сразу SQL)
```sql
SELECT * FROM ethereum.erc20_transfers
WHERE contract_address = '0xАдресКонтракта'
ORDER BY block_timestamp DESC
LIMIT 5;
```
**Особенность:** если токен достаточно популярен и уже есть в готовых датасетах Chainbase — писать индексатор не нужно вообще.

## Итог практики: что должно быть видно после сравнения
1. **Логика везде одна и та же**: слушаем событие `Transfer`, извлекаем `from/to/value`, сохраняем запись и обновляем агрегаты.
2. **Разница — в архитектуре исполнения**: WASM-песочница (The Graph/Goldsky) vs обычный Node.js-процесс (Envio/Ponder) vs батчевая ETL-модель (Subsquid) vs полностью готовые датасеты без написания кода (Chainbase/Ormi).
3. **Выбор инструмента — это выбор компромисса**, а не поиск "единственно правильного" варианта — итоговое решение зависит от требований к скорости, языку команды, необходимости в self-host контроле и от того, насколько типовая задача стоит перед проектом.

## Финальный чек-лист выбора индексатора для нового проекта
- [ ] Сеть EVM-совместима? Если нет — в первую очередь смотреть **SubQuery**.
- [ ] Нужна максимальная скорость первичной синхронизации большого объёма истории? → **Envio**.
- [ ] Команда предпочитает чистый TypeScript без изучения нового языка? → **Envio** или **Ponder**.
- [ ] Нужен готовый, широко распространённый стандарт с множеством готовых субграфов? → **The Graph**, при необходимости managed-хостинга → **Goldsky**.
- [ ] Нужна тяжёлая SQL-аналитика и агрегация больших объёмов? → **Subsquid**.
- [ ] Задача типовая (балансы/холдеры/история) и не хочется писать индексатор вообще? → **Ormi** или **Chainbase**.
- [ ] Нужен встроенный мониторинг газа и алерты в реальном времени? → **Sentio**.

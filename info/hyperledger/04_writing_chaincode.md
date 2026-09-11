# Написание смарт-контракта (chaincode) на Node.js

## Что такое chaincode
**Chaincode** — так в Hyperledger Fabric называется смарт-контракт: код, который определяет, какие транзакции допустимы и как они изменяют состояние реестра. Chaincode может быть написан на Go, Java или Node.js — рассматриваем вариант на Node.js как наиболее близкий для JS-разработчиков.

## Подготовка проекта chaincode
```bash
mkdir -p fabric-samples/my-project/chaincode-javascript/lib
cd fabric-samples/my-project/chaincode-javascript
npm init -y
npm install fabric-contract-api fabric-shim
```
- **fabric-contract-api** — высокоуровневый API: базовый класс `Contract`, от которого наследуются собственные контракты.
- **fabric-shim** — низкоуровневый интерфейс chaincode, обеспечивающий связь между кодом на Node.js и peer'ом Fabric (используется библиотекой `fabric-contract-api` "под капотом", но должен быть установлен явно).

В `package.json` обязательно добавляется скрипт запуска, без которого chaincode не сможет развернуться в сети:
```json
{
  "scripts": {
    "start": "fabric-chaincode-node start"
  }
}
```

## Базовая структура контракта
```js
// lib/cars.js
'use strict';

const { Contract } = require('fabric-contract-api');

class Cars extends Contract {
    constructor() {
        // Имя контракта. Если не передать — по умолчанию используется имя класса
        super('MyCars');
    }

    // Выполняется один раз при инициализации chaincode — заполняет реестр начальными данными
    async InitLedger(ctx) {
        const cars = [
            { ID: 'car1', Color: 'red', Brand: 'BMW', Owner: 'admin' },
            { ID: 'car2', Color: 'black', Brand: 'Mercedes', Owner: 'admin' },
        ];

        for (const car of cars) {
            car.docType = 'car';
            await ctx.stub.putState(car.ID, Buffer.from(JSON.stringify(car)));
        }
    }
}

module.exports = Cars;
```

## Ключевые правила написания контрактов Fabric

1. **Контракт обязан наследоваться от `Contract`** из `fabric-contract-api`.
2. **Первый параметр каждого метода — `ctx` (Context).** Через него доступны:
   - `ctx.stub` — API для работы с реестром (`putState`, `getState`, `deleteState`, `getStateByRange` и другие).
   - `ctx.clientIdentity` — данные о том, кто вызвал транзакцию (сертификат, MSP ID организации).
3. **Все методы должны быть асинхронными (`async`)** — взаимодействие с реестром идёт через промисы.
4. **Данные в реестре хранятся как массивы байт**, поэтому перед записью объект превращается в строку (`JSON.stringify`) и оборачивается в `Buffer`, а при чтении — обратный процесс (`.toString()` + `JSON.parse`).
5. **Класс контракта должен экспортироваться** (`module.exports = Cars`), чтобы быть доступным для регистрации в `index.js`.

## Полный пример: CRUD-операции над сущностью "машина"
```js
// lib/cars.js (продолжение класса Cars)

async AddCar(ctx, id, color, brand, owner) {
    const exists = await this.CarExists(ctx, id);
    if (exists) {
        throw new Error(`Машина ${id} уже существует`);
    }

    const car = { ID: id, Color: color, Brand: brand, Owner: owner, docType: 'car' };
    await ctx.stub.putState(id, Buffer.from(JSON.stringify(car)));
    return JSON.stringify(car);
}

async GetCar(ctx, id) {
    const carJSON = await ctx.stub.getState(id);
    if (!carJSON || carJSON.length === 0) {
        throw new Error(`Машины ${id} не существует`);
    }
    return carJSON.toString();
}

async TransferCar(ctx, id, newOwner) {
    const carJSON = await this.GetCar(ctx, id);
    const car = JSON.parse(carJSON);

    const oldOwner = car.Owner;
    car.Owner = newOwner;

    await ctx.stub.putState(id, Buffer.from(JSON.stringify(car)));
    return oldOwner;
}

async DeleteCar(ctx, id) {
    const exists = await this.CarExists(ctx, id);
    if (!exists) {
        throw new Error(`Машины ${id} не существует`);
    }
    return await ctx.stub.deleteState(id);
}

async CarExists(ctx, id) {
    const carJSON = await ctx.stub.getState(id);
    return carJSON && carJSON.length > 0;
}

// Перебор ВСЕХ записей в реестре через диапазонный запрос ('' , '' = весь диапазон ключей)
async GetAllCars(ctx) {
    const allResults = [];
    const iterator = await ctx.stub.getStateByRange('', '');

    let result = await iterator.next();
    while (!result.done) {
        const strValue = Buffer.from(result.value.value.toString()).toString('utf8');

        try {
            const record = JSON.parse(strValue);
            if (record.docType === 'car') {
                allResults.push(record);
            }
        } catch (err) {
            console.log(err);
        }

        result = await iterator.next();
    }

    return JSON.stringify(allResults);
}
```
*Поле `docType` — распространённая практика Fabric: так как в одном канале и одной базе состояния могут храниться разные типы сущностей (машины, пользователи, заказы), `docType` позволяет отличать их друг от друга при переборе всех записей через `getStateByRange`.*

## index.js — точка входа chaincode
```js
// index.js
'use strict';

const cars = require('./lib/cars');

module.exports.Cars = cars;
module.exports.contracts = [cars];
```
Массив `contracts` перечисляет все контракты, которые нужно зарегистрировать в этом chaincode — при добавлении нового контракта (например, токена, см. отдельный конспект) достаточно импортировать его класс и добавить в этот массив.

## Оптимизация структуры контрактов
Вместо одного огромного класса с десятками методов, при росте проекта стоит:
- Разносить сущности по отдельным файлам в `lib/` (`cars.js`, `users.js`, `orders.js`), каждая — отдельный класс-контракт.
- Выносить повторяющуюся логику (например, проверку существования записи, сериализацию) во вспомогательные функции/утилитный модуль, переиспользуемый разными контрактами, вместо копирования одного и того же метода `Exists(...)` в каждый класс.
- Для сущностей с большим количеством полей — использовать отдельные функции валидации входных данных перед `putState`, чтобы не дублировать проверки в каждом методе.

# 08. Написание смарт-контракта на JavaScript

## 8.1. Подготовка проекта
```bash
mkdir -p ~/projects/hl-project/my-project/chaincode-javascript/lib
cd ~/projects/hl-project/my-project/chaincode-javascript

npm init -y
npm install fabric-contract-api fabric-shim
```
В `package.json` добавьте (без этого peer не сможет запустить chaincode):
```json
{
  "scripts": {
    "start": "fabric-chaincode-node start"
  }
}
```

## 8.2. Правила, обязательные для любого контракта на JS
1. Контракт наследуется от `Contract` из `fabric-contract-api`.
2. Первый параметр каждого метода — `ctx` (Context).
3. Все методы — `async`.
4. Данные хранятся как байты: `Buffer.from(JSON.stringify(obj))` при записи, `.toString()` + `JSON.parse` при чтении.
5. Класс обязательно экспортируется.

## 8.3. Готовое полное решение: контракт Cars
```js
// lib/cars.js
'use strict';

const { Contract } = require('fabric-contract-api');

class Cars extends Contract {
    constructor() {
        super('MyCars');
    }

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

    async UpdateCar(ctx, id, color, brand, owner) {
        const exists = await this.CarExists(ctx, id);
        if (!exists) {
            throw new Error(`Машины ${id} не существует`);
        }

        const car = { ID: id, Color: color, Brand: brand, Owner: owner, docType: 'car' };
        await ctx.stub.putState(id, Buffer.from(JSON.stringify(car)));
        return JSON.stringify(car);
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

    // Требует базу состояния CouchDB (см. конспект 04) — поиск по конкретному полю
    async GetCarsByColor(ctx, color) {
        const query = { selector: { docType: 'car', Color: color } };
        const iterator = await ctx.stub.getQueryResult(JSON.stringify(query));

        const results = [];
        let result = await iterator.next();
        while (!result.done) {
            const strValue = Buffer.from(result.value.value.toString()).toString('utf8');
            results.push(JSON.parse(strValue));
            result = await iterator.next();
        }
        return JSON.stringify(results);
    }

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

    // История изменений конкретной машины — все версии значения по этому ключу
    async GetCarHistory(ctx, id) {
        const iterator = await ctx.stub.getHistoryForKey(id);
        const history = [];

        let result = await iterator.next();
        while (!result.done) {
            const record = {
                txId: result.value.txId,
                timestamp: result.value.timestamp,
                isDelete: result.value.isDelete,
                value: result.value.value.length > 0
                    ? JSON.parse(result.value.value.toString('utf8'))
                    : null,
            };
            history.push(record);
            result = await iterator.next();
        }

        return JSON.stringify(history);
    }
}

module.exports = Cars;
```

## 8.4. index.js — регистрация контракта(ов)
```js
// index.js
'use strict';

const cars = require('./lib/cars');

module.exports.Cars = cars;
module.exports.contracts = [cars];
```

## 8.5. Проверка прав доступа внутри методов (MSP + атрибуты)
```js
async DeleteCar(ctx, id) {
    const clientMSPID = ctx.clientIdentity.getMSPID();
    if (clientMSPID === 'Org3MSP') {
        throw new Error('Организации Org3 запрещено удалять машины');
    }

    const isManager = ctx.clientIdentity.assertAttributeValue('role', 'manager');
    if (!isManager) {
        throw new Error('Удалять машины может только пользователь с ролью manager');
    }

    const exists = await this.CarExists(ctx, id);
    if (!exists) {
        throw new Error(`Машины ${id} не существует`);
    }
    return await ctx.stub.deleteState(id);
}
```

## 8.6. Готовые тестовые вызовы через peer CLI (до написания API)
Перед тем как писать полноценное клиентское приложение, удобно проверить chaincode напрямую консольными командами (после деплоя, см. конспект 11):
```bash
peer chaincode query -C blockchain2024 -n Cars -c '{"Args":["GetAllCars"]}'

peer chaincode invoke -o localhost:7050 --tls --cafile "$ORDERER_CA" \
  -C blockchain2024 -n Cars \
  --peerAddresses localhost:7051 --tlsRootCertFiles "$PEER0_ORG1_CA" \
  --peerAddresses localhost:9051 --tlsRootCertFiles "$PEER0_ORG2_CA" \
  -c '{"Args":["AddCar","car3","white","Toyota","admin"]}'
```

## 8.7. Частые ошибки новичков при написании chaincode на JS
| Ошибка | Симптом | Решение |
|---|---|---|
| Забыт `async` перед методом | `TypeError: contract.MethodName is not a function` или зависание | Все методы контракта обязаны быть `async` |
| Запись объекта напрямую, без `JSON.stringify` | Ошибка сериализации или `[object Object]` в реестре | Всегда `Buffer.from(JSON.stringify(obj))` |
| Не проверяется существование перед `AddCar` | Тихая перезапись существующей записи | Всегда вызывать `Exists`-проверку перед созданием |
| `docType` не указан | `GetAllCars` возвращает вперемешку разные сущности при их накоплении в реестре | Всегда добавлять `docType` и фильтровать по нему |
| Chaincode "зависает" при `getStateByRange` на большом объёме данных | Медленный ответ, таймаут клиента | Для больших наборов данных использовать `getQueryResult` с пагинацией (`getQueryResultWithPagination`) вместо полного перебора |

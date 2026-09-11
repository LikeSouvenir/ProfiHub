# Практика: полный цикл — от сети до рабочего API (оптимизированная версия)

Собираем весь материал темы в единый, воспроизводимый сценарий: поднимаем сеть, пишем и разворачиваем chaincode, регистрируем пользователей и поднимаем оптимизированный API — с одним стартовым скриптом вместо набора разрозненных ручных команд.

## Структура проекта
```
hl-project/
├── fabric-samples/            # bin, builders, config, test-network (из install-fabric.sh)
├── my-project/
│   ├── chaincode-javascript/
│   │   ├── lib/cars.js
│   │   ├── index.js
│   │   └── package.json
│   └── application-javascript/
│       ├── .env
│       ├── express.js
│       └── package.json
├── src/                        # фронтенд (например, React из темы про Vite)
├── Start.sh
└── .gitignore
```

## Шаг 1. Конфигурация в одном месте — .env
```
# my-project/application-javascript/.env
CHANNEL_NAME=blockchain2024
CHAINCODE_NAME=Cars
CONTRACT_NAME=MyCars
ORG1_MSP_ID=Org1MSP
ORG2_MSP_ID=Org2MSP
ADMIN_USER_ID=admin
ADMIN_USER_PASSWD=adminpw
API_PORT=7000
```
```bash
npm install dotenv
```

## Шаг 2. Chaincode (напоминание из конспекта про написание контракта)
```js
// my-project/chaincode-javascript/lib/cars.js
'use strict';
const { Contract } = require('fabric-contract-api');

class Cars extends Contract {
    constructor() {
        super('MyCars');
    }

    async InitLedger(ctx) {
        const cars = [
            { ID: 'car1', Color: 'red', Brand: 'BMW', Owner: 'admin', docType: 'car' },
        ];
        for (const car of cars) {
            await ctx.stub.putState(car.ID, Buffer.from(JSON.stringify(car)));
        }
    }

    async AddCar(ctx, id, color, brand, owner) {
        const exists = await this.CarExists(ctx, id);
        if (exists) throw new Error(`Машина ${id} уже существует`);

        const car = { ID: id, Color: color, Brand: brand, Owner: owner, docType: 'car' };
        await ctx.stub.putState(id, Buffer.from(JSON.stringify(car)));
        return JSON.stringify(car);
    }

    async GetAllCars(ctx) {
        const allResults = [];
        const iterator = await ctx.stub.getStateByRange('', '');
        let result = await iterator.next();
        while (!result.done) {
            const strValue = Buffer.from(result.value.value.toString()).toString('utf8');
            try {
                const record = JSON.parse(strValue);
                if (record.docType === 'car') allResults.push(record);
            } catch (err) { console.log(err); }
            result = await iterator.next();
        }
        return JSON.stringify(allResults);
    }

    async CarExists(ctx, id) {
        const carJSON = await ctx.stub.getState(id);
        return carJSON && carJSON.length > 0;
    }
}

module.exports = Cars;
```
```js
// my-project/chaincode-javascript/index.js
'use strict';
const cars = require('./lib/cars');
module.exports.Cars = cars;
module.exports.contracts = [cars];
```

## Шаг 3. Оптимизированный API-слой (express.js)
```js
// my-project/application-javascript/express.js
require('dotenv').config();

const express = require('express');
const cors = require('cors');
const path = require('path');
const fs = require('fs');
const { Gateway, Wallets } = require('fabric-network');
const FabricCAServices = require('fabric-ca-client');

const app = express();
app.use(express.json());
app.use(cors());

const {
    CHANNEL_NAME, CHAINCODE_NAME, CONTRACT_NAME,
    ADMIN_USER_ID, ADMIN_USER_PASSWD, API_PORT,
} = process.env;

const orgMspIds = {
    org1: process.env.ORG1_MSP_ID,
    org2: process.env.ORG2_MSP_ID,
};

// ==== Кэш connection-профилей — читаем файл с диска только один раз ====
const ccpCache = {};
function buildCCPOrg(organization) {
    if (ccpCache[organization]) return ccpCache[organization];

    const ccpPath = path.resolve(
        __dirname, '..', '..', 'fabric-samples', 'test-network',
        'organizations', 'peerOrganizations',
        `${organization}.example.com`, `connection-${organization}.json`
    );
    const ccp = JSON.parse(fs.readFileSync(ccpPath, 'utf8'));
    ccpCache[organization] = ccp;
    return ccp;
}

async function buildCAClient(ccp, organization) {
    const caInfo = ccp.certificateAuthorities[`ca.${organization}.example.com`];
    return new FabricCAServices(caInfo.url, {
        trustedRoots: caInfo.tlsCACerts.pem,
        verify: false,
    }, caInfo.caName);
}

async function buildWallet(organization) {
    const walletPath = path.join(process.cwd(), `wallet/${organization}`);
    return await Wallets.newFileSystemWallet(walletPath);
}

// ==== Пул Gateway-подключений — переиспользуем между запросами ====
const gatewayPool = new Map();

async function getOrCreateGateway(organization, userID) {
    const key = `${organization}:${userID}`;
    if (gatewayPool.has(key)) return gatewayPool.get(key);

    const ccp = buildCCPOrg(organization);
    const wallet = await buildWallet(organization);
    const identity = await wallet.get(userID);

    if (!identity) {
        throw new Error(`Личность ${userID} не найдена — сначала выполните enroll`);
    }

    const gateway = new Gateway();
    await gateway.connect(ccp, { wallet, identity, discovery: { enabled: true, asLocalhost: true } });

    gatewayPool.set(key, gateway);
    return gateway;
}

async function getContract(organization, userID) {
    const gateway = await getOrCreateGateway(organization, userID);
    const network = await gateway.getNetwork(CHANNEL_NAME);
    return network.getContract(CHAINCODE_NAME, CONTRACT_NAME);
}

// ==== Идентити: enroll администратора и регистрация пользователя ====
async function enrollAdmin(organization) {
    organization = organization.toLowerCase();
    const ccp = buildCCPOrg(organization);
    const caClient = await buildCAClient(ccp, organization);
    const wallet = await buildWallet(organization);

    if (await wallet.get(ADMIN_USER_ID)) return 'Admin уже существует';

    const enrollment = await caClient.enroll({
        enrollmentID: ADMIN_USER_ID,
        enrollmentSecret: ADMIN_USER_PASSWD,
    });

    await wallet.put(ADMIN_USER_ID, {
        credentials: { certificate: enrollment.certificate, privateKey: enrollment.key.toBytes() },
        mspId: orgMspIds[organization],
        type: 'X.509',
    });

    return 'Admin зарегистрирован';
}

async function enrollUser(organization, userId) {
    organization = organization.toLowerCase();
    const ccp = buildCCPOrg(organization);
    const caClient = await buildCAClient(ccp, organization);
    const wallet = await buildWallet(organization);

    if (await wallet.get(userId)) return `Пользователь ${userId} уже существует`;

    const adminIdentity = await wallet.get(ADMIN_USER_ID);
    const provider = wallet.getProviderRegistry().getProvider(adminIdentity.type);
    const adminUser = await provider.getUserContext(adminIdentity, ADMIN_USER_ID);

    const secret = await caClient.register(
        { affiliation: `${organization}.department1`, enrollmentID: userId, role: 'client' },
        adminUser
    );

    const enrollment = await caClient.enroll({ enrollmentID: userId, enrollmentSecret: secret });

    await wallet.put(userId, {
        credentials: { certificate: enrollment.certificate, privateKey: enrollment.key.toBytes() },
        mspId: orgMspIds[organization],
        type: 'X.509',
    });

    return `Пользователь ${userId} зарегистрирован`;
}

// ==== Единая обработка ошибок вместо дублирования try/catch в каждом контроллере ====
function asyncHandler(fn) {
    return (req, res, next) => fn(req, res, next).catch(next);
}

app.post('/enrollAdmin', asyncHandler(async (req, res) => {
    const result = await enrollAdmin(req.body.organization);
    res.json({ success: true, message: result });
}));

app.post('/enrollUser', asyncHandler(async (req, res) => {
    const { organization, userID } = req.body;
    if (!userID) return res.status(400).json({ success: false, error: 'Не указан userID' });

    const result = await enrollUser(organization, userID);
    res.json({ success: true, message: result });
}));

app.post('/addCar', asyncHandler(async (req, res) => {
    const { organization, userID, id, color, brand, owner } = req.body;
    const contract = await getContract(organization, userID);

    const result = await contract.submitTransaction('AddCar', id, color, brand, owner);
    res.json({ success: true, data: JSON.parse(result.toString()) });
}));

app.post('/getAllCars', asyncHandler(async (req, res) => {
    const { organization, userID } = req.body;
    const contract = await getContract(organization, userID);

    const result = await contract.evaluateTransaction('GetAllCars');
    res.json({ success: true, data: JSON.parse(result.toString()) });
}));

// Middleware обработки ошибок — обязательно последним
app.use((err, req, res, next) => {
    console.error(err);
    res.status(500).json({ success: false, error: err.message });
});

app.listen(API_PORT, () => console.log(`Сервер запущен на http://localhost:${API_PORT}`));
```

## Шаг 4. Единый стартовый скрипт (Linux/WSL-версия, bash)
```bash
#!/bin/bash
# Start.sh — поднятие сети, деплой chaincode и запуск API одной командой

set -e  # остановить скрипт при первой же ошибке, а не продолжать "вслепую"

echo "== Поднятие сети и создание канала =="
cd fabric-samples/test-network
./network.sh down
./network.sh up createChannel -c blockchain2024 -ca -s couchdb

echo "== Установка зависимостей chaincode =="
cd ../my-project/chaincode-javascript
npm install

echo "== Разворачивание chaincode =="
cd ../../fabric-samples/test-network
./network.sh deployCC -ccn Cars -ccl javascript -ccp ../../my-project/chaincode-javascript -c blockchain2024 -cci InitLedger

echo "== Очистка старых wallet перед новыми enroll =="
rm -rf ../../my-project/application-javascript/wallet

echo "== Установка зависимостей и запуск API =="
cd ../../my-project/application-javascript
npm install
npm run dev &

echo "== Готово! API запущен в фоне =="
```
*Отличия от изначального `Start.bat` из инструкции: добавлен `set -e` (немедленная остановка при ошибке вместо продолжения выполнения "вслепую" после сбоя одного из шагов), автоматическая очистка `wallet` встроена прямо в скрипт запуска (а не оставлена как отдельная ручная инструкция "не забудьте удалить"), и все параметры сети берутся из явных переменных, а не разбросаны по коду.*

## Шаг 5. Проверка через cURL
```bash
curl -X POST http://localhost:7000/enrollAdmin -H "Content-Type: application/json" -d '{"organization":"org1"}'

curl -X POST http://localhost:7000/enrollUser -H "Content-Type: application/json" -d '{"organization":"org1","userID":"user1"}'

curl -X POST http://localhost:7000/addCar -H "Content-Type: application/json" \
  -d '{"organization":"org1","userID":"user1","id":"car3","color":"blue","brand":"Toyota","owner":"user1"}'

curl -X POST http://localhost:7000/getAllCars -H "Content-Type: application/json" \
  -d '{"organization":"org1","userID":"user1"}'
```

## Итог: что покрывает практика
1. **Установка и запуск сети** — через `install-fabric.sh` и `network.sh up createChannel`.
2. **Написание chaincode** — класс `Cars`, наследующий `Contract`, с CRUD-методами и `docType` для фильтрации.
3. **Деплой** — через `deployCC` с корректными флагами имени, языка, пути, канала и инициализатора.
4. **Появление пользователей** — `enrollAdmin` (по фиксированному admin/adminpw) и `enrollUser` (регистрация от имени администратора + enroll по выданному секрету), с сохранением личностей в `wallet`.
5. **Оптимизация API** — кэш connection-профиля, пул Gateway-подключений, конфигурация через `.env`, единая обработка ошибок через `asyncHandler` и middleware, вместо разрозненных `try/catch` и жёстко прописанных значений в коде из исходной инструкции.

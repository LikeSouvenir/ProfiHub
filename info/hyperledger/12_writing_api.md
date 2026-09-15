# 12. Написание API

## 12.1. Зачем нужен отдельный API-слой
Браузер не может напрямую подключаться к Fabric (нет gRPC/TLS-клиента внутри браузера, и хранить приватные ключи пользователей на фронтенде небезопасно). Поэтому между фронтендом и сетью всегда стоит серверное приложение — обычно на Express.js — которое хранит `wallet`, управляет `Gateway`-подключениями и отдаёт фронтенду простой HTTP/JSON API.

```
Браузер (React)  →  HTTP/JSON  →  Express API  →  gRPC/TLS  →  Fabric-сеть
```

## 12.2. Подготовка проекта
```bash
mkdir -p ~/projects/hl-project/my-project/application-javascript
cd ~/projects/hl-project/my-project/application-javascript

npm init -y
npm install fabric-network fabric-ca-client express cors dotenv
```

## 12.3. Конфигурация — .env
```
CHANNEL_NAME=blockchain2024
CHAINCODE_NAME=Cars
CONTRACT_NAME=MyCars
ORG1_MSP_ID=Org1MSP
ORG2_MSP_ID=Org2MSP
ADMIN_USER_ID=admin
ADMIN_USER_PASSWD=adminpw
API_PORT=7000
```

## 12.4. Полное готовое решение — express.js
```js
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

// ---- Connection Profile: читаем один раз, кэшируем ----
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

// ---- Пул Gateway-подключений — переиспользуются между запросами ----
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

// ---- Идентичности ----
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

// ---- Единая обработка асинхронных ошибок вместо try/catch в каждом эндпоинте ----
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

app.post('/getCar', asyncHandler(async (req, res) => {
    const { organization, userID, carId } = req.body;
    const contract = await getContract(organization, userID);

    const result = await contract.evaluateTransaction('GetCar', carId);
    res.json({ success: true, data: JSON.parse(result.toString()) });
}));

app.post('/getAllCars', asyncHandler(async (req, res) => {
    const { organization, userID } = req.body;
    const contract = await getContract(organization, userID);

    const result = await contract.evaluateTransaction('GetAllCars');
    res.json({ success: true, data: JSON.parse(result.toString()) });
}));

app.post('/transferCar', asyncHandler(async (req, res) => {
    const { organization, userID, id, newOwner } = req.body;
    const contract = await getContract(organization, userID);

    const result = await contract.submitTransaction('TransferCar', id, newOwner);
    res.json({ success: true, previousOwner: result.toString() });
}));

// ---- Подписка на события реестра (полезно для realtime-обновлений фронтенда) ----
app.get('/events/:organization/:userID', asyncHandler(async (req, res) => {
    const { organization, userID } = req.params;
    const contract = await getContract(organization, userID);

    res.writeHead(200, { 'Content-Type': 'text/event-stream', 'Cache-Control': 'no-cache', Connection: 'keep-alive' });

    const listener = async (event) => {
        res.write(`data: ${event.payload.toString()}\n\n`);
    };
    await contract.addContractListener(listener);

    req.on('close', () => contract.removeContractListener(listener));
}));

// Middleware обработки ошибок — обязательно последним
app.use((err, req, res, next) => {
    console.error(err);
    res.status(500).json({ success: false, error: err.message });
});

app.listen(API_PORT, () => console.log(`API запущен: http://localhost:${API_PORT}`));
```

## 12.5. Полный справочник эндпоинтов этого API
| Метод | Путь | Тело запроса | Что делает |
|---|---|---|---|
| POST | `/enrollAdmin` | `{ organization }` | Enroll администратора организации (один раз для каждой организации) |
| POST | `/enrollUser` | `{ organization, userID }` | Регистрация и enroll нового пользователя |
| POST | `/addCar` | `{ organization, userID, id, color, brand, owner }` | Добавить машину (транзакция) |
| POST | `/getCar` | `{ organization, userID, carId }` | Получить одну машину (чтение) |
| POST | `/getAllCars` | `{ organization, userID }` | Получить все машины (чтение) |
| POST | `/transferCar` | `{ organization, userID, id, newOwner }` | Сменить владельца (транзакция) |
| GET | `/events/:organization/:userID` | — | Поток событий реестра в реальном времени (Server-Sent Events) |

## 12.6. Проверка через cURL — весь набор
```bash
curl -X POST http://localhost:7000/enrollAdmin -H "Content-Type: application/json" -d '{"organization":"org1"}'

curl -X POST http://localhost:7000/enrollUser -H "Content-Type: application/json" -d '{"organization":"org1","userID":"user1"}'

curl -X POST http://localhost:7000/addCar -H "Content-Type: application/json" \
  -d '{"organization":"org1","userID":"user1","id":"car3","color":"blue","brand":"Toyota","owner":"user1"}'

curl -X POST http://localhost:7000/getAllCars -H "Content-Type: application/json" \
  -d '{"organization":"org1","userID":"user1"}'

curl -X POST http://localhost:7000/transferCar -H "Content-Type: application/json" \
  -d '{"organization":"org1","userID":"user1","id":"car3","newOwner":"user2"}'
```

## 12.7. Справочник частых ошибок API-слоя
| Ошибка от сервера | Причина | Решение |
|---|---|---|
| `Личность user1 не найдена — сначала выполните enroll` | Забыт вызов `/enrollUser` перед использованием других эндпоинтов | Сначала `/enrollAdmin`, затем `/enrollUser`, только потом остальные запросы |
| `ENOENT: no such file or directory, connection-org1.json` | Сеть не поднята, либо путь до `test-network` в `buildCCPOrg` неверный | Проверить `docker ps` — сеть должна быть поднята; сверить относительный путь |
| `access denied: channel [blockchain2024] creator org unknown` | Личность создана для сети, которая с тех пор была пересоздана (`network.sh down` + `up`) | Удалить папку `wallet/` целиком и заново выполнить enroll |
| `Endorsement policy failure` | Не хватает подписей нужных организаций (см. конспект 07) | Проверить endorsement policy chaincode и убедиться, что оба peer'а из политики доступны |

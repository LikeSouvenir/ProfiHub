# API поверх Fabric и оптимизация

## Зачем нужен отдельный API-слой
Сайт/фронтенд не может напрямую "разговаривать" с сетью Fabric — нужен серверный код, который: хранит подключение к CA и peer'ам, управляет wallet с личностями пользователей, и предоставляет фронтенду простые HTTP-эндпоинты. Для этого поверх `fabric-network`/`fabric-ca-client` пишется обычное серверное приложение — например, на **Express.js**.

## Установка зависимостей API-слоя
```bash
mkdir -p my-project/application-javascript
cd my-project/application-javascript
npm init -y

npm install fabric-network fabric-ca-client express cors
```
- **fabric-network** — API для подключения к сети, отправки транзакций (`submitTransaction`), запросов на чтение (`evaluateTransaction`) и подписки на события реестра.
- **fabric-ca-client** — клиент центра сертификации, нужен для `enroll`/`register` (см. предыдущий конспект).
- **express** — HTTP-сервер, на котором будут описаны эндпоинты API.
- **cors** — разрешает фронтенду с другого адреса (например, `localhost:5173`) обращаться к API (например, на `localhost:7000`) — без этого браузер заблокирует запрос политикой Same-Origin.

## Базовая структура сервера
```js
const express = require('express');
const cors = require('cors');

const app = express();
app.use(express.json()); // разбор тела запроса как JSON
app.use(cors());

app.listen(7000, () => console.log('Сервер запущен на http://localhost:7000'));
```

## Разделение ответственности: чтение vs запись
В Fabric, как и в блокчейн-темах ранее, полезно чётко разделять два вида обращений к chaincode:
```js
// Чтение — не создаёт транзакцию, не требует согласования (endorsement), быстрее
const result = await contract.evaluateTransaction('GetCar', carId);

// Запись — создаёт полноценную транзакцию, требует endorsement и попадания в блок
const result = await contract.submitTransaction('AddCar', id, color, brand, owner);
```
Использование `evaluateTransaction` там, где на самом деле нужна запись (или наоборот) — частая ошибка новичков: `evaluateTransaction` не изменит состояние реестра, даже если внутри chaincode вызывается `putState`.

## Пример эндпоинтов
```js
app.post('/addCar', async (req, res) => {
    const { organization, userID, id, color, brand, owner } = req.body;

    try {
        const gateway = await getGateway(organization, userID);
        const contract = await getContract(gateway, 'MyCars');

        const result = await contract.submitTransaction('AddCar', id, color, brand, owner);
        await gateway.disconnect();

        res.json({ success: true, data: result.toString() });
    } catch (error) {
        res.status(500).json({ success: false, error: error.message });
    }
});

app.post('/getAllCars', async (req, res) => {
    const { organization, userID } = req.body;

    try {
        const gateway = await getGateway(organization, userID);
        const contract = await getContract(gateway, 'MyCars');

        const result = await contract.evaluateTransaction('GetAllCars');
        await gateway.disconnect();

        res.json({ success: true, data: JSON.parse(result.toString()) });
    } catch (error) {
        res.status(500).json({ success: false, error: error.message });
    }
});
```
*Обратите внимание на добавленную единообразную обёртку ответа (`{ success, data }` / `{ success, error }`) и `try/catch` вокруг каждого запроса — в исходном варианте инструкции ошибки chaincode просто отправлялись как есть, без единого формата и без корректного HTTP-статуса, что усложняет обработку ошибок на фронтенде.*

## Проблемы неоптимального подхода из типовой инструкции и как их исправить

### Проблема 1: подключение к Gateway создаётся заново на каждый запрос
В обычном "быстром" варианте `getGateway` вызывается и `gateway.disconnect()` — на **каждый** HTTP-запрос, даже если один и тот же пользователь дёргает API десятки раз подряд. Установка подключения (`gateway.connect`) — не бесплатная операция.

**Более оптимально:** держать пул уже открытых Gateway-подключений, переиспользуя их между запросами одного и того же пользователя, и закрывать соединение только при простое дольше определённого времени:
```js
const gatewayPool = new Map(); // ключ: `${organization}:${userID}`

async function getOrCreateGateway(organization, userID) {
    const key = `${organization}:${userID}`;

    if (gatewayPool.has(key)) {
        return gatewayPool.get(key);
    }

    const gateway = await getGateway(organization, userID);
    gatewayPool.set(key, gateway);
    return gateway;
}
```
*Для реального продакшена такой пул стоит дополнить логикой очистки неиспользуемых подключений (например, через `setInterval`), чтобы не накапливать бесконечно растущее число открытых соединений.*

### Проблема 2: connection-профиль (CCP) считывается с диска при каждом запросе
```js
function buildCCPOrg(organization) {
    const ccpPath = path.resolve(__dirname, '...', `connection-${organization}.json`);
    return JSON.parse(fs.readFileSync(ccpPath, 'utf8'));
}
```
Файл конфигурации сети (connection profile) не меняется во время работы приложения — читать его с диска на каждый вызов не нужно.

**Более оптимально:** прочитать и закэшировать один раз при старте сервера:
```js
const ccpCache = {};

function buildCCPOrg(organization) {
    if (ccpCache[organization]) {
        return ccpCache[organization];
    }

    const ccpPath = path.resolve(__dirname, '...', `connection-${organization}.json`);
    const ccp = JSON.parse(fs.readFileSync(ccpPath, 'utf8'));
    ccpCache[organization] = ccp;
    return ccp;
}
```

### Проблема 3: жёстко "зашитые" названия канала, chaincode и MSP ID прямо в коде
```js
const channelName = 'blockchain2024';
const chaincodeName = 'Cars';
const orgMspIds = { org1: 'Org1MSP', org2: 'Org2MSP' };
```
При смене имени канала/chaincode (что происходит регулярно на этапе разработки) приходится искать и менять эти значения по всему файлу API.

**Более оптимально:** вынести все такие значения в `.env`-файл и переменные окружения:
```
CHANNEL_NAME=blockchain2024
CHAINCODE_NAME=Cars
ORG1_MSP_ID=Org1MSP
ORG2_MSP_ID=Org2MSP
```
```js
require('dotenv').config();

const channelName = process.env.CHANNEL_NAME;
const chaincodeName = process.env.CHAINCODE_NAME;
```
Это делает конфигурацию единой точкой изменения и решает ту же проблему, что и предложенное в теме про запуск сети избегание ручного переименования организаций по всем файлам — конфигурационные значения должны находиться в одном явном месте, а не быть "размазаны" по коду.

### Проблема 4: отсутствие единой обработки ошибок и логирования
В исходном примере каждая функция (`addCar`, `getCar`, `getAllCars`) по отдельности содержит одинаковый `try/catch` с `console.error` и возвратом строки с ошибкой прямо как результата (`return errorMsg`), что смешивает успешные и ошибочные ответы в одном формате.

**Более оптимально:** вынести общую обработку в middleware Express и всегда выбрасывать (`throw`) настоящие ошибки из вспомогательных функций, а не возвращать текст ошибки как обычный результат:
```js
// Общий middleware обработки ошибок — в самом конце всех app.use()/app.post()
app.use((err, req, res, next) => {
    console.error(err);
    res.status(500).json({ success: false, error: err.message });
});

// Функции выбрасывают ошибку, а не возвращают строку с текстом ошибки
async function addCar(organization, userID, id, color, brand, owner) {
    const gateway = await getOrCreateGateway(organization, userID);
    const contract = await getContract(gateway, 'MyCars');
    const result = await contract.submitTransaction('AddCar', id, color, brand, owner);
    return result.toString();
}
```

## Итоговая оптимизированная схема API-слоя
```
.env (конфигурация: канал, chaincode, MSP ID)
        │
buildCCPOrg (с кэшированием файла)
        │
getOrCreateGateway (пул подключений на пользователя)
        │
Контроллеры (addCar, getCar, getAllCars — только бизнес-логика, throw при ошибке)
        │
Единый middleware обработки ошибок Express
```
Такой подход не меняет саму логику работы с Fabric (regenerate/enroll/submit остаются прежними), но заметно снижает накладные расходы на каждый запрос и делает код проще в сопровождении при росте проекта.

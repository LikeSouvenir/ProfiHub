# 05. Справочник методов API (Node.js SDK)

В отличие от публичных блокчейнов вроде Waves, где есть единый REST API ноды, в Fabric методы разделены между **двумя разными библиотеками**: одна отвечает за личности (CA), другая — за сами транзакции (Network). Ниже — полный справочник методов обеих, чтобы не искать их по кусочкам в разных источниках.

## 5.1. fabric-ca-client — методы работы с личностями

```js
const FabricCAServices = require('fabric-ca-client');
const caClient = new FabricCAServices(caUrl, tlsOptions, caName);
```

| Метод | Кто вызывает | Что делает |
|---|---|---|
| `caClient.enroll({ enrollmentID, enrollmentSecret })` | Сам пользователь (или код от его имени) | Получает сертификат и приватный ключ по уже выданному секрету |
| `caClient.register({ enrollmentID, role, affiliation, attrs }, registrarUser)` | Администратор (registrar) | Заводит запись о новом пользователе в CA, возвращает секрет для его `enroll` |
| `caClient.reenroll(currentUser)` | Пользователь с уже действующим сертификатом | Получает **новый** сертификат взамен старого (например, перед истечением срока действия) без повторной регистрации |
| `caClient.revoke({ enrollmentID, reason }, registrarUser)` | Администратор | Отзывает сертификат(ы) пользователя — они перестают приниматься сетью |
| `caClient.newIdentityService()` | Администратор | Возвращает сервис для управления записями идентичностей в CA (создание/чтение/удаление записи без выпуска сертификата) |
| `caClient.newAffiliationService()` | Администратор | Управление списком affiliation (подразделений организации) |

### Пример: полный набор операций с идентичностью
```js
// Регистрация нового пользователя администратором
const secret = await caClient.register({
    enrollmentID: 'user1',
    role: 'client',
    affiliation: 'org1.department1',
    attrs: [{ name: 'role', value: 'manager', ecert: true }], // кастомный атрибут для ABAC — см. 05.3
}, adminUser);

// Получение сертификата этим пользователем
const enrollment = await caClient.enroll({
    enrollmentID: 'user1',
    enrollmentSecret: secret,
});

// Отзыв сертификата (например, сотрудник уволился)
await caClient.revoke({
    enrollmentID: 'user1',
    reason: 'cessationofoperation',
}, adminUser);
```

## 5.2. fabric-network — методы работы с сетью и транзакциями

```js
const { Wallets, Gateway } = require('fabric-network');
```

### Wallets — хранилище личностей
| Метод | Что делает |
|---|---|
| `Wallets.newFileSystemWallet(path)` | Хранилище личностей в виде файлов на диске |
| `Wallets.newInMemoryWallet()` | Хранилище в оперативной памяти (личности пропадают при перезапуске процесса — удобно для тестов) |
| `wallet.put(label, identity)` | Сохранить личность под именем `label` |
| `wallet.get(label)` | Получить личность по имени |
| `wallet.remove(label)` | Удалить личность из хранилища |
| `wallet.list()` | Список всех сохранённых меток |

### Gateway — подключение к сети
| Метод | Что делает |
|---|---|
| `new Gateway()` | Создать объект будущего подключения |
| `gateway.connect(connectionProfile, options)` | Подключиться к сети от имени личности, указанной в `options.identity` |
| `gateway.getNetwork(channelName)` | Получить объект конкретного канала |
| `gateway.disconnect()` | Закрыть подключение |

```js
await gateway.connect(ccp, {
    wallet,
    identity: 'user1',
    discovery: { enabled: true, asLocalhost: true }, // asLocalhost: true — для локальной тестовой сети
});
```

### Network — канал
| Метод | Что делает |
|---|---|
| `network.getContract(chaincodeName, contractName?)` | Получить объект конкретного контракта внутри chaincode |
| `network.addBlockListener(callback)` | Подписаться на появление новых блоков в канале |
| `network.getChannel()` | Доступ к низкоуровневому API канала |

### Contract — вызов функций chaincode
| Метод | Меняет реестр? | Что делает |
|---|---|---|
| `contract.submitTransaction(fn, ...args)` | ✅ Да | Полноценная транзакция: endorsement + отправка Orderer'у + ожидание коммита |
| `contract.evaluateTransaction(fn, ...args)` | ❌ Нет | Только чтение — выполняется на одном peer'е, без Orderer'а и без записи |
| `contract.createTransaction(fn)` | ✅ Да (после `.submit()`) | Собрать транзакцию вручную, чтобы донастроить перед отправкой (см. ниже) |
| `contract.addContractListener(callback)` | — | Подписка на события, которые emit'ит именно этот контракт |
| `contract.evaluateTransaction(...)` | — | Альтернатива query, аналог "read-only call" |

### Расширенная сборка транзакции (createTransaction) — когда одного submitTransaction недостаточно
```js
const transaction = contract.createTransaction('AddCar');

// Указать transient-данные (приватные данные, не попадающие в публичный блок — см. конспект про деплой)
transaction.setTransient({
    price: Buffer.from('15000'),
});

// Указать, какие организации ДОЛЖНЫ одобрить именно эту транзакцию
transaction.setEndorsingOrganizations('Org1MSP', 'Org2MSP');

const result = await transaction.submit('car1', 'red', 'BMW', 'admin');
```

## 5.3. ABAC — атрибуты пользователя и проверка прав внутри chaincode
Атрибуты (`attrs`), заданные при `register`, попадают прямо в сертификат пользователя и доступны внутри chaincode:
```js
// Внутри chaincode
const hasManagerRole = ctx.clientIdentity.assertAttributeValue('role', 'manager');
const roleValue = ctx.clientIdentity.getAttributeValue('role');
```
Это позволяет строить проверки прав не только "по организации" (MSP ID), но и "по роли конкретного человека внутри организации" — например, разрешить `DeleteCar` только пользователям с атрибутом `role=manager`.

## 5.4. ctx.clientIdentity — методы внутри chaincode
| Метод | Что возвращает |
|---|---|
| `ctx.clientIdentity.getID()` | Полный уникальный ID вызывающего (включает сертификат в закодированном виде) |
| `ctx.clientIdentity.getMSPID()` | MSP ID организации вызывающего (например, `Org1MSP`) |
| `ctx.clientIdentity.getAttributeValue(attrName)` | Значение конкретного атрибута сертификата |
| `ctx.clientIdentity.assertAttributeValue(attrName, value)` | `true`/`false` — совпадает ли атрибут с ожидаемым значением |
| `ctx.clientIdentity.getX509Certificate()` | Полный разобранный X.509-сертификат |

## 5.5. ctx.stub — методы работы с реестром внутри chaincode
| Метод | Что делает |
|---|---|
| `ctx.stub.putState(key, buffer)` | Записать значение по ключу |
| `ctx.stub.getState(key)` | Прочитать значение по ключу |
| `ctx.stub.deleteState(key)` | Удалить значение |
| `ctx.stub.getStateByRange(startKey, endKey)` | Перебрать все ключи в диапазоне (`''`, `''` — весь диапазон) |
| `ctx.stub.getQueryResult(couchDbQueryString)` | Сложный запрос по полям JSON (**требует CouchDB**) |
| `ctx.stub.getHistoryForKey(key)` | Полная история изменений конкретного ключа (все версии) |
| `ctx.stub.putPrivateData(collection, key, buffer)` | Запись в приватную коллекцию данных (видна не всем участникам канала) |
| `ctx.stub.getPrivateData(collection, key)` | Чтение из приватной коллекции |
| `ctx.stub.setEvent(name, payload)` | Сгенерировать событие, на которое можно подписаться через `addContractListener` |
| `ctx.stub.getTxID()` | ID текущей транзакции |
| `ctx.stub.getTxTimestamp()` | Время транзакции (по времени Orderer'а, а не локальных часов peer'а) |

## 5.6. peer CLI — консольные команды (альтернатива SDK)
Всё, что делает Node.js SDK, можно вызвать и напрямую из консоли — полезно для отладки без написания кода:
```bash
# Чтение (query) — аналог evaluateTransaction
peer chaincode query -C blockchain2024 -n Cars -c '{"Args":["GetAllCars"]}'

# Запись (invoke) — аналог submitTransaction
peer chaincode invoke -o localhost:7050 --tls --cafile "$ORDERER_CA" \
  -C blockchain2024 -n Cars \
  --peerAddresses localhost:7051 --tlsRootCertFiles "$PEER0_ORG1_CA" \
  --peerAddresses localhost:9051 --tlsRootCertFiles "$PEER0_ORG2_CA" \
  -c '{"Args":["AddCar","car9","white","Kia","admin"]}'
```
*В реальной работе такие длинные команды с сертификатами удобнее не набирать руками, а брать из готового скрипта `setOrgEnv.sh` в `test-network` — он подставляет все переменные окружения (`$ORDERER_CA`, `$PEER0_ORG1_CA` и т.д.) автоматически.*

## Итог: сравнение "плоскости" API с Waves-подобными сетями
В отличие от Waves, где практически всё доступно через единый HTTP REST API ноды (`/transactions/broadcast`, `/addresses`, `/utils/...`), Fabric разделяет ответственность на уровне библиотек:
- **CA API** (`fabric-ca-client`) — только про личности;
- **Network API** (`fabric-network`) — только про сам вызов транзакций;
- **peer CLI** — низкоуровневая консольная альтернатива тому же самому.

Это не "меньше" функциональности, а другая архитектура: в публичной сети один узел отвечает буквально за всё, а в Fabric роли CA, peer и orderer физически разделены, поэтому и API у них разное.

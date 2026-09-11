# Как в системе появляются пользователи

## Личность в Fabric — это сертификат, а не пароль
В отличие от привычных веб-приложений с логином/паролем, в Hyperledger Fabric личность участника (**identity**) — это **цифровой сертификат X.509**, выданный конкретным **Certificate Authority (CA)** организации. Именно этим сертификатом подписываются все транзакции пользователя, и именно по нему сеть определяет, из какой организации пришёл запрос и какими правами обладает отправитель.

## Два разных действия: register и enroll
Появление нового пользователя в Fabric — это два разных, последовательных шага:

| Действие | Кто выполняет | Что происходит |
|---|---|---|
| **Register (регистрация)** | Администратор организации | CA "заводит запись" о будущем пользователе: присваивает ему `enrollmentID` и секрет для первого входа |
| **Enroll (получение сертификата)** | Сам пользователь (или код от его имени) | По `enrollmentID` и секрету CA выдаёт настоящий X.509-сертификат и приватный ключ — именно это и есть "личность" пользователя дальше |

```
Администратор:  register(userId, роль)  →  CA выдаёт временный секрет
                                                     │
Пользователь:   enroll(userId, секрет)  →  CA выдаёт сертификат + приватный ключ
                                                     │
                                          Личность сохраняется в Wallet
```

## Enrollment администратора организации
Прежде чем регистрировать кого-либо ещё, у организации должен появиться **администратор** — его учётные данные (`enrollmentID`/`enrollmentSecret`) заранее заданы при настройке CA организации (в тестовой сети — стандартные `admin`/`adminpw`):
```js
const enrollment = await caClient.enroll({
    enrollmentID: 'admin',
    enrollmentSecret: 'adminpw',
});
```
Администратор не "регистрируется" через `register` — он единственный, кто enroll'ится напрямую по заранее известным данным, и дальше уже сам регистрирует остальных пользователей организации.

## Wallet — локальное хранилище личностей
**Wallet (кошелёк)** — это не кошелёк с криптовалютой, как в Ethereum, а просто локальное хранилище (обычно — папка с файлами) для сертификатов и приватных ключей уже enroll'нутых личностей на стороне клиентского приложения:
```js
const { Wallets } = require('fabric-network');

const wallet = await Wallets.newFileSystemWallet('./wallet/org1');
```
Каждая идентичность сохраняется в `wallet` под своим `userId` — при следующих запросах приложение достаёт уже готовый сертификат из wallet, вместо повторного `enroll`.

## Полный процесс появления администратора в приложении
```js
async function enrollAdmin(organization) {
    const ccp = buildCCPOrg(organization);          // читаем connection-профиль организации
    const caClient = await buildCAClient(ccp, organization);
    const wallet = await buildWallet(organization);

    if (await wallet.get('admin')) {
        return 'Admin уже существует в wallet';
    }

    const enrollment = await caClient.enroll({
        enrollmentID: 'admin',
        enrollmentSecret: 'adminpw',
    });

    const x509Identity = {
        credentials: {
            certificate: enrollment.certificate,
            privateKey: enrollment.key.toBytes(),
        },
        mspId: 'Org1MSP',
        type: 'X.509',
    };

    await wallet.put('admin', x509Identity);
    return 'Admin зарегистрирован';
}
```

## Регистрация и enroll обычного пользователя
Обычного пользователя регистрирует уже сам **администратор организации** (используя свою, ранее полученную личность):
```js
async function enrollUser(organization, userId) {
    const ccp = buildCCPOrg(organization);
    const caClient = await buildCAClient(ccp, organization);
    const wallet = await buildWallet(organization);

    if (await wallet.get(userId)) {
        return `Пользователь ${userId} уже существует`;
    }

    // Получаем личность администратора из wallet — от его имени будем регистрировать нового пользователя
    const adminIdentity = await wallet.get('admin');
    const provider = wallet.getProviderRegistry().getProvider(adminIdentity.type);
    const adminUser = await provider.getUserContext(adminIdentity, 'admin');

    // Шаг 1: РЕГИСТРАЦИЯ — CA выдаёт секрет для нового пользователя
    const secret = await caClient.register(
        {
            affiliation: `${organization}.department1`,
            enrollmentID: userId,
            role: 'client',
        },
        adminUser // регистрация выполняется ОТ ИМЕНИ администратора
    );

    // Шаг 2: ENROLL — по секрету получаем настоящий сертификат
    const enrollment = await caClient.enroll({
        enrollmentID: userId,
        enrollmentSecret: secret,
    });

    const x509Identity = {
        credentials: {
            certificate: enrollment.certificate,
            privateKey: enrollment.key.toBytes(),
        },
        mspId: orgMspIds[organization],
        type: 'X.509',
    };

    await wallet.put(userId, x509Identity);
    return `Пользователь ${userId} зарегистрирован и enroll'нут`;
}
```

## MSP ID — принадлежность к организации
**MSP ID** (например, `Org1MSP`, `Org2MSP`) — идентификатор, "приклеенный" к личности при её создании, указывающий, из какой именно организации этот сертификат. Именно MSP ID (а не сам `userId`) используется в chaincode при проверке прав:
```js
// Внутри chaincode: доступ к MSP ID вызывающего транзакцию
const clientMSPID = ctx.clientIdentity.getMSPID();

if (clientMSPID === 'Org3MSP') {
    throw new Error('Организации Org3 запрещено вызывать эту функцию');
}
```
Так реализуются ограничения доступа на уровне бизнес-логики — например, в теме про токен ERC-20 именно так запрещается организации Org3 инициализировать контракт или чеканить токены.

## Использование личности для вызова транзакции: Gateway
```js
async function getGateway(organization, userId) {
    const ccp = buildCCPOrg(organization);
    const wallet = await buildWallet(organization);

    const identity = await wallet.get(userId);
    if (!identity) {
        throw new Error(`Личность ${userId} не найдена в wallet — сначала выполните enroll`);
    }

    const gateway = new Gateway();
    await gateway.connect(ccp, {
        wallet,
        identity,
        discovery: { enabled: true, asLocalhost: true },
    });

    return gateway;
}
```
`Gateway` — точка подключения к сети Fabric от имени конкретной, уже enroll'нутой личности из wallet. Все дальнейшие вызовы транзакций (`submitTransaction`/`evaluateTransaction`) через этот `gateway` будут подписаны именно сертификатом выбранного пользователя.

## Важный практический момент: пересоздание сети и wallet
Если тестовая сеть была перезапущена (`./network.sh down` + `up`), центр сертификации генерирует **новые** корневые сертификаты — старые личности в `wallet`, выданные предыдущим CA, становятся недействительными. Перед повторной работой с пересозданной сетью папку `wallet` нужно **полностью очистить** и заново выполнить `enrollAdmin` и `enrollUser` — иначе приложение будет получать необъяснимые ошибки авторизации при попытке использовать "устаревшие" сертификаты.

## Оптимизация процесса управления пользователями
1. **Проверка существования перед enroll/register** (как в примерах выше через `wallet.get(...)`) обязательна — повторный `enroll` того же `enrollmentID` без явной причины приведёт к ошибке CA.
2. **Не размещать реальные приватные ключи и секреты admin/adminpw в общем репозитории** — для тестовой сети это допустимо, но в реальном проекте секреты CA должны браться из переменных окружения или менеджера секретов.
3. **Разделять роли пользователей на уровне `affiliation`/`role`** при регистрации (`client`, `admin`, `peer`) — это позволяет позже гибче настраивать политику доступа именно на уровне Fabric, а не только внутри бизнес-логики chaincode.
4. **Кэшировать Gateway-подключения** для часто используемых пользователей вместо создания нового подключения на каждый HTTP-запрос API — см. подробнее в конспекте про оптимизацию API.

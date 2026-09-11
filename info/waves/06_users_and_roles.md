# Как в системе появляются пользователи

Важно различать два разных уровня "пользователей":

1. **Сетевой уровень** — blockchain-адрес участника сети (пара ключей), появляется, когда для него генерируют ключи (например, через `config-manager` — так появились адреса нод в `credentials.txt`) и ему выдают сетевые permission (`connect`, `contract_developer` и т.д.). Этим управляет администратор **сети**, а не вашего контракта.
2. **Уровень бизнес-логики контракта** — то, что ваш контракт сам считает "пользователем": запись в своём состоянии с ролью, именем, статусом активности. Этим управляет администратор **контракта**, и это то, что вы пишете сами в коде.

Ниже — типовой паттерн появления пользователей на уровне контракта, на примере реального контракта из учебной базы (цифровизация экспортно-ориентированного производства).

## Шаг 1. Первый пользователь появляется сам — через init()
```java
@ContractInit
public void init() {
    String wallet = call.getCaller();   // адрес того, кто задеплоил контракт (транзакция 103)
    ORGS.put(wallet, new Organization(wallet, "Operator", OrgType.OPERATOR, "ALL", ""));
    USERS.put(wallet, new User(wallet, wallet, Role.ADMIN, "Admin", "", "Administrator"));
}
```
`call.getCaller()` внутри `@ContractInit` — это адрес того, кто отправил транзакцию `CreateContract`. Простейший и самый частый паттерн: **тот, кто задеплоил контракт, автоматически становится администратором**. Никакой отдельной "регистрации" для первого пользователя не нужно — он появляется в тот же момент, что и сам контракт.

## Шаг 2. Остальные пользователи — через метод регистрации, доступный только администратору
```java
@ContractAction
public void registerOrg(String orgWallet, String name, String orgType, String region,
                        String description, String userWallet, String userRole,
                        String fullName, String contact, String position) {
    User admin = USERS.get(call.getCaller());
    if (Objects.equals(admin.role, Role.ADMIN)) {
        if (!ORGS.has(orgWallet)) {
            // организация ещё не существует — создаём её смарт-аккаунт
            ORGS.put(orgWallet, new Organization(orgWallet, name, orgType, region, description));
        } else {
            // организация уже есть — добавляем к ней ещё одного сотрудника
            Organization org = ORGS.get(orgWallet);
            org.keys = org.keys + "," + userWallet;
            ORGS.put(orgWallet, org);
        }
        if (!USERS.has(userWallet))
            USERS.put(userWallet, new User(userWallet, orgWallet, userRole, fullName, contact, position));
    }
}
```
Что здесь происходит по шагам:
1. Метод узнаёт, **кто** его вызвал (`call.getCaller()`), находит этого человека в `USERS` и проверяет, что у него роль `ADMIN`. Это и есть контроль доступа — без него **любой** участник сети мог бы зарегистрировать себя с любой ролью.
2. Если организации с таким `orgWallet` ещё нет — она создаётся впервые.
3. Если организация уже есть — новый сотрудник просто добавляется в список её ключей (`org.keys`), дублирования организации не происходит.
4. Пользователь регистрируется в `USERS` со своей персональной ролью — то есть роль хранится у **человека**, а не у организации: в одной организации могут быть сотрудники с разными ролями.
5. Проверка `!USERS.has(userWallet)` защищает от повторной регистрации одного и того же кошелька — вызов метода второй раз с теми же данными ничего не сломает.

## Шаг 3. Управление пользователями дальше
Обычно рядом с регистрацией сразу нужны ещё два метода — изменение данных и блокировка:
```java
@ContractAction
public void updateAccount(String orgWallet, String name, String region, String description,
                          String userWallet, String fullName, String contact, String position) {
    User admin = USERS.get(call.getCaller());
    if (Objects.equals(admin.role, Role.ADMIN)) {
        // пустая строка в orgWallet/userWallet = "не трогать этот объект"
        if (!userWallet.isEmpty()) {
            User user = USERS.get(userWallet);
            user.fullName = fullName;
            user.contact = contact;
            user.position = position;
            USERS.put(userWallet, user);
        }
    }
}

@ContractAction
public void setActive(String orgWallet, String userWallet, boolean isActive) {
    User admin = USERS.get(call.getCaller());
    if (Objects.equals(admin.role, Role.ADMIN)) {
        if (!userWallet.isEmpty()) {
            User user = USERS.get(userWallet);
            user.isActive = isActive;
            USERS.put(userWallet, user);
        }
    }
}
```
`setActive(..., false)` — это "мягкое удаление": пользователь не стирается из состояния (история его действий в блокчейне всё равно неизменна), а просто помечается неактивным. Дальше все бизнес-методы контракта должны проверять `user.isActive` перед тем, как что-то разрешить.

## Общий принцип, который стоит запомнить
Любой метод контракта, который что-то меняет, обычно начинается с одной и той же трёхшаговой проверки:
```java
String wallet = call.getCaller();
User user = USERS.get(wallet);
if (user == null || !user.isActive) throw new RuntimeException("User blocked or not found");
if (!user.role.equals(Role.NEEDED_ROLE)) throw new RuntimeException("Access denied");
```
1. Кто вызывает — есть ли такой пользователь вообще.
2. Активен ли он.
3. Есть ли у него нужная для этого действия роль.

Это и есть ответ на вопрос "как в системе появляются пользователи": не автоматически по факту транзакции в сеть (это дало бы любому желающему возможность стать кем угодно), а осознанно — через метод контракта, который сам решает, кто и с какими правами регистрируется, и на каждом следующем шаге проверяет эти права заново.

# Написание смарт-контракта

Смарт-контракт Waves Enterprise — это обычное Java/Kotlin-приложение на **WE Contract SDK**, которое умеет разговаривать с нодой по gRPC. Нода вызывает методы контракта и записывает то, что контракт изменил, в блокчейн.

## Шаг 1. Создание проекта
1. В IntelliJ IDEA: **New Project** → язык **Java**, система сборки — **Gradle**, JDK — **corretto-17** (или другой дистрибутив JDK 17), Gradle DSL — **Kotlin**.
2. Если появится предупреждение `Module JDK is not defined` — нажмите **Setup SDK** и выберите corretto-17.
3. Дождитесь установки Gradle-зависимостей.

## Шаг 2. Корневой build.gradle.kts
В корне проекта нужен `build.gradle.kts`, подключающий репозитории Waves Enterprise и версии базовых библиотек:
```kotlin
import io.spring.gradle.dependencymanagement.dsl.DependencyManagementExtension

val weSdkBomVersion: String by project

plugins {
    kotlin("jvm") apply false
    id("io.spring.dependency-management") apply false
    `maven-publish`
}

allprojects {
    group = "com.wavesenterprise.app"
    version = "1.0.0-SNAPSHOT"

    repositories {
        mavenCentral()
        maven {
            name = "maven-releases"
            url = uri("https://artifacts.wavesenterprise.com/repository/maven-releases/")
            mavenContent { releasesOnly() }
        }
    }
}

subprojects {
    apply(plugin = "io.spring.dependency-management")
    apply(plugin = "kotlin")

    the<DependencyManagementExtension>().apply {
        imports {
            mavenBom("com.wavesenterprise:we-sdk-bom:$weSdkBomVersion")
        }
    }
}
```
Ключевая строка — `maven { url = uri("https://artifacts.wavesenterprise.com/repository/maven-releases/") }`: именно там лежит `we-sdk-bom` и SDK для контрактов, их нет в обычном Maven Central.

`settings.gradle.kts`:
```kotlin
pluginManagement {
    repositories {
        gradlePluginPortal()
        mavenCentral()
        maven { url = uri("https://artifacts.wavesenterprise.com/repository/maven-releases/") }
    }
}

rootProject.name = "MyContractProject"
include("SmartContract")   // модуль самого контракта
```

## Шаг 3. Модуль контракта
ПКМ по корню проекта → **New → Module**, назовите его, например, `SmartContract`.

`SmartContract/build.gradle.kts`:
```kotlin
import com.github.jengelman.gradle.plugins.shadow.tasks.ShadowJar

plugins {
    application
    java
    id("com.github.johnrengelman.shadow") version "7.1.2"   // собирает fat-jar со всеми зависимостями
}

dependencies {
    implementation(kotlin("stdlib"))
    implementation("com.wavesenterprise:we-contract-sdk-grpc")
    implementation("com.fasterxml.jackson.datatype:jackson-datatype-jsr310")
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
}

tasks.withType<ShadowJar> {
    manifest {
        attributes["Main-Class"] = "com.wavesenterprise.app.Dispatcher"
    }
}
```
`ShadowJar` собирает **fat-jar** — один jar-файл со всеми зависимостями внутри. Это важно: контракт запускается внутри отдельного минимального Docker-контейнера, где никаких библиотек, кроме JRE, нет — поэтому всё нужное должно быть упаковано в один файл.

## Шаг 4. Структура пакета контракта
Внутри `com.<название_проекта>` создайте три директории:
```
com.myproject/
├── Dispatcher.java     # точка входа приложения
├── api/                # интерфейсы, роли, статусы — общий контракт API
├── app/                # сама реализация контракта
└── domain/              # модели данных (сущности, которые храним в состоянии)
```

### Dispatcher — точка входа
```java
package com.wavesenterprise.app;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.wavesenterprise.app.app.Contract;
import com.wavesenterprise.sdk.contract.core.dispatch.ContractDispatcher;
import com.wavesenterprise.sdk.contract.grpc.GrpcJacksonContractDispatcherBuilder;

public class Dispatcher {
    public static void main(String[] args) {
        ContractDispatcher contractDispatcher = GrpcJacksonContractDispatcherBuilder
                .builder()
                .contractHandlerType(Contract.class)
                .objectMapper(new ObjectMapper())
                .build();
        contractDispatcher.dispatch();
    }
}
```
`Dispatcher` — это класс с `main()`, который запускает бесконечный цикл: подключается к ноде по gRPC и ждёт входящих вызовов, передавая их в класс `Contract`.

### api/IContract — контракт интерфейса
Здесь удобно держать сигнатуры методов и константы (роли, статусы), общие для всего проекта:
```java
package com.wavesenterprise.app.api;

public interface IContract {
    class Role {
        public static final String ADMIN = "ADMIN";
        public static final String OPERATOR = "OPERATOR";
        // ...другие роли вашего бизнес-процесса
    }
}
```

### domain — модели данных
Обычный класс с полями, геттерами и сеттерами (ПКМ по классу → **Generate → Getter and Setter**):
```java
package com.wavesenterprise.app.domain;

public class User {
    public String wallet;
    public String fullName;
    public String role;
    // геттеры/сеттеры
}
```
Эти классы сериализуются в JSON движком SDK автоматически — вручную писать сериализацию не нужно.

### app/Contract — сама логика
```java
package com.wavesenterprise.app.app;

import com.wavesenterprise.sdk.contract.api.annotation.ContractAction;
import com.wavesenterprise.sdk.contract.api.annotation.ContractHandler;
import com.wavesenterprise.sdk.contract.api.annotation.ContractInit;
import com.wavesenterprise.sdk.contract.api.domain.ContractCall;
import com.wavesenterprise.sdk.contract.api.state.ContractState;
import com.wavesenterprise.sdk.contract.api.state.TypeReference;
import com.wavesenterprise.sdk.contract.api.state.mapping.Mapping;

@ContractHandler
public class Contract implements IContract {

    private final ContractCall call;
    private final ContractState state;
    private final Mapping<User> USERS;

    public Contract(ContractCall call, ContractState state) {
        this.call = call;
        this.state = state;
        USERS = state.getMapping(new TypeReference<User>() {}, "USERS");
    }

    @ContractInit
    public void init() {
        // вызывается один раз при деплое контракта (транзакция 103)
    }

    @ContractAction
    public void someBusinessMethod(String param1, int param2) {
        // вызывается при обращении к методу (транзакция 104)
    }
}
```

Разберём аннотации и типы:

| Элемент | Назначение |
|---|---|
| `@ContractHandler` | помечает класс, который SDK будет вызывать как реализацию контракта |
| `@ContractInit` | метод-точка входа — выполняется один раз, при создании контракта (транзакция 103 `CreateContract`) |
| `@ContractAction` | обычный бизнес-метод контракта — вызывается транзакцией 104 `CallContract`, параметры метода приходят из `params` транзакции |
| `ContractCall call` | информация о текущем вызове — например, `call.getCaller()` возвращает адрес отправителя транзакции |
| `ContractState state` | доступ к постоянному хранилищу контракта (то, что сохраняется в блокчейне между вызовами) |
| `Mapping<T> state.getMapping(...)` | типизированная "таблица" ключ → объект внутри состояния контракта, аналог словаря/hashmap, но с персистентностью в реестре |

`Mapping` — самый частый способ хранить данные в контракте: `USERS.put(wallet, user)` кладёт объект по ключу, `USERS.get(wallet)` читает, `USERS.has(wallet)` проверяет наличие. Каждая такая операция в итоге превращается в запись `ключ:значение`, которую можно прочитать напрямую через REST API (см. файл про API).

## Как нода узнаёт, какой именно метод вызвать
В одном классе, помеченном `@ContractHandler`, обычно живёт **несколько** методов с `@ContractAction` (в примере кейса про экспортное производство их больше десяти: `registerOrg`, `addProduct`, `createBid` и т.д.). Транзакция 104 передаёт только `contractId` и массив `params` — значит, нужен способ сказать, какой конкретно метод вызывать.

Для этого в `params` **всегда** добавляется служебная пара с ключом `action`, значение которой — точное имя Java-метода:
```json
{
  "type": "string",
  "key": "action",
  "value": "registerOrg"
}
```
Остальные элементы `params` — это уже аргументы самого метода, они мэпятся на параметры по имени (`key` совпадает с именем аргумента в сигнатуре метода). Без правильного `action` нода не поймёт, какой метод вызывать, даже если остальные параметры переданы верно. Тот же принцип действует и для `@ContractInit` при создании контракта (транзакция 103) — на случай, если в контракте несколько init-методов.

## Шаг 5. Сборка
```bash
./gradlew build
```
или через панель **Gradle → SmartContract → Tasks → build → build** в IntelliJ IDEA.

Дальше нужно упаковать jar в Docker-образ — это следующий шаг, деплой.

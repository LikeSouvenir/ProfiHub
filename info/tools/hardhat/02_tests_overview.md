# Как устроены тесты в Hardhat

## Из чего состоит тестовое окружение
Тесты в Hardhat (в JS/TS-варианте) строятся на связке нескольких инструментов:
| Инструмент | Роль |
|---|---|
| **Mocha** | Фреймворк для запуска тестов: организует их в блоки `describe`/`it` |
| **Chai** | Библиотека для проверки утверждений (assertions): `expect(x).to.equal(y)` |
| **hardhat-chai-matchers** | Дополнительные Chai-проверки, специфичные для блокчейна: проверка событий, `revert`, изменения баланса |
| **Hardhat Network** | Встроенная локальная блокчейн-сеть, автоматически поднимающаяся при запуске тестов |
| **ethers.js** (через hardhat-toolbox) | Для деплоя контрактов и вызова их функций внутри теста |

## Где хранятся тесты
```
test/
├── Storage.js
├── Token.js
└── Auction.js
```
Каждый файл обычно соответствует одному контракту, но это не строгое правило — можно организовывать тесты и по-другому (по функциональности, по сценариям).

## Запуск тестов
```bash
npx hardhat test                       # запустить ВСЕ тесты
npx hardhat test test/Storage.js        # запустить только один конкретный файл
npx hardhat test --grep "должен увеличивать значение"  # запустить тесты, чьё описание содержит указанный текст
```

## Базовая структура теста
```js
const { expect } = require("chai");

describe("Storage", function () {
    it("должен возвращать 0 сразу после деплоя", async function () {
        const Storage = await ethers.getContractFactory("Storage");
        const storage = await Storage.deploy();

        expect(await storage.get()).to.equal(0);
    });
});
```
- `describe("Storage", ...)` — группа тестов, относящихся к одному контракту (или одной функциональности).
- `it("должен ...", ...)` — один конкретный тестовый сценарий; текст описания должен ясно объяснять, что именно проверяется.
- `expect(...).to.equal(...)` — само утверждение: если оно не выполняется, тест считается проваленным.

## Fixtures — эффективная подготовка окружения теста
Разворачивать контракт заново перед **каждым** тестом — правильно с точки зрения изоляции тестов, но может быть медленно при большом числе тестов. Hardhat решает это через `loadFixture`, который деплоит контракт один раз и затем мгновенно "откатывает" блокчейн к этому состоянию перед каждым тестом:

```js
const { loadFixture } = require("@nomicfoundation/hardhat-toolbox/network-helpers");
const { expect } = require("chai");

describe("Storage", function () {
    async function deployStorageFixture() {
        const [owner, otherAccount] = await ethers.getSigners();

        const Storage = await ethers.getContractFactory("Storage");
        const storage = await Storage.deploy();

        return { storage, owner, otherAccount };
    }

    it("должен возвращать 0 сразу после деплоя", async function () {
        const { storage } = await loadFixture(deployStorageFixture);
        expect(await storage.get()).to.equal(0);
    });

    it("должен сохранять новое значение после set", async function () {
        const { storage } = await loadFixture(deployStorageFixture);
        await storage.set(42);
        expect(await storage.get()).to.equal(42);
    });
});
```
`loadFixture` под капотом использует специальную функцию блокчейна для мгновенного "снимка" и "отката" состояния сети — вместо повторного деплоя контракта перед каждым тестом (что требует времени и газа), Hardhat просто возвращает сеть к уже сохранённому снимку.

## Тестовые аккаунты: ethers.getSigners()
```js
const [owner, addr1, addr2] = await ethers.getSigners();
```
Возвращает массив тестовых аккаунтов, предоставленных Hardhat Network (те же 20 аккаунтов с 10 000 ETH, что видны при `npx hardhat node`). Первый аккаунт (`owner`) по договорённости обычно используется как аккаунт, деплоящий контракт.

## Проверка ревертов (revert)
Одна из важнейших вещей для тестирования смарт-контрактов — проверка, что функция действительно отклоняет некорректный вызов:
```js
it("должен отклонять вызов не от владельца", async function () {
    const { storage, otherAccount } = await loadFixture(deployStorageFixture);

    await expect(
        storage.connect(otherAccount).set(100)
    ).to.be.revertedWith("Только владелец может менять значение");
});
```
`.connect(otherAccount)` меняет, от чьего имени вызывается функция контракта — позволяет протестировать поведение именно для другого, не основного аккаунта.

## Проверка событий (events)
```js
it("должен генерировать событие ValueChanged при set", async function () {
    const { storage } = await loadFixture(deployStorageFixture);

    await expect(storage.set(42))
        .to.emit(storage, "ValueChanged")
        .withArgs(42);
});
```

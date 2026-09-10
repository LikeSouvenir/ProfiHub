# Написание тестов на JS

## Хуки Mocha: before, beforeEach, after, afterEach
```js
const { expect } = require("chai");

describe("Token", function () {
    let token, owner, addr1;

    // Выполняется один раз, перед ВСЕМИ тестами в этом describe
    before(async function () {
        console.log("Подготовка перед всеми тестами");
    });

    // Выполняется перед КАЖДЫМ тестом — удобно для деплоя "чистого" контракта заново
    beforeEach(async function () {
        [owner, addr1] = await ethers.getSigners();

        const Token = await ethers.getContractFactory("Token");
        token = await Token.deploy(1000);
    });

    it("владелец должен получить весь начальный выпуск токенов", async function () {
        expect(await token.balanceOf(owner.address)).to.equal(1000);
    });

    it("баланс нового аккаунта должен быть равен 0", async function () {
        expect(await token.balanceOf(addr1.address)).to.equal(0);
    });
});
```
*На практике вместо `beforeEach` с деплоем чаще используют `loadFixture` (см. предыдущий конспект) — он даёт тот же результат "чистого" состояния перед каждым тестом, но заметно быстрее за счёт снимков блокчейна вместо повторного деплоя.*

## Полезные Chai-матчеры для смарт-контрактов
```js
// Проверка точного равенства значения
expect(await token.totalSupply()).to.equal(1000);

// Проверка изменения баланса ETH конкретного адреса после транзакции
await expect(() =>
    owner.sendTransaction({ to: addr1.address, value: 100 })
).to.changeEtherBalance(addr1, 100);

// Проверка изменения баланса токенов (ERC20) сразу у нескольких адресов
await expect(
    token.transfer(addr1.address, 50)
).to.changeTokenBalances(token, [owner, addr1], [-50, 50]);

// Проверка, что вызов отклонён с конкретной причиной (require/revert с сообщением)
await expect(token.transfer(addr1.address, 999999))
    .to.be.revertedWith("Недостаточно токенов");

// Проверка отклонения с кастомной ошибкой (error ...; из Solidity)
await expect(token.transfer(addr1.address, 999999))
    .to.be.revertedWithCustomError(token, "InsufficientBalance");

// Проверка события с конкретными аргументами
await expect(token.transfer(addr1.address, 50))
    .to.emit(token, "Transfer")
    .withArgs(owner.address, addr1.address, 50);
```

## Работа со временем в тестах
Некоторые контракты (например, аукционы, стейкинг с периодом блокировки) зависят от времени блока. Hardhat Network позволяет управлять временем искусственно:
```js
const { time } = require("@nomicfoundation/hardhat-toolbox/network-helpers");

it("должен разблокировать средства через 30 дней", async function () {
    const { lock } = await loadFixture(deployLockFixture);

    const thirtyDays = 30 * 24 * 60 * 60; // 30 дней в секундах
    await time.increase(thirtyDays);       // "перематываем" блокчейн-время вперёд

    await expect(lock.withdraw()).not.to.be.reverted;
});
```
Без такого инструмента протестировать логику, зависящую от времени, пришлось бы либо ждать реальные 30 дней, либо переписывать контракт под тесты — `time.increase` решает эту проблему мгновенно.

## Параметризованные тесты (проверка нескольких значений в цикле)
```js
const testCases = [
    { input: 0, expected: 0 },
    { input: 50, expected: 50 },
    { input: 1000, expected: 1000 },
];

describe("Storage: set/get для разных значений", function () {
    testCases.forEach(({ input, expected }) => {
        it(`должен сохранять значение ${input}`, async function () {
            const { storage } = await loadFixture(deployStorageFixture);

            await storage.set(input);
            expect(await storage.get()).to.equal(expected);
        });
    });
});
```
Такой подход избавляет от копирования одного и того же теста несколько раз с разными числами вручную.

## Тестирование вложенных вызовов между контрактами
```js
it("Factory должен создавать новый контракт Wallet", async function () {
    const { factory, owner } = await loadFixture(deployFactoryFixture);

    const tx = await factory.createWallet(owner.address);
    const receipt = await tx.wait();

    // Проверяем, что после транзакции появился новый адрес контракта
    const walletsCount = await factory.getWalletsCount();
    expect(walletsCount).to.equal(1);
});
```

## Полный пример теста для контракта Storage с проверкой владельца
```js
const { expect } = require("chai");
const { loadFixture } = require("@nomicfoundation/hardhat-toolbox/network-helpers");

describe("Storage", function () {
    async function deployFixture() {
        const [owner, otherAccount] = await ethers.getSigners();
        const Storage = await ethers.getContractFactory("Storage");
        const storage = await Storage.deploy();
        return { storage, owner, otherAccount };
    }

    describe("Деплой", function () {
        it("значение сразу после деплоя должно быть 0", async function () {
            const { storage } = await loadFixture(deployFixture);
            expect(await storage.get()).to.equal(0);
        });
    });

    describe("Запись значения", function () {
        it("владелец может изменить значение", async function () {
            const { storage } = await loadFixture(deployFixture);
            await storage.set(42);
            expect(await storage.get()).to.equal(42);
        });

        it("должно сгенерироваться событие ValueChanged", async function () {
            const { storage } = await loadFixture(deployFixture);
            await expect(storage.set(42))
                .to.emit(storage, "ValueChanged")
                .withArgs(42);
        });

        it("не владелец не может изменить значение", async function () {
            const { storage, otherAccount } = await loadFixture(deployFixture);
            await expect(
                storage.connect(otherAccount).set(100)
            ).to.be.revertedWith("Только владелец может менять значение");
        });
    });
});
```
Вложенные блоки `describe` (по деплою, по каждой группе функциональности) делают вывод тестов структурированным и легко читаемым, особенно при большом количестве тестовых сценариев.

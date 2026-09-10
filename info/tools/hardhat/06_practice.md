# Практика: полный цикл Hardhat — от установки до отчётов

Собираем воедино весь материал темы: создаём проект, пишем контракт, тестируем его и на JS, и на Solidity в Forge-стиле, и получаем отчёты по газу и покрытию.

## Шаг 1. Создание и настройка проекта
```bash
mkdir hardhat-bank && cd hardhat-bank
npm init -y
npm install --save-dev hardhat
npx hardhat init
# Выбираем: Create a JavaScript project
```

```bash
npm install --save-dev @nomicfoundation/hardhat-toolbox solidity-coverage forge-std
```

## Шаг 2. Контракт — простой банковский вклад
```solidity
// contracts/Bank.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract Bank {
    address public owner;
    mapping(address => uint256) public balances;

    event Deposited(address indexed user, uint256 amount);
    event Withdrawn(address indexed user, uint256 amount);

    constructor() {
        owner = msg.sender;
    }

    function deposit() public payable {
        require(msg.value > 0, unicode"Сумма должна быть больше нуля");
        balances[msg.sender] += msg.value;
        emit Deposited(msg.sender, msg.value);
    }

    function withdraw(uint256 amount) public {
        require(balances[msg.sender] >= amount, unicode"Недостаточно средств");

        balances[msg.sender] -= amount;
        payable(msg.sender).transfer(amount);

        emit Withdrawn(msg.sender, amount);
    }
}
```

## Шаг 3. Тесты на JS
```js
// test/Bank.js
const { expect } = require("chai");
const { loadFixture } = require("@nomicfoundation/hardhat-toolbox/network-helpers");

describe("Bank", function () {
    async function deployBankFixture() {
        const [owner, user1] = await ethers.getSigners();
        const Bank = await ethers.getContractFactory("Bank");
        const bank = await Bank.deploy();
        return { bank, owner, user1 };
    }

    it("должен принимать депозит и обновлять баланс", async function () {
        const { bank, user1 } = await loadFixture(deployBankFixture);

        await bank.connect(user1).deposit({ value: ethers.parseEther("1") });

        expect(await bank.balances(user1.address)).to.equal(ethers.parseEther("1"));
    });

    it("должен генерировать событие Deposited", async function () {
        const { bank, user1 } = await loadFixture(deployBankFixture);

        await expect(bank.connect(user1).deposit({ value: ethers.parseEther("1") }))
            .to.emit(bank, "Deposited")
            .withArgs(user1.address, ethers.parseEther("1"));
    });

    it("должен отклонять депозит на 0", async function () {
        const { bank, user1 } = await loadFixture(deployBankFixture);

        await expect(
            bank.connect(user1).deposit({ value: 0 })
        ).to.be.revertedWith(unicode"Сумма должна быть больше нуля");
    });

    it("должен позволять снять внесённые средства и менять баланс ETH", async function () {
        const { bank, user1 } = await loadFixture(deployBankFixture);

        await bank.connect(user1).deposit({ value: ethers.parseEther("1") });

        await expect(
            bank.connect(user1).withdraw(ethers.parseEther("1"))
        ).to.changeEtherBalance(user1, ethers.parseEther("1"));
    });

    it("должен отклонять снятие больше, чем есть на балансе", async function () {
        const { bank, user1 } = await loadFixture(deployBankFixture);

        await expect(
            bank.connect(user1).withdraw(ethers.parseEther("1"))
        ).to.be.revertedWith(unicode"Недостаточно средств");
    });
});
```

## Шаг 4. Тесты на Solidity (Forge-стиль)
```solidity
// contracts/Bank.t.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";
import "./Bank.sol";

contract BankTest is Test {
    Bank public bank;
    address public user1 = address(0x1);

    function setUp() public {
        bank = new Bank();
        vm.deal(user1, 10 ether); // выдаём тестовому адресу баланс для депозитов
    }

    function testDepositUpdatesBalance() public {
        vm.prank(user1);
        bank.deposit{value: 1 ether}();

        assertEq(bank.balances(user1), 1 ether);
    }

    function testRevertsOnZeroDeposit() public {
        vm.prank(user1);
        vm.expectRevert(unicode"Сумма должна быть больше нуля");

        bank.deposit{value: 0}();
    }

    function testWithdrawReturnsFunds() public {
        vm.prank(user1);
        bank.deposit{value: 1 ether}();

        uint256 balanceBefore = user1.balance;

        vm.prank(user1);
        bank.withdraw(1 ether);

        assertEq(user1.balance, balanceBefore + 1 ether);
        assertEq(bank.balances(user1), 0);
    }

    function testRevertsOnInsufficientBalance() public {
        vm.prank(user1);
        vm.expectRevert(unicode"Недостаточно средств");

        bank.withdraw(1 ether);
    }
}
```

## Шаг 5. Запуск всех тестов
```bash
npx hardhat test              # JS-тесты (и Solidity-тесты, если поддерживаются одной командой)
npx hardhat test solidity     # только тесты в Forge-стиле, если нужно разделить запуск
```

## Шаг 6. Отчёт по газу
```js
// hardhat.config.js (добавляем секцию)
module.exports = {
    solidity: "0.8.24",
    gasReporter: {
        enabled: true,
    },
};
```
```bash
REPORT_GAS=true npx hardhat test
```
Отчёт покажет, сколько газа тратят `deposit` и `withdraw` — обе функции содержат `require` и изменение состояния (`mapping`), поэтому их стоимость стоит держать под контролем при дальнейшем расширении контракта.

## Шаг 7. Отчёт по покрытию
```bash
npx hardhat coverage
```
Ожидаемый результат — высокое покрытие для `Bank.sol`, так как тесты (и на JS, и на Solidity) покрывают оба "счастливых" сценария (`deposit`/`withdraw` работают) и оба сценария ошибок (`revert` при нулевом депозите и при нехватке средств).

## Итог: что демонстрирует практика
1. **Установка** — от `npm init` до готового Hardhat-проекта с `hardhat-toolbox`, `solidity-coverage` и `forge-std`.
2. **Контракт `Bank.sol`** — простая, но реалистичная логика с `require`, событиями и передачей ETH — достаточно для демонстрации всех видов проверок.
3. **JS-тесты** — используют `loadFixture`, Chai-матчеры для событий (`.to.emit`), ревертов (`.to.be.revertedWith`) и изменения баланса ETH (`.to.changeEtherBalance`).
4. **Solidity-тесты** — тот же набор сценариев, но написанный в Forge-стиле с `vm.prank`, `vm.deal`, `vm.expectRevert` и стандартными ассертами (`assertEq`) — выполняются быстрее и не требуют перехода в JS-окружение.
5. **Отчёты** — gas reporter показывает стоимость каждой функции, coverage — какая часть кода реально проверена, что вместе даёт полную картину качества и эффективности контракта перед деплоем.

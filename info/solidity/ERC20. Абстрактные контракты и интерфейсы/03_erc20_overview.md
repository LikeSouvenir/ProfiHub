# ERC20: обзор стандарта и разбор контракта

**ERC20** (Ethereum Request for Comments 20) — это стандарт токенов в сети Ethereum. Он описывает единый интерфейс, который должен реализовать смарт-контракт, чтобы кошельки, биржи и другие контракты могли работать с ним одинаковым образом — независимо от того, что это за токен: стейблкоин, игровая валюта или токен голосования.

По сути, ERC20 — это классический пример применения **интерфейса** из предыдущего конспекта: стандарт как раз и оформлен как интерфейс `IERC20`.

## Интерфейс IERC20
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IERC20 {
    // ==== Функции чтения (view) ====
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function allowance(address owner, address spender) external view returns (uint256);

    // ==== Функции записи ====
    function transfer(address to, uint256 amount) external returns (bool);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address from, address to, uint256 amount) external returns (bool);

    // ==== События ====
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
}
```

## Разбор функций стандарта

| Функция | Назначение |
|---|---|
| `totalSupply()` | Общее количество токенов в обращении |
| `balanceOf(address)` | Баланс токенов на конкретном адресе |
| `transfer(to, amount)` | Перевод токенов от `msg.sender` на адрес `to` |
| `approve(spender, amount)` | Разрешить адресу `spender` списать до `amount` токенов с твоего баланса |
| `allowance(owner, spender)` | Сколько токенов `spender` ещё может списать с адреса `owner` |
| `transferFrom(from, to, amount)` | Перевод токенов **от чужого имени**, но только в пределах разрешённого через `approve` лимита |

### Зачем нужны approve и transferFrom
Прямой `transfer` работает только "от своего имени". Но часто нужно, чтобы **другой контракт** (например, децентрализованная биржа) мог списать твои токены самостоятельно, без твоего непосредственного участия в момент сделки. Для этого:
1. Пользователь вызывает `approve(dexAddress, 100)` — разрешает бирже списать до 100 токенов.
2. Биржа в удобный момент вызывает `transferFrom(userAddress, someoneElse, 100)` — переводит токены, используя выданное разрешение.

## Полная реализация токена ERC20
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./IERC20.sol";

contract MyToken is IERC20 {
    string public name = "MyToken";
    string public symbol = "MTK";
    uint8 public decimals = 18;

    uint256 private _totalSupply;

    mapping(address => uint256) private _balances;
    mapping(address => mapping(address => uint256)) private _allowances;

    constructor(uint256 _initialSupply) {
        // Изначальный выпуск токенов получает тот, кто деплоит контракт
        _totalSupply = _initialSupply;
        _balances[msg.sender] = _initialSupply;
        emit Transfer(address(0), msg.sender, _initialSupply);
    }

    function totalSupply() external view override returns (uint256) {
        return _totalSupply;
    }

    function balanceOf(address account) external view override returns (uint256) {
        return _balances[account];
    }

    function transfer(address to, uint256 amount) external override returns (bool) {
        require(_balances[msg.sender] >= amount, unicode"Недостаточно токенов");

        _balances[msg.sender] -= amount;
        _balances[to] += amount;

        emit Transfer(msg.sender, to, amount);
        return true;
    }

    function approve(address spender, uint256 amount) external override returns (bool) {
        _allowances[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }

    function allowance(address owner, address spender) external view override returns (uint256) {
        return _allowances[owner][spender];
    }

    function transferFrom(address from, address to, uint256 amount) external override returns (bool) {
        require(_balances[from] >= amount, unicode"Недостаточно токенов у отправителя");
        require(_allowances[from][msg.sender] >= amount, unicode"Лимит списания превышен");

        _balances[from] -= amount;
        _balances[to] += amount;
        _allowances[from][msg.sender] -= amount;

        emit Transfer(from, to, amount);
        return true;
    }
}
```

## Пример использования из другого контракта
Благодаря тому, что `MyToken` реализует `IERC20`, любой сторонний контракт может работать с ним через интерфейс, не зная деталей реализации:

```solidity
contract SimpleStore {
    IERC20 public token;
    address public owner;

    constructor(address _tokenAddress) {
        token = IERC20(_tokenAddress);
        owner = msg.sender;
    }

    // Покупатель заранее делает approve на адрес SimpleStore,
    // после чего магазин сам списывает токены через transferFrom
    function buyItem(uint256 _price) public {
        token.transferFrom(msg.sender, owner, _price);
    }
}
```

## Почему в реальных проектах ERC20 обычно не пишут с нуля
На практике почти никто не реализует ERC20 вручную — используют проверенную, многократно проаудированную библиотеку **OpenZeppelin**:
```solidity
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract MyToken is ERC20 {
    constructor(uint256 _initialSupply) ERC20("MyToken", "MTK") {
        _mint(msg.sender, _initialSupply);
    }
}
```
Здесь `ERC20` из OpenZeppelin — это как раз **абстрактный контракт**: он уже реализует всю стандартную логику (`transfer`, `approve` и т.д.), а разработчику остаётся только вызвать `_mint` в конструкторе и, при желании, переопределить (`override`) отдельные функции под свои нужды.

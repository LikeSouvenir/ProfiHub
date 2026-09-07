# Разбор практического задания: Токен с системой наград

В этом задании объединим интерфейс `IERC20`, абстрактный контракт и полноценную реализацию токена в одном небольшом проекте, разбитом на файлы.

## Условие задачи
- `IERC20.sol` — стандартный интерфейс токена.
- `RewardableToken.sol` — **абстрактный** контракт: реализует весь стандарт ERC20, но оставляет нереализованной функцию `rewardRate()` — размер награды за стейкинг, который каждый конкретный токен должен задать сам.
- `GameToken.sol` — конкретный токен игры, наследующий `RewardableToken` и задающий свою ставку награды.

## Решение с подробными комментариями

**IERC20.sol**
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IERC20 {
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address to, uint256 amount) external returns (bool);

    event Transfer(address indexed from, address indexed to, uint256 value);
}
```

**RewardableToken.sol**
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./IERC20.sol";

// abstract, потому что rewardRate() не реализована — сам по себе контракт не деплоится
abstract contract RewardableToken is IERC20 {
    string public name;
    uint256 private _totalSupply;
    mapping(address => uint256) private _balances;

    // Общий для всех наследников счётчик "очков стейкинга"
    mapping(address => uint256) public stakedBalance;

    constructor(string memory _name, uint256 _initialSupply) {
        name = _name;
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

    // Общая логика стейкинга — одинакова для любого токена-наследника
    function stake(uint256 _amount) public {
        require(_balances[msg.sender] >= _amount, unicode"Недостаточно токенов для стейкинга");
        _balances[msg.sender] -= _amount;
        stakedBalance[msg.sender] += _amount;
    }

    // Считает награду, используя ставку каждого конкретного токена
    function calculateReward(address _user) public view returns (uint256) {
        return stakedBalance[_user] * rewardRate();
    }

    // Нереализованная функция — каждый токен-наследник обязан задать свою ставку награды
    function rewardRate() public view virtual returns (uint256);
}
```

**GameToken.sol**
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./RewardableToken.sol";

contract GameToken is RewardableToken {
    // Своя ставка награды: 5% за единицу застейканных токенов
    uint256 public constant REWARD_RATE = 5;

    constructor(uint256 _initialSupply) RewardableToken("GameToken", _initialSupply) {}

    // Обязательная реализация — иначе GameToken тоже остался бы abstract
    function rewardRate() public pure override returns (uint256) {
        return REWARD_RATE;
    }
}
```

## Что здесь происходит
1. `IERC20` описывает **что** должен уметь любой ERC20-токен.
2. `RewardableToken` реализует **общую** логику (баланс, переводы, стейкинг), но специально оставляет `rewardRate()` без реализации — эта деталь у каждого токена своя, поэтому контракт `abstract` и не может быть задеплоен напрямую.
3. `GameToken` — единственный, кого реально можно задеплоить: он наследует всю готовую логику из `RewardableToken` и добавляет только то, что уникально для него — ставку награды.

Такой подход экономит время: если завтра появится `StableToken` с другой ставкой награды, достаточно унаследовать `RewardableToken` и переопределить одну функцию `rewardRate()`, не переписывая весь ERC20-функционал заново.

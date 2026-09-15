# Модуль 5: Интерфейсы и стандарт токенов ERC-20 — Эталонное решение

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/// @title IERC20 - Официальный интерфейс стандарта EIP-20
interface IERC20 {
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);

    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address to, uint256 value) external returns (bool);
    function allowance(address owner, address spender) external view returns (uint256);
    function approve(address spender, uint256 value) external returns (bool);
    function transferFrom(address from, address to, uint256 value) external returns (bool);
}

/// @title MyCustomToken - Реализация стандарта ERC-20 с нуля
contract MyCustomToken is IERC20 {
    string public name = "ProfiHub Token";
    string public symbol = "PHT";
    uint8 public immutable decimals = 18;

    uint256 private _totalSupply;
    mapping(address => uint256) private _balances;
    mapping(address => mapping(address => uint256)) private _allowances;

    constructor(uint256 _initialSupply) {
        // Начальная эмиссия с учетом 18 десятичных знаков
        _mint(msg.sender, _initialSupply * 10 ** decimals);
    }

    function totalSupply() external view override returns (uint256) {
        return _totalSupply;
    }

    function balanceOf(address account) external view override returns (uint256) {
        return _balances[account];
    }

    function transfer(address to, uint256 amount) external override returns (bool) {
        _transfer(msg.sender, to, amount);
        return true;
    }

    function allowance(address owner, address spender) external view override returns (uint256) {
        return _allowances[owner][spender];
    }

    function approve(address spender, uint256 amount) external override returns (bool) {
        _approve(msg.sender, spender, amount);
        return true;
    }

    function transferFrom(address from, address to, uint256 amount) external override returns (bool) {
        uint256 currentAllowance = _allowances[from][msg.sender];
        require(currentAllowance >= amount, "ERC20: insufficient allowance");

        // Уменьшаем выделенный лимит
        _approve(from, msg.sender, currentAllowance - amount);

        // Переводим токены
        _transfer(from, to, amount);
        return true;
    }

    /// @notice Публичная функция сжигания токенов
    function burn(uint256 amount) external {
        _burn(msg.sender, amount);
    }

    // --- Внутренняя логика изменения балансов ---

    function _transfer(address from, address to, uint256 amount) internal {
        require(from != address(0), "ERC20: transfer from zero");
        require(to != address(0), "ERC20: transfer to zero");
        require(_balances[from] >= amount, "ERC20: transfer exceeds balance");

        _balances[from] -= amount;
        _balances[to] += amount;
        emit Transfer(from, to, amount);
    }

    function _mint(address account, uint256 amount) internal {
        require(account != address(0), "ERC20: mint to zero");

        _totalSupply += amount;
        _balances[account] += amount;
        emit Transfer(address(0), account, amount);
    }

    function _burn(address account, uint256 amount) internal {
        require(account != address(0), "ERC20: burn from zero");
        require(_balances[account] >= amount, "ERC20: burn exceeds balance");

        _balances[account] -= amount;
        _totalSupply -= amount;
        emit Transfer(account, address(0), amount);
    }

    function _approve(address owner, address spender, uint256 amount) internal {
        require(owner != address(0), "ERC20: approve from zero");
        require(spender != address(0), "ERC20: approve to zero");

        _allowances[owner][spender] = amount;
        emit Approval(owner, spender, amount);
    }
}

/// @title TokenLocker - Контракт удержания токенов, взаимодействующий через IERC20
contract TokenLocker {
    mapping(address => mapping(address => uint256)) public lockedBalances;

    event TokensLocked(address indexed token, address indexed user, uint256 amount);

    /// @notice Блокировка токенов
    /// @dev Пользователь предварительно должен вызвать approve(address(this), amount) в смарт-контракте токена
    function lockTokens(address _tokenAddress, uint256 _amount) external {
        require(_amount > 0, "Amount must be > 0");

        IERC20 token = IERC20(_tokenAddress);

        // Перевод токенов со счета пользователя на счет локера
        bool success = token.transferFrom(msg.sender, address(this), _amount);
        require(success, "Token transfer failed");

        lockedBalances[_tokenAddress][msg.sender] += _amount;

        emit TokensLocked(_tokenAddress, msg.sender, _amount);
    }
}
```

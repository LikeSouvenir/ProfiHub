# Глобальные переменные: address, msg и block

Solidity предоставляет доступ к специальным глобальным переменным, которые содержат информацию о текущей транзакции и свойствах блокчейна.

## Атрибуты address
Тип `address` хранит 20-байтовый адрес аккаунта Ethereum. У него есть важный встроенный атрибут:
- `address.balance` — возвращает текущий баланс указанного адреса в wei.

```solidity
function getBalance(address _user) public view returns (uint256) {
    // Возвращает баланс любого переданного адреса
    return _user.balance;
}

function getMyBalance() public view returns (uint256) {
    // Возвращает баланс текущего смарт-контракта
    return address(this).balance;
}
```

## msg и его атрибуты
Объект `msg` содержит информацию о текущем вызове смарт-контракта (транзакции).
Самые важные атрибуты:
- `msg.sender` (address) — адрес того, кто вызвал данную функцию. Это может быть как обычный пользователь (кошелек), так и другой смарт-контракт.
- `msg.value` (uint) — количество эфира (в wei), которое было отправлено ВМЕСТЕ с вызовом этой функции (функция должна иметь модификатор `payable`).
- `msg.data` (bytes) — полные данные вызова (calldata).
- `msg.sig` (bytes4) — первые 4 байта от `msg.data`, которые являются идентификатором вызываемой функции.

```solidity
function deposit() public payable {
    require(msg.value > 0, "You must send some Ether");
    // msg.sender отправил msg.value эфира на этот контракт
}
```

## block и его атрибуты (Обзор значений блока)
Объект `block` содержит информацию о текущем блоке в блокчейне, в который будет включена эта транзакция.
- `block.timestamp` (uint) — время создания блока в формате Unix timestamp (количество секунд с 1 января 1970 года). Ранее использовался алиас `now`.
- `block.number` (uint) — текущий номер блока.
- `block.coinbase` (address) — адрес майнера (или валидатора), который добыл этот блок.
- `block.chainid` (uint) — идентификатор текущей сети (например, 1 для Ethereum Mainnet, 11155111 для Sepolia).
- `block.gaslimit` (uint) — максимальный лимит газа для текущего блока.

```solidity
function getBlockInfo() public view returns (uint, uint, address) {
    return (
        block.timestamp, // Время создания
        block.number,    // Номер блока
        block.coinbase   // Кто добыл блок
    );
}
```
*Примечание: Использовать `block.timestamp` для генерации случайных чисел небезопасно, так как валидаторы могут манипулировать этим значением в небольших пределах.*

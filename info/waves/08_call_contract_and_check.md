# Вызов методов контракта и проверка результата

## Транзакция 104 CallContract
Любой бизнес-метод, помеченный `@ContractAction`, вызывается транзакцией типа `104`. В `params` передаётся массив пар "ключ-значение-тип" — они попадают в аргументы вашего Java-метода по имени параметра.

Например, для метода
```java
@ContractAction
public void registerOrg(String orgWallet, String name, String orgType, String region,
                        String description, String userWallet, String userRole,
                        String fullName, String contact, String position) { ... }
```
запрос будет таким:
```bash
curl -s -X POST "$NODE_URL/transactions/signAndBroadcast" \
  --header 'Content-Type: application/json' \
  --header 'X-API-Key: we' \
  -d "{
    \"type\": 104,
    \"version\": 2,
    \"sender\": \"$SENDER\",
    \"password\": \"$KEYPAIR_PASSWORD\",
    \"contractId\": \"$CONTRACT_ID\",
    \"contractVersion\": 1,
    \"fee\": 500000,
    \"params\": [
      { \"type\": \"string\", \"key\": \"action\",     \"value\": \"registerOrg\" },
      { \"type\": \"string\", \"key\": \"orgWallet\",  \"value\": \"3N...\" },
      { \"type\": \"string\", \"key\": \"name\",       \"value\": \"Supplier LLC\" },
      { \"type\": \"string\", \"key\": \"orgType\",    \"value\": \"SUPPLIER\" },
      { \"type\": \"string\", \"key\": \"region\",     \"value\": \"KZ\" },
      { \"type\": \"string\", \"key\": \"description\",\"value\": \"\" },
      { \"type\": \"string\", \"key\": \"userWallet\", \"value\": \"3N...\" },
      { \"type\": \"string\", \"key\": \"userRole\",   \"value\": \"SUPPLIER\" },
      { \"type\": \"string\", \"key\": \"fullName\",   \"value\": \"Иван Иванов\" },
      { \"type\": \"string\", \"key\": \"contact\",    \"value\": \"ivan@example.com\" },
      { \"type\": \"string\", \"key\": \"position\",   \"value\": \"Manager\" }
    ]
  }" | tee call_result.json
```
Допустимые типы в `params`: `string`, `integer`, `boolean`, `binary` — должны совпадать с типом аргумента метода. Первый элемент массива — всегда служебная пара `action` с именем вызываемого Java-метода (см. файл про написание контракта); остальные — уже аргументы самого метода по именам.

`sender` в этом запросе — не обязательно администратор: это тот, **от чьего имени будет вызван метод**, то есть `call.getCaller()` внутри контракта вернёт именно этот адрес. Метод сам решит, пускать этого отправителя или нет (см. файл про пользователей и роли).

## Проверка статуса вызова
Так же, как и для деплоя — по `id` из ответа:
```bash
CALL_ID=$(jq -r '.id' call_result.json)
curl -s "$NODE_URL/contracts/status/$CALL_ID" | jq
```
`status: "ERROR"` — контракт сам отклонил вызов (например, сработал `throw new RuntimeException(...)` из-за нехватки прав) — это ожидаемая, не критическая ситуация. `status: "FAILURE"` — что-то пошло не так на уровне платформы/контейнера, стоит смотреть логи контракта:
```bash
docker logs $(docker ps --filter "ancestor=registry.hub.docker.com/myuser/my-contract:1.0.0" -q)
```

## Чтение состояния контракта
Всё, что контракт положил в `Mapping` через `state`, можно прочитать напрямую по REST API, без отдельного вызова метода — состояние публично читается любым участником сети:
```bash
curl -s "$NODE_URL/contracts/$CONTRACT_ID" | jq
```
Ответ — массив пар ключ-значение всего состояния контракта:
```json
[
  { "type": "string", "key": "USERS_3N...", "value": "{\"wallet\":\"3N...\",\"role\":\"SUPPLIER\", ...}" }
]
```
Если нужно не всё состояние, а конкретный ключ:
```bash
curl -s "$NODE_URL/contracts/$CONTRACT_ID/USERS_3N..." | jq
```
А если ключей много и нужна выборка по маске — используется `POST /contracts/{contractId}` с параметрами `limit`, `offset`, `matches` (регулярное выражение по ключам):
```bash
curl -s -X POST "$NODE_URL/contracts/$CONTRACT_ID" \
  -H 'Content-Type: application/json' \
  -d '{"limit": 50, "offset": 0, "matches": "^USERS_.*"}' | jq
```

## Информация о конкретной транзакции
Если нужно посмотреть саму транзакцию (не статус её исполнения контрактом, а данные транзакции как таковой — отправитель, комиссия, высота блока):
```bash
curl -s "$NODE_URL/transactions/info/$CALL_ID" | jq
```
А результат исполнения именно контрактом (что конкретно изменилось в state) — по идентификатору `105 ExecutedContract` транзакции, которую нода создаёт автоматически после успешного 104-вызова:
```bash
curl -s "$NODE_URL/contracts/executed-tx-for/$CALL_ID" | jq
```
В ответе, в поле `resultsMap` (или `results` для более старых версий транзакции), будут именно те пары ключ-значение, которые контракт записал в state в рамках этого вызова — удобно для аудита "что именно изменилось этим вызовом", а не выгружать весь state целиком.

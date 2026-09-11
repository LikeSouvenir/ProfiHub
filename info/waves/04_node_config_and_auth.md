# Настройка ноды и авторизация в API

## Что внутри node.conf
Главный конфиг ноды — `node.conf`. Config-manager генерирует его автоматически, но полезно понимать ключевые секции, если нужно что-то донастроить вручную.

Раздел REST API и авторизации:
```hocon
api {
  rest {
    enable = yes
    bind-address = "0.0.0.0"   # на каких адресах слушать; 0.0.0.0 — на всех
    port = 6862
    tls = no
    cors = yes

    # ограничение выдачи по адресу — не бесконечная выгрузка всех транзакций сразу
    transactions-by-address-limit = 10000
  }

  auth {
    type = "api-key"   # второй вариант — "oauth2", если развёрнут отдельный сервис авторизации

    # хэш строки API-ключа — сам ключ в открытом виде нигде в конфиге не хранится
    api-key-hash = "G3PZAsY6EA8esgpKxB2UYTQJZJPzc14gLnNbm2xvcDf6"
  }
}
```
Важно: в `credentials.txt` вы видите **сам ключ** (`API key: we`), а в `node.conf` хранится только его **хэш** (`api-key-hash`) — так безопаснее, если конфиг случайно утечёт. Если нужно сгенерировать хэш вручную под свой ключ, используется утилита `ApiKeyHash` из пакета генераторов:
```bash
java -jar generator-x.x.x.jar ApiKeyHash api-key-hash.conf
```
где `api-key-hash.conf`:
```hocon
apikeyhash-generator {
  crypto.type = WAVES
  api-key = "мой секретный ключ"
}
```

## Как авторизовать запрос к приватным методам
Не все методы REST API требуют авторизации — чтение публичных данных (блоки, статус ноды, статус транзакции) открыто всем. А вот методы, которые трогают keystore ноды (например, подпись транзакции) требуют заголовок **`X-API-Key`** со значением API-ключа этой ноды:

```bash
curl -X POST \
  --header 'Content-Type: application/json' \
  --header 'Accept: application/json' \
  --header 'X-API-Key: we' \
  -d '{"...тело транзакции..."}' \
  http://localhost:6882/transactions/signAndBroadcast
```

## Почему curl/скрипт оптимальнее Swagger UI
Кликать в Swagger UI (Authorize → вставить ключ → раскрыть метод → Try it out → вставить JSON → Execute) удобно один раз, чтобы посмотреть, что вообще есть в API. Но для регулярной работы — сборка контракта, деплой, вызов методов, проверка результата — гораздо быстрее и надёжнее завести пару переменных окружения и curl-запросы:

```bash
export NODE_URL=http://localhost:6882
export SENDER=3Nt9234ZD7DiXrMUfCvjiJEGE63TwxDMjQS      # blockchain address ноды
export KEYPAIR_PASSWORD=gkCrlv9arT7EWNQUPdWqYg          # keypair password из credentials.txt
export API_KEY=we
```
Дальше любой запрос — это одна команда без ручных кликов, а главное — воспроизводимая: её можно положить в bash-скрипт и гонять на каждый деплой без повторения одних и тех же действий руками. Пример проверки, что нода жива:
```bash
curl -s "$NODE_URL/node/status" | jq
```
`jq` тут не обязателен, но сильно облегчает чтение JSON-ответа — сразу форматирует его красиво вместо одной строки.

## Проверка, что нода отвечает
```bash
curl -s "$NODE_URL/node/status"
```
Ответ:
```json
{
  "blockchainHeight": 47041,
  "stateHeight": 47041,
  "updatedTimestamp": 1544709501138,
  "updatedDate": "2018-12-13T13:58:21.138Z"
}
```
- `blockchainHeight` — высота последнего известной ноде блока;
- `stateHeight` — высота, до которой применено состояние (в нормальной ситуации совпадает с `blockchainHeight`);
- если `stateHeight` заметно отстаёт от `blockchainHeight` — нода ещё догоняет сеть (синхронизируется).

Версию платформы можно узнать так:
```bash
curl -s "$NODE_URL/node/version"
```

С этого момента у нас есть работающая, настроенная сеть, к которой можно обращаться скриптами — переходим к написанию самого смарт-контракта.

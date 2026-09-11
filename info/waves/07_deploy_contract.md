# Деплой смарт-контракта

Деплой состоит из двух независимых частей: **упаковать контракт в Docker-образ и опубликовать его в registry**, а затем **отправить в сеть транзакцию**, которая говорит ноде "скачай этот образ и запусти его как контракт".

## Шаг 1. Dockerfile контракта
В корне модуля контракта:
```dockerfile
FROM eclipse-temurin:17-jre-alpine
MAINTAINER Waves Enterprise <>

ENV JAVA_MEM="-Xmx256M"
ENV JAVA_OPTS=""

ADD build/libs/*-all.jar app.jar
RUN apk --no-cache add curl

RUN chmod +x app.jar
CMD ["/bin/sh", "-c", "java $JAVA_MEM $JAVA_OPTS -jar app.jar"]
```
`*-all.jar` — это тот самый fat-jar, который собирает плагин `shadow` (см. предыдущий файл): один файл со всем кодом и зависимостями, потому что внутри alpine-образа кроме JRE ничего нет.

## Шаг 2. Сборка и публикация образа
Скрипт `build_and_push_to_docker.sh` в одну команду делает всё сразу:
```bash
#!/bin/bash
# $1 — куда пушим, например registry.hub.docker.com/myuser/my-contract:1.0.0
./gradlew clean build
docker build . --platform linux/amd64 -t $1
docker push $1

inspectResult=$(docker inspect $1 | grep '"Id": "sha256')
imageHash=$(awk -F'"Id": "sha256:|",' '{print $2}' <<< "$inspectResult")
printf "image - $1 \nimageHash - $imageHash\n"
```
Запуск:
```bash
docker login registry.hub.docker.com   # один раз ввести логин/пароль Docker Hub
sh build_and_push_to_docker.sh registry.hub.docker.com/myuser/my-contract:1.0.0
```
На выходе скрипт печатает `image` (полный путь к образу) и `imageHash` (sha256-хэш образа) — оба значения понадобятся в транзакции создания контракта. Приватный registry (вместо Docker Hub) работает точно так же — меняется только адрес в `$1`.

## Шаг 3. Транзакция 103 CreateContract
Публикация контракта в блокчейн — это подписанная транзакция типа `103`. У неё два обязательных поля со ссылкой на образ и его хэш, плюс имя, отправитель и комиссия:

```bash
export NODE_URL=http://localhost:6882
export SENDER=3Nt9234ZD7DiXrMUfCvjiJEGE63TwxDMjQS
export KEYPAIR_PASSWORD=gkCrlv9arT7EWNQUPdWqYg

curl -s -X POST "$NODE_URL/transactions/signAndBroadcast" \
  --header 'Content-Type: application/json' \
  --header 'X-API-Key: we' \
  -d "{
    \"type\": 103,
    \"version\": 1,
    \"sender\": \"$SENDER\",
    \"password\": \"$KEYPAIR_PASSWORD\",
    \"image\": \"registry.hub.docker.com/myuser/my-contract:1.0.0\",
    \"imageHash\": \"7d3b915c82930dd79591aab040657338f64e5d8b842abe2d73d5c8f828584b65\",
    \"contractName\": \"my-contract\",
    \"params\": [
      { \"type\": \"string\", \"key\": \"action\", \"value\": \"init\" }
    ],
    \"fee\": 100000000
  }" | tee create_result.json
```
`/transactions/signAndBroadcast` — комбинированный метод: подписывает транзакцию приватным ключом `$SENDER`, который лежит в keystore ноды (для этого и нужен `password`), и сразу отправляет её в сеть, без промежуточной пересылки данных между `/sign` и `/broadcast`.

Обратите внимание на параметр `action` со значением `"init"` — имя вашего `@ContractInit`-метода. Это служебный параметр, по которому диспетчер контракта понимает, какой именно метод вызвать (подробнее — в файле про написание контракта).

**Важно:** идентификатор созданного контракта (`contractId`) — это поле `id` из ответа этой транзакции. Сохраните его сразу:
```bash
CONTRACT_ID=$(jq -r '.id' create_result.json)
echo "Contract ID: $CONTRACT_ID"
```

## Шаг 4. Что происходит на стороне ноды
1. Нода получает транзакцию 103, скачивает образ по полю `image`, проверяет, что его хэш совпадает с `imageHash`.
2. Запускает образ как Docker-контейнер (у ноды для этого есть доступ к Docker-сокету — помните volume `/var/run/docker.sock` из `docker-compose.yml`?).
3. Контейнер подключается к ноде по gRPC и получает вызов метода, помеченного `@ContractInit`.
4. Изменения состояния, которые сделал `init()`, записываются в блокчейн.

## Шаг 5. Проверяем, что контракт задеплоился
Статус конкретной транзакции создания:
```bash
curl -s "$NODE_URL/contracts/status/$CONTRACT_ID" | jq
```
Ожидаемый ответ при успехе:
```json
{
  "sender": "3Nt9234ZD7DiXrMUfCvjiJEGE63TwxDMjQS",
  "senderPublicKey": "5Pk3KM7...",
  "txId": "5tGx8h...",
  "status": "Success",
  "code": null,
  "message": "Contract transaction successfully mined",
  "timestamp": 1727000000000
}
```
Если `status` — `Error` (некритическая, например бизнес-ошибка внутри `init()`) или `Failure` (системная ошибка) — в поле `message` и `code` будет причина.

Список всех контрактов на сети — включая ваш новый:
```bash
curl -s "$NODE_URL/contracts" | jq
```
Каждая запись содержит `contract_id`, `image`, `imageHash`, `version` и `active` (запущен контейнер сейчас или нет).

Если статус `Success`, а `active` — `true`, контракт готов принимать вызовы бизнес-методов — переходим к следующему файлу.

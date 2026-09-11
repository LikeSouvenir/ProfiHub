# Практика: сквозной цикл деплоя без единого клика мышью

Инструкция, по которой обычно учат этот блок, построена на ручных действиях: открыть Postman, вставить JSON, нажать Send, скопировать `id`, вставить его в следующий запрос — и так на каждый деплой заново. Это нормально для первого знакомства, но плохо масштабируется: на чемпионате, где контракт дорабатывается и передеплоивается десятки раз, каждая ручная операция — это точка, где легко забыть скопировать значение или взять устаревший `contractId`.

Ниже — тот же сценарий (создать контракт → задеплоить → вызвать метод → проверить результат), но одним воспроизводимым bash-скриптом поверх curl и jq.

## Подготовка
```bash
# .env — один раз заполняем данными из credentials.txt
export NODE_URL=http://localhost:6882
export SENDER=3Nt9234ZD7DiXrMUfCvjiJEGE63TwxDMjQS
export KEYPAIR_PASSWORD=gkCrlv9arT7EWNQUPdWqYg
export API_KEY=we
export IMAGE=registry.hub.docker.com/myuser/my-contract:1.0.0
```
```bash
sudo apt install jq -y   # если ещё не стоит — понадобится парсить JSON-ответы
```

## Вспомогательные функции
```bash
#!/bin/bash
set -euo pipefail
source .env

node_call() {
  curl -s -X "$1" "$NODE_URL$2" \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $API_KEY" \
    ${3:+-d "$3"}
}

wait_for_status() {
  local tx_id=$1
  for i in $(seq 1 20); do
    result=$(node_call GET "/contracts/status/$tx_id")
    status=$(echo "$result" | jq -r '.status // empty')
    if [ "$status" = "SUCCESS" ]; then
      echo "$result"; return 0
    elif [ "$status" = "ERROR" ] || [ "$status" = "FAILURE" ]; then
      echo "Ошибка: $result" >&2; return 1
    fi
    sleep 1
  done
  echo "Тайм-аут ожидания статуса транзакции $tx_id" >&2; return 1
}
```
`wait_for_status` решает частую проблему ручной работы: транзакция не появляется в блоке мгновенно, а Swagger UI не умеет сам "подождать и повторить" — приходится жать Refresh руками, гадая, сколько ждать.

## Шаг 1. Сборка и публикация образа
```bash
./gradlew clean build
docker build . --platform linux/amd64 -t "$IMAGE"
docker push "$IMAGE"

IMAGE_HASH=$(docker inspect "$IMAGE" | grep '"Id": "sha256' | awk -F'"Id": "sha256:|",' '{print $2}')
echo "Image hash: $IMAGE_HASH"
```

## Шаг 2. Деплой (103 CreateContract)
```bash
create_result=$(node_call POST /transactions/signAndBroadcast "{
  \"type\": 103, \"version\": 1,
  \"sender\": \"$SENDER\", \"password\": \"$KEYPAIR_PASSWORD\",
  \"image\": \"$IMAGE\", \"imageHash\": \"$IMAGE_HASH\",
  \"contractName\": \"my-contract\",
  \"params\": [{\"type\": \"string\", \"key\": \"action\", \"value\": \"init\"}],
  \"fee\": 100000000
}")

CONTRACT_ID=$(echo "$create_result" | jq -r '.id')
echo "Contract ID: $CONTRACT_ID"

wait_for_status "$CONTRACT_ID"
```

## Шаг 3. Вызов метода (104 CallContract)
```bash
call_result=$(node_call POST /transactions/signAndBroadcast "{
  \"type\": 104, \"version\": 2,
  \"sender\": \"$SENDER\", \"password\": \"$KEYPAIR_PASSWORD\",
  \"contractId\": \"$CONTRACT_ID\", \"contractVersion\": 1, \"fee\": 500000,
  \"params\": [
    {\"type\": \"string\", \"key\": \"action\", \"value\": \"someBusinessMethod\"},
    {\"type\": \"string\", \"key\": \"name\", \"value\": \"Test Org\"}
  ]
}")

CALL_ID=$(echo "$call_result" | jq -r '.id')
wait_for_status "$CALL_ID"
```

## Шаг 4. Проверка результата
```bash
echo "Итоговое состояние контракта:"
node_call GET "/contracts/$CONTRACT_ID" | jq

echo "Что изменил именно этот вызов:"
node_call GET "/contracts/executed-tx-for/$CALL_ID" | jq '.resultsMap // .results'
```

## Почему это оптимальнее ручного пути из инструкции
1. **Воспроизводимость.** Один и тот же скрипт можно запускать сколько угодно раз подряд без риска перепутать поля местами или вставить не тот `contractId` — это частая причина "почему у меня не работает" в командах, торопящихся к дедлайну.
2. **Автоматическое ожидание статуса** вместо ручного `docker ps` / Refresh в Swagger.
3. **Секреты не мелькают в интерфейсе** — они лежат в `.env`, который не коммитится в git (`.gitignore`), а не вводятся заново в каждое окно Postman.
4. **Скрипт можно подключить к CI** (например, GitHub Actions) — передеплой контракта при каждом пуше в ветку, без участия человека вообще.
5. **Один файл — вся история действий**: если что-то пошло не так, весь путь от сборки до вызова виден построчно, а не разбросан по вкладкам Postman.

Этот же паттерн (переменные окружения + функция-обёртка над curl + ожидание статуса) стоит использовать для любого нового смарт-контракта на чемпионате — меняются только `params` в теле 104-вызова под конкретные бизнес-методы вашего контракта.

# Запуск локальной сети нод

Для разработки и тестов не нужна связь с реальным mainnet/testnet — поднимается своя локальная приватная сеть из нескольких нод в Docker, полностью изолированная на вашей машине.

## Шаг 1. Описываем сеть в docker-compose.yml
Создайте директорию проекта, а в ней — файл `docker-compose.yml`. Актуальный пример для песочницы всегда можно взять из официальной документации: `doc.web3tech.ru/ru/latest/get-started/sandbox/docker-compose-sandbox.html`. Минимальная сеть из трёх нод выглядит так:

```yaml
version: '3'
services:
  node-0:
    image: web3techru/confident:v1.9.0
    ports:
      - "6862:6862"   # REST API
      - "6864:6864"   # gRPC для смарт-контрактов
      - "6865:6865"   # gRPC для клиентских запросов
    networks:
      - w3-network
    hostname: node-0
    container_name: node-0
    env_file:
      - ./env/node-0.env
    volumes:
      - ./configs/nodes/node-0/node.conf:/node/node.conf
      - ./configs/nodes/node-0/keystore.dat:/node/keystore.dat
      - node-0-data:/node/data
      - /var/run/docker.sock:/var/run/docker.sock   # нода сама поднимает контейнеры контрактов
    restart: always

  node-1:
    image: web3techru/confident:v1.9.0
    ports:
      - "6872:6862"
      - "6874:6864"
      - "6875:6865"
    networks:
      - w3-network
    hostname: node-1
    container_name: node-1
    env_file:
      - ./env/node-1.env
    volumes:
      - ./configs/nodes/node-1/node.conf:/node/node.conf
      - ./configs/nodes/node-1/keystore.dat:/node/keystore.dat
      - node-1-data:/node/data
      - /var/run/docker.sock:/var/run/docker.sock
    restart: always

  node-2:
    image: web3techru/confident:v1.9.0
    ports:
      - "6882:6862"
      - "6884:6864"
      - "6885:6865"
    networks:
      - w3-network
    hostname: node-2
    container_name: node-2
    env_file:
      - ./env/node-2.env
    volumes:
      - ./configs/nodes/node-2/node.conf:/node/node.conf
      - ./configs/nodes/node-2/keystore.dat:/node/keystore.dat
      - node-2-data:/node/data
      - /var/run/docker.sock:/var/run/docker.sock
    restart: always

networks:
  w3-network:
    driver: bridge
volumes:
  node-0-data:
  node-1-data:
  node-2-data:
```
Обратите внимание на проброс портов: слева — порт на вашей машине, справа — порт внутри контейнера. У всех нод один и тот же набор внутренних портов (`6862/6864/6865`), поэтому снаружи их разносят на `6862`, `6872`, `6882` и т.д., чтобы не было конфликта.

## Шаг 2. Генерируем конфигурацию нод
Вручную писать `node.conf`, ключи и `api-key-hash` для каждой ноды — долго и легко ошибиться. Для этого есть **config-manager** — служебный образ, который сам генерирует конфиги, ключевые пары и файл с учётными данными:

```bash
docker run --rm -ti -v $(pwd):/config-manager/output web3techru/config-manager:v1.9.0
```
Дождитесь сообщения об окончании развёртывания:
```
INFO [launcher] WE network environment is ready!
```
В результате в текущей папке появятся:
- `configs/nodes/node-0..N/` — `node.conf` и `keystore.dat` для каждой ноды;
- `env/node-0..N.env` — переменные окружения (пароль от keystore и т.п.);
- `credentials.txt` — сводка по всем нодам: адрес, публичный ключ, пароль от keystore, пароль от пары ключей и **API key**.

Посмотреть учётные данные:
```bash
cat credentials.txt
```
Пример содержимого на одну ноду:
```
node-0
blockchain address: 3NiVCCtz3AZuhayby2HZhqmoDnf6NkrzNWd
public key:         2Wh9VC8sTPGBvmth6JtsEFVJNkGyd38Uf419vBH5hhNj
keystore password:  R7oHM1yQ6yS491L1g2Ka8A
keypair password:   wDr_N-S8p7BMx-YP5O1JwQ
API key:            we
```
Эти значения понадобятся почти на каждом следующем шаге: `blockchain address` и `keypair password` — чтобы подписывать транзакции от имени ноды, `API key` — чтобы обращаться к приватным методам REST API.

## Шаг 3. Поднимаем ноды
```bash
docker-compose up -d
```
Флаг `-d` запускает контейнеры в фоне. Проверить, что всё поднялось:
```bash
docker ps
docker logs -f node-0     # если нода не стартует — смотрим логи именно тут
```

## Шаг 4. Проверяем через Swagger UI
Каждая нода поднимает автоматически сгенерированный Swagger по адресу вида `http://localhost:<REST-порт>` (например, `http://localhost:6882` для `node-2`).

1. Откройте адрес в браузере.
2. В разделе **Schemes** выберите `HTTP`.
3. Нажмите зелёную кнопку **Authorize** справа.
4. В поле **Value** вставьте `API key` соответствующей ноды из `credentials.txt` и нажмите **Authorize**.

После этого в Swagger UI можно выполнять запросы к приватным методам прямо из браузера — удобно для первого знакомства с API, но для регулярной работы удобнее curl-скрипты (см. следующий файл).

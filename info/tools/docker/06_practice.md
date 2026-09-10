# Практика: от Dockerfile до Docker Hub и Compose

Разберём сквозной сценарий: берём простое Node.js-приложение с базой данных PostgreSQL, пишем для него Dockerfile, собираем и запускаем контейнер вручную, публикуем образ на Docker Hub, а затем описываем весь стек через Docker Compose.

## Структура проекта
```
my-app/
├── server.js
├── package.json
├── Dockerfile
├── .dockerignore
└── docker-compose.yml
```

## server.js — простое приложение
```js
const express = require('express');
const app = express();
const PORT = process.env.PORT || 3000;

app.get('/', (req, res) => {
    res.send('Привет из Docker-контейнера!');
});

app.listen(PORT, () => {
    console.log(`Сервер запущен на порту ${PORT}`);
});
```

## package.json
```json
{
  "name": "my-app",
  "version": "1.0.0",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.19.2"
  }
}
```

## Шаг 1. Пишем Dockerfile
```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 3000

CMD ["npm", "start"]
```

## .dockerignore
```
node_modules
npm-debug.log
.git
.env
```

## Шаг 2. Собираем образ
```bash
docker build -t myusername/my-app:1.0 .
```

## Шаг 3. Запускаем контейнер вручную и проверяем
```bash
docker run -d -p 3000:3000 --name my-app-container myusername/my-app:1.0

# Проверяем, что контейнер запущен
docker ps

# Смотрим логи
docker logs my-app-container
```
Открываем `http://localhost:3000` в браузере — должна отобразиться строка "Привет из Docker-контейнера!".

```bash
# Останавливаем и удаляем тестовый контейнер, он больше не нужен
docker stop my-app-container
docker rm my-app-container
```

## Шаг 4. Публикуем образ на Docker Hub
```bash
docker login

docker push myusername/my-app:1.0
```
Теперь образ доступен по адресу `hub.docker.com/r/myusername/my-app`, и любой человек может скачать точно такое же окружение:
```bash
docker pull myusername/my-app:1.0
docker run -p 3000:3000 myusername/my-app:1.0
```

## Шаг 5. Добавляем базу данных через Docker Compose
Дополняем `server.js` подключением к PostgreSQL (в реальном проекте — через библиотеку `pg`), а инфраструктуру описываем в `docker-compose.yml`:

```yaml
services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - PORT=3000
      - DATABASE_URL=postgres://appuser:apppass@db:5432/appdb
    depends_on:
      - db

  db:
    image: postgres:16-alpine
    environment:
      - POSTGRES_USER=appuser
      - POSTGRES_PASSWORD=apppass
      - POSTGRES_DB=appdb
    volumes:
      - app_db_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

volumes:
  app_db_data:
```

## Шаг 6. Запуск всего стека одной командой
```bash
docker compose up -d --build
```
- `--build` — принудительно пересобрать образ `app` из Dockerfile перед запуском.
- `-d` — запуск в фоновом режиме.

Проверка:
```bash
docker compose ps            # оба сервиса (app и db) должны быть в статусе "running"
docker compose logs -f app   # смотрим логи приложения в реальном времени
```

## Шаг 7. Остановка и очистка
```bash
docker compose down       # остановить и удалить контейнеры (том db_data сохранится)
docker compose down -v    # остановить, удалить контейнеры И том с данными базы (полная очистка)
```

## Разбор решения
1. **Dockerfile** копирует сначала только `package*.json` и ставит зависимости, а уже потом копирует весь остальной код — это сохраняет кэш слоя с `npm install` при последующих изменениях самого кода приложения, ускоряя пересборку.
2. **Ручной запуск через `docker run`** с флагами `-d` (фон), `-p` (проброс порта) и `--name` (удобное имя) — стандартный способ быстро проверить единичный контейнер перед тем, как переходить к более сложной оркестровке.
3. **Публикация на Docker Hub** через `docker login` + `docker push` делает образ доступным для скачивания на любой другой машине — сервере, ноутбуке коллеги, облачной инфраструктуре — без необходимости заново настраивать окружение вручную.
4. **`docker-compose.yml`** описывает сразу оба сервиса (`app` и `db`) как единую систему: они автоматически оказываются в общей сети и видят друг друга по именам сервисов (`db` в `DATABASE_URL` — это буквально имя сервиса, а не IP-адрес).
5. **Именованный том `app_db_data`** гарантирует, что данные PostgreSQL переживут перезапуск и даже полное удаление контейнера `db` — до тех пор, пока не выполнена команда `docker compose down -v`, которая удаляет и сами тома.

Такой сквозной путь — от Dockerfile для одного приложения до полноценного мультиконтейнерного стека через Compose — покрывает практически весь стандартный рабочий процесс контейнеризации реального проекта.

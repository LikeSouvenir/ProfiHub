# Docker Compose

## Проблема, которую решает Compose
Реальное приложение редко состоит из одного контейнера — обычно нужны сразу: сам сервер (backend), база данных, кэш (Redis), а иногда и фронтенд отдельным контейнером. Запускать каждый из них вручную через длинные команды `docker run` с кучей флагов — неудобно и легко ошибиться.

**Docker Compose** — инструмент для описания и запуска **нескольких** связанных контейнеров одной конфигурацией в файле `docker-compose.yml`, и управления ими как единым целым.

## Базовая структура docker-compose.yml
```yaml
services:
  backend:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgres://user:password@db:5432/mydb
    depends_on:
      - db

  db:
    image: postgres:16-alpine
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=mydb
    volumes:
      - db_data:/var/lib/postgresql/data

volumes:
  db_data:
```

## Разбор ключевых полей

### services — список контейнеров приложения
Каждый сервис описывает один контейнер: `backend` и `db` в примере выше — фактически два будущих контейнера, объединённых в одну систему.

### build vs image
```yaml
backend:
  build: .          # собрать образ из Dockerfile в текущей папке
db:
  image: postgres:16-alpine   # взять готовый образ из Docker Hub, ничего не собирая
```

### ports — проброс портов
```yaml
ports:
  - "3000:3000"   # хост:контейнер, точно как флаг -p у docker run
```

### environment — переменные окружения
```yaml
environment:
  - NODE_ENV=production
  - DATABASE_URL=postgres://user:password@db:5432/mydb
```
Обратите внимание: в качестве хоста базы данных указано `db` — имя сервиса из этого же `docker-compose.yml`. Compose автоматически создаёт для всех сервисов общую сеть, где они видны друг другу по имени сервиса — точно так же, как в теме про пользовательские Docker-сети.

### depends_on — порядок запуска
```yaml
depends_on:
  - db
```
Указывает Compose запускать `db` раньше, чем `backend`. *Важная деталь: `depends_on` гарантирует только порядок **запуска контейнера**, но не то, что база данных внутри уже полностью готова принимать подключения — для более надёжной проверки на практике часто добавляют `healthcheck`.*

### volumes — постоянное хранилище
```yaml
volumes:
  - db_data:/var/lib/postgresql/data
```
Данные базы сохранятся даже после удаления и пересоздания контейнера `db`, так как физически хранятся в томе `db_data`, а не внутри самого контейнера.

## Основные команды Docker Compose
```bash
docker compose up             # собрать (если нужно) и запустить ВСЕ сервисы
docker compose up -d          # то же самое, но в фоновом режиме
docker compose up --build     # принудительно пересобрать образы перед запуском

docker compose down           # остановить и удалить все контейнеры, созданные Compose
docker compose down -v        # то же самое + удалить связанные тома (данные будут потеряны!)

docker compose ps             # список контейнеров текущего проекта Compose и их статус
docker compose logs           # логи всех сервисов сразу
docker compose logs -f backend # логи конкретного сервиса в реальном времени

docker compose exec backend bash  # зайти внутрь работающего контейнера сервиса backend
docker compose restart backend    # перезапустить один конкретный сервис
```

## Пример с тремя сервисами: backend + frontend + база данных
```yaml
services:
  frontend:
    build: ./frontend
    ports:
      - "5173:5173"
    depends_on:
      - backend

  backend:
    build: ./backend
    ports:
      - "4000:4000"
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/appdb
    depends_on:
      - db

  db:
    image: postgres:16-alpine
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
      - POSTGRES_DB=appdb
    volumes:
      - pg_data:/var/lib/postgresql/data

volumes:
  pg_data:
```
Запуск всей системы (три контейнера, связанных сетью, с одним постоянным хранилищем для базы) — одна команда:
```bash
docker compose up -d
```

## Compose для разработки: bind mount с "живой" перезагрузкой
```yaml
services:
  backend:
    build: .
    ports:
      - "3000:3000"
    volumes:
      - .:/app                # монтируем текущую папку хоста внутрь контейнера
      - /app/node_modules      # но node_modules берём из контейнера, а не с хоста
    command: npm run dev
```
Такая конфигурация — стандартный приём для разработки: изменения кода на компьютере сразу отражаются внутри контейнера без пересборки образа, а зависимости при этом остаются теми, что были установлены именно внутри контейнера (актуально, если хост и контейнер используют разные ОС).

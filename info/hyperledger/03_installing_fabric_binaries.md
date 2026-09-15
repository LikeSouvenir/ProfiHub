# 03. Установка Hyperledger Fabric

## 3.1. Что именно мы сейчас установим
Одной командой будут скачаны сразу три вещи:
1. **Репозиторий `fabric-samples`** — готовые примеры, скрипты, тестовая сеть.
2. **Docker-образы** — готовые "контейнерные" версии всех программ Fabric (peer, orderer, CA и т.д.), их не нужно компилировать самостоятельно.
3. **Бинарные файлы (bin)** — консольные утилиты `peer`, `orderer`, `cryptogen`, `configtxgen`, которые запускаются напрямую, не в контейнере.

## 3.2. Команда установки
Выполняется **внутри терминала Ubuntu**, в папке, где будет жить весь проект:
```bash
mkdir -p ~/projects/hl-project
cd ~/projects/hl-project

curl -sSLO https://raw.githubusercontent.com/hyperledger/fabric/main/scripts/install-fabric.sh
chmod +x install-fabric.sh
./install-fabric.sh
```
Скрипт скачивает довольно много данных (несколько гигабайт Docker-образов) — на слабом интернет-соединении процесс может занять 10-20 минут. Дождитесь полного завершения без ошибок.

### ✅ Проверка
```bash
docker images | grep hyperledger
```
Должен вывестись список образов вида `hyperledger/fabric-peer`, `hyperledger/fabric-orderer`, `hyperledger/fabric-ca` и другие.

```bash
ls fabric-samples/bin
```
Должны быть видны файлы `peer`, `orderer`, `cryptogen`, `configtxgen`, `configtxlator`, `discover`, `osnadmin`, `fabric-ca-client`.

## 3.3. Полный список используемых Docker-образов и зачем каждый нужен
| Образ | Роль |
|---|---|
| `hyperledger/fabric-peer` | Контейнер peer-узла — исполняет chaincode, хранит реестр |
| `hyperledger/fabric-orderer` | Контейнер orderer-узла — упорядочивает транзакции в блоки |
| `hyperledger/fabric-ca` | Центр сертификации — выдаёт сертификаты участникам |
| `hyperledger/fabric-ccenv` | Окружение для сборки chaincode перед запуском (в старом режиме "chaincode-as-a-server" не всегда используется, но участвует в part пайплайна) |
| `hyperledger/fabric-baseos` | Базовая ОС-прослойка для контейнеров chaincode |
| `hyperledger/fabric-nodeenv` | Готовая среда Node.js специально для запуска chaincode на JS внутри контейнера |
| `couchdb` | База данных для хранения World State с поддержкой сложных JSON-запросов |
| `busybox` | Минимальный служебный образ, используется вспомогательными скриптами |

## 3.4. Структура fabric-samples — что реально нужно, а что можно игнорировать
```
fabric-samples/
├── bin/                 # ← НУЖНО: консольные утилиты Fabric
├── builders/            # ← НУЖНО: скрипты сборки chaincode
├── config/               # ← НУЖНО: базовые конфиги (используются test-network)
├── test-network/         # ← НУЖНО: главная рабочая папка — сеть из 2 организаций
│   ├── organizations/    #   Сертификаты и MSP-данные — появятся ПОСЛЕ первого запуска сети
│   ├── addOrg3/          #   Готовый скрипт добавления 3-й организации
│   ├── compose/          #   Docker Compose файлы, из которых собирается сеть
│   └── network.sh        #   Главный скрипт управления сетью
├── token-erc-20/         # Пример готовой реализации токена (пригодится в теме про токены)
├── asset-transfer-basic/ # Пример готового chaincode на разных языках — удобно как справочник
├── asset-transfer-*/     # Другие готовые примеры — можно не трогать
├── commercial-paper/     # Более сложный сквозной пример — можно не трогать
└── ... (ещё десяток папок с другими примерами — для этого курса не нужны)
```
**Правило простое:** если задача не требует явно — работаем только с `bin/`, `builders/`, `config/` и `test-network/`. Остальные папки — справочный материал на будущее, их можно спокойно удалить, если важна экономия места на диске:
```bash
cd fabric-samples
rm -rf asset-transfer-* commercial-paper* fabcar* off_chain_data high-throughput auction*
```

## 3.5. Переменные окружения PATH (чтобы не писать длинные пути)
```bash
echo 'export PATH=$PATH:$HOME/projects/hl-project/fabric-samples/bin' >> ~/.bashrc
echo 'export FABRIC_CFG_PATH=$HOME/projects/hl-project/fabric-samples/config' >> ~/.bashrc
source ~/.bashrc
```
После этого консольные утилиты (`peer`, `cryptogen` и т.д.) можно вызывать из любой папки, не прописывая полный путь к `bin/` каждый раз.

### ✅ Проверка
```bash
peer version
cryptogen version
```

## 3.6. Версии, которые ставятся по умолчанию (актуально на момент написания курса)
| Компонент | Версия |
|---|---|
| fabric-peer / fabric-orderer | 2.5.9 |
| fabric-ca | 1.5.12 |
| fabric-ccenv / fabric-baseos | 2.5.9 |
| fabric-nodeenv | 2.5.7 |
| couchdb | 3.3.3 |

*Версии могут обновляться со временем — актуальный список всегда можно посмотреть командой `docker images` после установки. Смешивать сильно разные мажорные версии peer/orderer/CA не стоит — рекомендуется, чтобы все компоненты сети были из одной "линейки" 2.5.x.*

## Итог
После этого конспекта на диске лежит: рабочая копия `fabric-samples`, все нужные Docker-образы скачаны локально, бинарники добавлены в `PATH`. Сеть ещё **не запущена** — этим займётся следующий конспект.

# 11. Настройка и деплой смарт-контракта

## 11.1. Быстрый способ — готовый скрипт deployCC
Подходит для 95% учебных и небольших сценариев:
```bash
cd fabric-samples/test-network

./network.sh deployCC \
  -ccn Cars \
  -ccl javascript \
  -ccp ../my-project/chaincode-javascript \
  -c blockchain2024 \
  -cci InitLedger
```

### Полная таблица флагов deployCC
| Флаг | Значение | Обязателен? |
|---|---|---|
| `-ccn` | Имя chaincode | Да |
| `-ccl` | Язык: `javascript`, `typescript`, `go`, `java` | Да |
| `-ccp` | Путь к папке с кодом | Да |
| `-c` | Имя канала | Да |
| `-cci` | Функция-инициализатор (вызывается один раз сразу после деплоя) | Нет, но рекомендуется |
| `-ccv` | Версия chaincode | Нет (по умолчанию `1.0`) |
| `-ccs` | Sequence (счётчик коммитов) | Нет (по умолчанию `1`) |
| `-cccg` | Путь к файлу конфигурации endorsement-политики | Нет |
| `-ccep` | Endorsement policy текстом прямо в команде | Нет |

### Перед деплоем зависимости обязательно должны быть установлены
```bash
cd ../my-project/chaincode-javascript && npm install && cd ../../test-network
```
Пропуск этого шага — самая частая причина падения `deployCC` на этапе установки на peer.

## 11.2. Что происходит "внутри" deployCC — полный ручной путь
Понимание ручного пути нужно, чтобы уметь диагностировать проблемы и настраивать нестандартные сценарии (кастомная endorsement policy, приватные коллекции).

```bash
export PATH=${PWD}/../bin:$PATH
export FABRIC_CFG_PATH=${PWD}/../config

# Устанавливаем переменные окружения для организации Org1 (готовый скрипт test-network)
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=localhost:7051

# Шаг 1: Package — упаковать код в архив
peer lifecycle chaincode package cars.tar.gz \
  --path ../my-project/chaincode-javascript \
  --lang node \
  --label cars_1.0

# Шаг 2: Install — установить пакет НА PEER ORG1
peer lifecycle chaincode install cars.tar.gz

# Получить Package ID (понадобится для approve)
peer lifecycle chaincode queryinstalled
# Пример вывода: Package ID: cars_1.0:a1b2c3..., Label: cars_1.0

# Шаг 3: Approve — Org1 одобряет определение chaincode
peer lifecycle chaincode approveformyorg \
  -o localhost:7050 --tls --cafile "$ORDERER_CA" \
  --channelID blockchain2024 --name Cars --version 1.0 \
  --package-id cars_1.0:a1b2c3... --sequence 1 --init-required
```

Затем **то же самое** (Install + Approve) повторяется от лица **Org2** — с переключением переменных окружения на `Org2MSP` и её пути к MSP:
```bash
export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_MSPCONFIGPATH=${PWD}/organizations/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
export CORE_PEER_ADDRESS=localhost:9051
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt

peer lifecycle chaincode install cars.tar.gz
peer lifecycle chaincode approveformyorg \
  -o localhost:7050 --tls --cafile "$ORDERER_CA" \
  --channelID blockchain2024 --name Cars --version 1.0 \
  --package-id cars_1.0:a1b2c3... --sequence 1 --init-required
```

```bash
# Проверка готовности к commit — покажет, каких организаций ещё не хватает
peer lifecycle chaincode checkcommitreadiness \
  --channelID blockchain2024 --name Cars --version 1.0 --sequence 1 --init-required \
  -o localhost:7050 --tls --cafile "$ORDERER_CA"

# Шаг 4: Commit — фиксация определения в канале (может выполнить любой admin, когда кворум набран)
peer lifecycle chaincode commit \
  -o localhost:7050 --tls --cafile "$ORDERER_CA" \
  --channelID blockchain2024 --name Cars --version 1.0 --sequence 1 --init-required \
  --peerAddresses localhost:7051 --tlsRootCertFiles "$PEER0_ORG1_CA" \
  --peerAddresses localhost:9051 --tlsRootCertFiles "$PEER0_ORG2_CA"

# Шаг 5: Invoke инициализатора
peer chaincode invoke -o localhost:7050 --tls --cafile "$ORDERER_CA" \
  -C blockchain2024 -n Cars --isInit \
  --peerAddresses localhost:7051 --tlsRootCertFiles "$PEER0_ORG1_CA" \
  --peerAddresses localhost:9051 --tlsRootCertFiles "$PEER0_ORG2_CA" \
  -c '{"Args":["InitLedger"]}'
```

**Хорошая новость: `./network.sh deployCC` выполняет буквально всё это одной командой** — ручной путь нужен только для нестандартных случаев или диагностики.

## 11.3. Настройка endorsement policy (кто должен одобрить транзакцию)
```bash
./network.sh deployCC -ccn Cars -ccl javascript -ccp ../my-project/chaincode-javascript \
  -c blockchain2024 -cci InitLedger \
  -ccep "OR('Org1MSP.member','Org2MSP.member')"
```
| Политика | Значение |
|---|---|
| `OR('Org1MSP.member','Org2MSP.member')` | Достаточно одобрения ЛЮБОЙ из двух организаций |
| `AND('Org1MSP.member','Org2MSP.member')` | Требуется одобрение ОБЕИХ организаций |
| `OutOf(2, 'Org1MSP.member','Org2MSP.member','Org3MSP.member')` | Достаточно любых 2 из 3 организаций |

## 11.4. Приватные коллекции данных (Private Data Collections)
Иногда часть данных должна быть видна не всем участникам канала, а только конкретным организациям (например, закупочная цена видна поставщику и покупателю, но не остальным участникам сети).

**collections_config.json**
```json
[
    {
        "name": "carsPriceCollection",
        "policy": "OR('Org1MSP.member','Org2MSP.member')",
        "requiredPeerCount": 1,
        "maxPeerCount": 2,
        "blockToLive": 0,
        "memberOnlyRead": true
    }
]
```
```bash
./network.sh deployCC -ccn Cars -ccl javascript -ccp ../my-project/chaincode-javascript \
  -c blockchain2024 -cci InitLedger \
  -cccg ../my-project/chaincode-javascript/collections_config.json
```
Внутри chaincode работа с приватными данными идёт через отдельные методы:
```js
await ctx.stub.putPrivateData('carsPriceCollection', id, Buffer.from(JSON.stringify({ price: 15000 })));
const priceData = await ctx.stub.getPrivateData('carsPriceCollection', id);
```
Клиент передаёт такие данные через `transient` (см. конспект 05, метод `transaction.setTransient(...)`) — они **не попадают** в публичный блок и не видны при обычном чтении реестра сторонними организациями.

## 11.5. Обновление уже развёрнутого chaincode
При изменении кода **обязательно** увеличивается `sequence` (Fabric отслеживает историю изменений определения именно по нему):
```bash
./network.sh deployCC -ccn Cars -ccl javascript -ccp ../my-project/chaincode-javascript \
  -c blockchain2024 -ccv 2.0 -ccs 2
```
- `-ccv 2.0` — произвольная метка версии (для людей — может быть любой строкой).
- `-ccs 2` — обязательное увеличение sequence на 1 относительно предыдущего значения (проверяется самим Fabric, попытка закоммитить с тем же sequence будет отклонена).

*Если инициализатор (`InitLedger`) уже был вызван при первом деплое — при обновлении версии повторный `-cci` обычно не указывают, чтобы не перезаписать уже накопленные реальные данные заново начальными значениями.*

## 11.6. Проверка развёрнутого chaincode
```bash
peer lifecycle chaincode querycommitted --channelID blockchain2024 --name Cars
```
Покажет текущую закоммиченную версию, sequence и endorsement policy — полезно для диагностики рассинхронизации между организациями.

## 11.7. Частые ошибки этапа деплоя
| Ошибка | Причина | Решение |
|---|---|---|
| `chaincode install failed: failed to normalize chaincode path` | Не установлены зависимости (`npm install`/`go mod tidy`/`gradle build`) внутри папки chaincode | Установить зависимости ДО деплоя |
| `chaincode definition not agreed to by this org` | Одна из организаций не выполнила `approveformyorg` | Повторить approve для недостающей организации |
| `ProposalResponsePayloads do not match` | Разные организации выполняют разную версию/логику chaincode (рассинхронизация кода между `install` на разных peer'ах) | Убедиться, что на всех peer'ах установлен один и тот же архив пакета |
| `chaincode already successfully defined with sequence N` | Забыт инкремент `-ccs` при обновлении | Указать `-ccs` на 1 больше предыдущего значения |

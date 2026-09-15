# 14. Итоговая практика: полный проект от нуля

Этот конспект — пошаговый чек-лист, объединяющий все предыдущие 13 конспектов в один воспроизводимый путь. Каждый пункт можно отмечать как выполненный и сверяться с конкретным конспектом, если что-то непонятно.

## Чек-лист всего проекта
- [ ] **1.** Прочитано введение в технологию (конспект 01)
- [ ] **2.** Установлено окружение: WSL, Docker, Node.js, jq (конспект 02)
- [ ] **3.** Установлен Fabric: бинарники и образы (конспект 03)
- [ ] **4.** Поднята сеть с каналом `blockchain2024`, CA и CouchDB (конспект 04)
- [ ] **5.** Написан chaincode `Cars` (выбрать один язык — JS/Go/Java, конспекты 08–10)
- [ ] **6.** Chaincode развёрнут в сети (конспект 11)
- [ ] **7.** Enroll администратора и хотя бы одного пользователя (конспект 06)
- [ ] **8.** Написан и запущен Express API (конспект 12)
- [ ] **9.** Написан и запущен React-фронтенд (конспект 13)
- [ ] **10.** Полный сценарий проверен end-to-end через интерфейс

## Шаг за шагом — команды в порядке выполнения

### 1. Поднятие сети
```bash
cd ~/projects/hl-project/fabric-samples/test-network
./network.sh up createChannel -c blockchain2024 -ca -s couchdb
```
✅ Проверка: `docker ps` показывает peer0.org1, peer0.org2, orderer, ca_org1, ca_org2, couchdb0, couchdb1.

### 2. Подготовка и деплой chaincode (вариант на JavaScript)
```bash
cd ../my-project/chaincode-javascript
npm install
cd ../../fabric-samples/test-network

./network.sh deployCC -ccn Cars -ccl javascript \
  -ccp ../my-project/chaincode-javascript \
  -c blockchain2024 -cci InitLedger
```
✅ Проверка:
```bash
peer lifecycle chaincode querycommitted --channelID blockchain2024 --name Cars
```

### 3. Настройка API
```bash
cd ../my-project/application-javascript
npm install
```
Создать `.env` (см. конспект 12), убедиться, что `CHANNEL_NAME`, `CHAINCODE_NAME`, `CONTRACT_NAME` совпадают с тем, что использовалось при деплое.
```bash
npm run dev
```
✅ Проверка: в консоли видно `API запущен: http://localhost:7000`.

### 4. Регистрация первых пользователей через API
```bash
curl -X POST http://localhost:7000/enrollAdmin -H "Content-Type: application/json" -d '{"organization":"org1"}'
curl -X POST http://localhost:7000/enrollUser -H "Content-Type: application/json" -d '{"organization":"org1","userID":"user1"}'
```
✅ Проверка: оба запроса вернули `{"success": true, ...}`, в папке `application-javascript/wallet/org1` появились файлы `admin.id` и `user1.id`.

### 5. Проверка chaincode напрямую через API (до фронтенда)
```bash
curl -X POST http://localhost:7000/getAllCars -H "Content-Type: application/json" -d '{"organization":"org1","userID":"user1"}'
```
✅ Ожидаемый результат: массив с `car1` и `car2` — теми, что были добавлены функцией `InitLedger`.

```bash
curl -X POST http://localhost:7000/addCar -H "Content-Type: application/json" \
  -d '{"organization":"org1","userID":"user1","id":"car3","color":"blue","brand":"Toyota","owner":"user1"}'
```
✅ Проверка: повторный `getAllCars` возвращает уже 3 машины.

### 6. Запуск фронтенда
```bash
cd ../../src   # или отдельная папка hl-frontend, см. конспект 13
npm install
npm run dev
```
✅ Проверка: открыть `http://localhost:5173`, увидеть список машин, форму регистрации пользователя и форму добавления машины.

### 7. Полный пользовательский сценарий через интерфейс
1. Зарегистрировать нового пользователя через форму (`EnrollUserForm`).
2. Добавить новую машину через форму (`AddCarForm`) — убедиться, что список обновился без перезагрузки страницы.
3. Открыть DevTools → Network, убедиться, что запросы идут на `localhost:7000`, а не напрямую к каким-либо портам Fabric.

## Что делать, если сеть нужно пересобрать с нуля
```bash
cd fabric-samples/test-network
./network.sh down

rm -rf ../../my-project/application-javascript/wallet   # старые личности станут недействительны

./network.sh up createChannel -c blockchain2024 -ca -s couchdb
./network.sh deployCC -ccn Cars -ccl javascript -ccp ../my-project/chaincode-javascript -c blockchain2024 -cci InitLedger
```
После этого обязательно заново выполнить `/enrollAdmin` и `/enrollUser` — см. конспект 06, раздел про пересоздание сети.

## Идеи для усложнения проекта (после базовой сдачи)
| Усложнение | Какой конспект использовать |
|---|---|
| Добавить третью организацию с ограниченными правами | 04 (addOrg3) + 06 (роли/MSP) |
| Ограничить удаление машин только пользователям с атрибутом `role=manager` | 05 (ABAC) + 08/09/10 (проверка в chaincode) |
| Скрыть цену машины от посторонних организаций | 11 (Private Data Collections) |
| Показывать обновления списка в реальном времени без обновления страницы | 12 (SSE-эндпоинт) + 13 (EventSource) |
| Переписать endorsement policy так, чтобы хватало одобрения любой ОДНОЙ организации | 07 + 11 (`-ccep "OR(...)"`) |
| Реализовать тот же контракт на Go или Java и сравнить производительность | 09, 10 |

## Итоговая проверка готовности (финальный чек-лист перед сдачей)
- [ ] Сеть поднимается одной командой без ошибок
- [ ] Chaincode деплоится одной командой без ручных правок кода под конкретную машину
- [ ] Есть минимум 2 пользователя из минимум 2 организаций, оба могут выполнять операции
- [ ] API возвращает понятные сообщения об ошибках (а не "Internal Server Error" без объяснений)
- [ ] Фронтенд отображает актуальные данные из блокчейна, а не статичные заглушки
- [ ] Есть хотя бы одна операция записи (submitTransaction) и одна операция чтения (evaluateTransaction), обе видны и работают через интерфейс

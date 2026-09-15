# 09. Написание смарт-контракта на Go

## 9.1. Зачем вариант на Go
Go — язык, на котором написан сам Hyperledger Fabric, и исторически самый производительный вариант для chaincode. Если для JS-разработчика Go может показаться непривычным — ниже разобран построчно тот же самый пример `Cars`, что и в конспекте про JS, чтобы сравнение было прямым.

## 9.2. Подготовка проекта
```bash
mkdir -p ~/projects/hl-project/my-project/chaincode-go
cd ~/projects/hl-project/my-project/chaincode-go

go mod init cars_chaincode
go get github.com/hyperledger/fabric-contract-api-go/v2/contractapi
```

## 9.3. Структура проекта
```
chaincode-go/
├── cars.go          # Основной код контракта
├── main.go          # Точка входа — запускает chaincode
├── go.mod
└── go.sum
```

## 9.4. Готовое полное решение: контракт Cars на Go
```go
// cars.go
package main

import (
    "encoding/json"
    "fmt"

    "github.com/hyperledger/fabric-contract-api-go/v2/contractapi"
)

// CarsContract — основная структура контракта (аналог класса Cars в JS)
type CarsContract struct {
    contractapi.Contract
}

// Car — структура одной записи (аналог объекта car в JS)
type Car struct {
    DocType string `json:"docType"`
    ID      string `json:"ID"`
    Color   string `json:"Color"`
    Brand   string `json:"Brand"`
    Owner   string `json:"Owner"`
}

// InitLedger — заполнение реестра начальными данными
func (c *CarsContract) InitLedger(ctx contractapi.TransactionContextInterface) error {
    cars := []Car{
        {DocType: "car", ID: "car1", Color: "red", Brand: "BMW", Owner: "admin"},
        {DocType: "car", ID: "car2", Color: "black", Brand: "Mercedes", Owner: "admin"},
    }

    for _, car := range cars {
        carJSON, err := json.Marshal(car)
        if err != nil {
            return err
        }

        err = ctx.GetStub().PutState(car.ID, carJSON)
        if err != nil {
            return fmt.Errorf("не удалось записать машину %s: %v", car.ID, err)
        }
    }

    return nil
}

// AddCar — создание новой записи
func (c *CarsContract) AddCar(ctx contractapi.TransactionContextInterface, id string, color string, brand string, owner string) error {
    exists, err := c.CarExists(ctx, id)
    if err != nil {
        return err
    }
    if exists {
        return fmt.Errorf("машина %s уже существует", id)
    }

    car := Car{DocType: "car", ID: id, Color: color, Brand: brand, Owner: owner}
    carJSON, err := json.Marshal(car)
    if err != nil {
        return err
    }

    return ctx.GetStub().PutState(id, carJSON)
}

// GetCar — чтение записи по id
func (c *CarsContract) GetCar(ctx contractapi.TransactionContextInterface, id string) (*Car, error) {
    carJSON, err := ctx.GetStub().GetState(id)
    if err != nil {
        return nil, fmt.Errorf("ошибка чтения из реестра: %v", err)
    }
    if carJSON == nil {
        return nil, fmt.Errorf("машины %s не существует", id)
    }

    var car Car
    err = json.Unmarshal(carJSON, &car)
    if err != nil {
        return nil, err
    }

    return &car, nil
}

// TransferCar — смена владельца
func (c *CarsContract) TransferCar(ctx contractapi.TransactionContextInterface, id string, newOwner string) (string, error) {
    car, err := c.GetCar(ctx, id)
    if err != nil {
        return "", err
    }

    oldOwner := car.Owner
    car.Owner = newOwner

    carJSON, err := json.Marshal(car)
    if err != nil {
        return "", err
    }

    err = ctx.GetStub().PutState(id, carJSON)
    if err != nil {
        return "", err
    }

    return oldOwner, nil
}

// DeleteCar — удаление записи
func (c *CarsContract) DeleteCar(ctx contractapi.TransactionContextInterface, id string) error {
    exists, err := c.CarExists(ctx, id)
    if err != nil {
        return err
    }
    if !exists {
        return fmt.Errorf("машины %s не существует", id)
    }

    return ctx.GetStub().DelState(id)
}

// CarExists — вспомогательная проверка существования
func (c *CarsContract) CarExists(ctx contractapi.TransactionContextInterface, id string) (bool, error) {
    carJSON, err := ctx.GetStub().GetState(id)
    if err != nil {
        return false, fmt.Errorf("ошибка чтения из реестра: %v", err)
    }
    return carJSON != nil, nil
}

// GetAllCars — перебор всех записей с фильтрацией по docType
func (c *CarsContract) GetAllCars(ctx contractapi.TransactionContextInterface) ([]*Car, error) {
    resultsIterator, err := ctx.GetStub().GetStateByRange("", "")
    if err != nil {
        return nil, err
    }
    defer resultsIterator.Close()

    var cars []*Car
    for resultsIterator.HasNext() {
        queryResponse, err := resultsIterator.Next()
        if err != nil {
            return nil, err
        }

        var car Car
        err = json.Unmarshal(queryResponse.Value, &car)
        if err != nil {
            continue // пропускаем записи, которые не парсятся как Car (другой docType)
        }

        if car.DocType == "car" {
            cars = append(cars, &car)
        }
    }

    return cars, nil
}
```

## 9.5. Проверка прав доступа (MSP) в Go
```go
import "github.com/hyperledger/fabric-chaincode-go/v2/pkg/cid"

func (c *CarsContract) DeleteCar(ctx contractapi.TransactionContextInterface, id string) error {
    mspID, err := cid.GetMSPID(ctx.GetStub())
    if err != nil {
        return err
    }
    if mspID == "Org3MSP" {
        return fmt.Errorf("организации Org3 запрещено удалять машины")
    }

    // ... остальная логика удаления
    return nil
}
```

## 9.6. main.go — точка входа
```go
// main.go
package main

import (
    "log"

    "github.com/hyperledger/fabric-contract-api-go/v2/contractapi"
)

func main() {
    carsChaincode, err := contractapi.NewChaincode(&CarsContract{})
    if err != nil {
        log.Panicf("Ошибка создания chaincode Cars: %v", err)
    }

    if err := carsChaincode.Start(); err != nil {
        log.Panicf("Ошибка запуска chaincode Cars: %v", err)
    }
}
```

## 9.7. Сборка и проверка перед деплоем
```bash
cd ~/projects/hl-project/my-project/chaincode-go
go mod tidy       # подтягивает и фиксирует все зависимости в go.sum
go build .        # проверка, что код компилируется без ошибок — ДО попытки деплоя
```
*Важное практическое правило: если `go build` не проходит локально — деплой в сеть тем более не сработает. Всегда сначала добиваться чистой локальной сборки, а уже потом переходить к `deployCC`.*

## 9.8. Сравнение синтаксиса JS ↔ Go для одних и тех же операций
| Действие | JavaScript | Go |
|---|---|---|
| Запись в реестр | `ctx.stub.putState(key, Buffer.from(JSON.stringify(obj)))` | `ctx.GetStub().PutState(key, jsonBytes)` (после `json.Marshal`) |
| Чтение из реестра | `await ctx.stub.getState(key)` | `ctx.GetStub().GetState(key)` |
| Удаление | `await ctx.stub.deleteState(key)` | `ctx.GetStub().DelState(key)` |
| MSP ID вызывающего | `ctx.clientIdentity.getMSPID()` | `cid.GetMSPID(ctx.GetStub())` |
| Обработка ошибок | `throw new Error(...)` | `return fmt.Errorf(...)` (Go не использует исключения — ошибка возвращается явно) |
| Сериализация объекта | `JSON.stringify(obj)` | `json.Marshal(obj)` |

## 9.9. Деплой chaincode на Go
Отличие от деплоя JS-варианта — только в флаге языка:
```bash
./network.sh deployCC -ccn Cars -ccl go -ccp ../my-project/chaincode-go -c blockchain2024 -cci InitLedger
```
Подробный разбор всех флагов `deployCC` — в конспекте 11.

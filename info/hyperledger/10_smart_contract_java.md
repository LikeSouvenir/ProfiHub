# 10. Написание смарт-контракта на Java

## 10.1. Подготовка проекта
Java-контракты в Fabric используют аннотации, похожие на Spring — декларативно описывают, какие методы являются транзакциями контракта.

```bash
mkdir -p ~/projects/hl-project/my-project/chaincode-java
cd ~/projects/hl-project/my-project/chaincode-java

gradle init --type java-application
```
В `build.gradle` добавить зависимости:
```groovy
dependencies {
    implementation 'org.hyperledger.fabric-chaincode-java:fabric-chaincode-shim:2.5.7'
    implementation 'com.google.code.gson:gson:2.10.1'
}
```

## 10.2. Структура проекта
```
chaincode-java/
├── build.gradle
└── src/
    └── main/
        └── java/
            └── org/example/
                ├── Car.java
                ├── CarsContract.java
                └── Starter.java
```

## 10.3. Car.java — модель данных
```java
package org.example;

public class Car {
    private String docType = "car";
    private String ID;
    private String color;
    private String brand;
    private String owner;

    public Car(String id, String color, String brand, String owner) {
        this.ID = id;
        this.color = color;
        this.brand = brand;
        this.owner = owner;
    }

    // Геттеры и сеттеры обязательны для сериализации через Gson
    public String getID() { return ID; }
    public String getColor() { return color; }
    public String getBrand() { return brand; }
    public String getOwner() { return owner; }
    public void setOwner(String owner) { this.owner = owner; }
    public String getDocType() { return docType; }
}
```

## 10.4. Готовое полное решение: CarsContract.java
```java
package org.example;

import com.google.gson.Gson;
import org.hyperledger.fabric.contract.Context;
import org.hyperledger.fabric.contract.ContractInterface;
import org.hyperledger.fabric.contract.annotation.Contract;
import org.hyperledger.fabric.contract.annotation.Default;
import org.hyperledger.fabric.contract.annotation.Transaction;
import org.hyperledger.fabric.shim.ChaincodeException;
import org.hyperledger.fabric.shim.ledger.QueryResultsIterator;
import org.hyperledger.fabric.shim.ledger.KeyValue;

import java.util.ArrayList;
import java.util.List;

@Contract(name = "MyCars")
@Default
public class CarsContract implements ContractInterface {

    private final Gson gson = new Gson();

    @Transaction()
    public void InitLedger(final Context ctx) {
        addCarInternal(ctx, "car1", "red", "BMW", "admin");
        addCarInternal(ctx, "car2", "black", "Mercedes", "admin");
    }

    @Transaction()
    public Car AddCar(final Context ctx, final String id, final String color,
                       final String brand, final String owner) {
        if (carExists(ctx, id)) {
            throw new ChaincodeException("Машина " + id + " уже существует");
        }
        return addCarInternal(ctx, id, color, brand, owner);
    }

    @Transaction(intent = Transaction.TYPE.EVALUATE)
    public Car GetCar(final Context ctx, final String id) {
        String carJSON = ctx.getStub().getStringState(id);
        if (carJSON == null || carJSON.isEmpty()) {
            throw new ChaincodeException("Машины " + id + " не существует");
        }
        return gson.fromJson(carJSON, Car.class);
    }

    @Transaction()
    public String TransferCar(final Context ctx, final String id, final String newOwner) {
        Car car = GetCar(ctx, id);
        String oldOwner = car.getOwner();
        car.setOwner(newOwner);

        ctx.getStub().putStringState(id, gson.toJson(car));
        return oldOwner;
    }

    @Transaction()
    public void DeleteCar(final Context ctx, final String id) {
        if (!carExists(ctx, id)) {
            throw new ChaincodeException("Машины " + id + " не существует");
        }
        ctx.getStub().delState(id);
    }

    @Transaction(intent = Transaction.TYPE.EVALUATE)
    public boolean CarExistsPublic(final Context ctx, final String id) {
        return carExists(ctx, id);
    }

    @Transaction(intent = Transaction.TYPE.EVALUATE)
    public String GetAllCars(final Context ctx) {
        List<Car> results = new ArrayList<>();

        QueryResultsIterator<KeyValue> queryResults = ctx.getStub().getStateByRange("", "");
        for (KeyValue result : queryResults) {
            try {
                Car car = gson.fromJson(result.getStringValue(), Car.class);
                if ("car".equals(car.getDocType())) {
                    results.add(car);
                }
            } catch (Exception e) {
                // пропускаем записи другого типа
            }
        }

        return gson.toJson(results);
    }

    // ==== Вспомогательные приватные методы (не аннотированы @Transaction — недоступны извне) ====

    private Car addCarInternal(Context ctx, String id, String color, String brand, String owner) {
        Car car = new Car(id, color, brand, owner);
        ctx.getStub().putStringState(id, gson.toJson(car));
        return car;
    }

    private boolean carExists(Context ctx, String id) {
        String carJSON = ctx.getStub().getStringState(id);
        return carJSON != null && !carJSON.isEmpty();
    }
}
```

## 10.5. Проверка прав доступа (MSP) в Java
```java
@Transaction()
public void DeleteCar(final Context ctx, final String id) {
    String mspId = ctx.getClientIdentity().getMSPID();
    if ("Org3MSP".equals(mspId)) {
        throw new ChaincodeException("Организации Org3 запрещено удалять машины");
    }

    if (!carExists(ctx, id)) {
        throw new ChaincodeException("Машины " + id + " не существует");
    }
    ctx.getStub().delState(id);
}
```

## 10.6. Starter.java — точка входа
```java
package org.example;

import org.hyperledger.fabric.shim.ChaincodeBase;

public class Starter {
    public static void main(String[] args) {
        ChaincodeBase.main(args);
    }
}
```

## 10.7. Ключевые отличия Java-варианта от JS и Go
| Особенность | Java |
|---|---|
| Как объявляется контракт | Аннотация `@Contract` над классом, `implements ContractInterface` |
| Как объявляется транзакция | Аннотация `@Transaction()` над методом |
| Разделение чтения/записи | `@Transaction(intent = Transaction.TYPE.EVALUATE)` — явно помечает функцию как "только чтение" |
| Как выбрасывается ошибка | `throw new ChaincodeException(сообщение)` |
| Сериализация объектов | Обычно через сторонние библиотеки (Gson, Jackson) — Fabric не навязывает конкретную |
| Приватные вспомогательные методы | Обычные методы без `@Transaction` — недоступны для внешнего вызова, как приватные методы класса |

## 10.8. Сборка перед деплоем
```bash
cd ~/projects/hl-project/my-project/chaincode-java
gradle build
```
Убедитесь, что сборка (`BUILD SUCCESSFUL`) проходит локально до попытки развернуть chaincode в сети — так же, как и с Go.

## 10.9. Деплой chaincode на Java
```bash
./network.sh deployCC -ccn Cars -ccl java -ccp ../my-project/chaincode-java -c blockchain2024 -cci InitLedger
```

## 10.10. Когда выбирать какой язык — итоговая рекомендация
| Язык | Когда выбирать |
|---|---|
| **JavaScript** | Команда уже пишет на JS/Node.js (фронтенд, API) — минимальный порог входа, единый язык для всего проекта |
| **Go** | Важна максимальная производительность chaincode, команда знакома с Go, либо предполагается вклад в сам Fabric/его инструменты |
| **Java** | Команда с бэкграундом в корпоративной Java-разработке (Spring и подобное), проект интегрируется с существующей Java-инфраструктурой |

Все три варианта полностью равноценны с точки зрения возможностей Fabric — разница только в удобстве для конкретной команды разработчиков.

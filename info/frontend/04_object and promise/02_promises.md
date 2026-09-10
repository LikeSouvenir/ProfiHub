# Promise

## Проблема: асинхронный код
Некоторые операции в JavaScript выполняются не мгновенно: запрос к серверу, чтение файла, таймер. JavaScript не "замирает" в ожидании результата — он продолжает выполнять остальной код, а результат такой операции приходит **позже**, асинхронно.

```js
console.log("1. Начало");

setTimeout(function () {
    console.log("2. Прошло 2 секунды");
}, 2000);

console.log("3. Конец");

// Порядок вывода в консоль: 1 → 3 → 2 (а не 1 → 2 → 3!)
```

## Что такое Promise
**Promise (обещание)** — это объект, который представляет результат асинхронной операции, которого пока может не быть, но он **обязательно появится** в будущем — либо успешно (**resolve**), либо с ошибкой (**reject**).

Promise может находиться в одном из трёх состояний:
| Состояние | Значение |
|---|---|
| `pending` | Операция ещё выполняется, результата пока нет |
| `fulfilled` | Операция завершилась успешно (был вызван `resolve`) |
| `rejected` | Операция завершилась с ошибкой (был вызван `reject`) |

## Создание Promise
```js
const promise = new Promise(function (resolve, reject) {
    const success = true;

    if (success) {
        resolve("Операция прошла успешно!"); // переводим Promise в состояние fulfilled
    } else {
        reject("Произошла ошибка!"); // переводим Promise в состояние rejected
    }
});
```
Функция, переданная в `new Promise(...)`, называется **исполнителем (executor)** — она выполняется сразу же, синхронно, и принимает две функции: `resolve` (вызвать при успехе) и `reject` (вызвать при ошибке).

## Обработка результата: then / catch / finally
```js
promise
    .then(function (result) {
        console.log("Успех:", result); // выполнится, если был вызван resolve
    })
    .catch(function (error) {
        console.log("Ошибка:", error); // выполнится, если был вызван reject
    })
    .finally(function () {
        console.log("Выполнится в любом случае"); // и при успехе, и при ошибке
    });
```

## Пример с задержкой (имитация запроса к серверу)
```js
function fetchUserData(userId) {
    return new Promise(function (resolve, reject) {
        setTimeout(function () {
            if (userId > 0) {
                resolve({ id: userId, name: "Алина" });
            } else {
                reject("Некорректный ID пользователя");
            }
        }, 1000);
    });
}

fetchUserData(1)
    .then(function (user) {
        console.log("Пользователь получен:", user);
    })
    .catch(function (error) {
        console.log("Ошибка:", error);
    });
```

## Цепочка промисов (chaining)
Каждый `.then()` возвращает новый Promise, поэтому вызовы можно выстраивать в цепочку, где результат одного шага передаётся в следующий:
```js
function double(number) {
    return new Promise(resolve => resolve(number * 2));
}

double(5)
    .then(result => double(result)) // 10 -> передаём дальше
    .then(result => double(result)) // 20 -> передаём дальше
    .then(finalResult => console.log(finalResult)); // 40
```

## async/await — удобная запись поверх Promise
```js
async function getUser() {
    try {
        const user = await fetchUserData(1); // "ждём" результат Promise
        console.log("Пользователь:", user);
    } catch (error) {
        console.log("Ошибка:", error);
    }
}

getUser();
```
`await` можно использовать только внутри функции, объявленной с ключевым словом `async`. Он "приостанавливает" выполнение именно этой функции (не блокируя всю программу) до тех пор, пока Promise не завершится — что делает асинхронный код внешне похожим на обычный, последовательный.

## Promise.all — дождаться сразу нескольких промисов
```js
const p1 = fetchUserData(1);
const p2 = fetchUserData(2);

Promise.all([p1, p2]).then(function (results) {
    console.log("Оба пользователя загружены:", results);
});
```
`Promise.all` ждёт, пока **все** переданные промисы завершатся успешно, и возвращает массив с их результатами в том же порядке. Если хотя бы один из промисов завершится с `reject`, весь `Promise.all` сразу перейдёт в состояние `rejected`.

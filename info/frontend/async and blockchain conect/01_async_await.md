# Асинхронное программирование: async/await

## Зачем нужен async/await
`.then()`/`.catch()` из темы про Promise отлично работают, но при большом количестве последовательных асинхронных шагов код превращается в длинную цепочку вложенных вызовов, которую тяжело читать. `async/await` — это синтаксический "сахар" поверх Promise, который позволяет писать асинхронный код так, будто он выполняется последовательно, сверху вниз, как обычный синхронный.

## Ключевое слово async
Ставится перед объявлением функции и означает: **эта функция всегда возвращает Promise**, даже если внутри неё нет явного `return new Promise(...)`.

```js
async function getGreeting() {
    return "Привет!";
}

// Несмотря на обычный return, функция вернёт Promise
getGreeting().then(result => console.log(result)); // "Привет!"
```

## Ключевое слово await
Используется **только внутри** `async`-функции. "Приостанавливает" выполнение именно этой функции до тех пор, пока промис, стоящий справа от `await`, не завершится — и возвращает его результат напрямую, без `.then()`.

```js
function wait(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
}

async function main() {
    console.log("Начало");

    await wait(1000); // ждём 1 секунду

    console.log("Прошла секунда");
}

main();
```
Важно: `await` не блокирует всю программу — параллельно продолжают выполняться другие части кода (например, обработчики событий на странице), приостанавливается только конкретная `async`-функция.

## Обработка ошибок: try/catch
С `.then()/.catch()` ошибки ловятся через `.catch()`. С `async/await` для этого используется обычный `try/catch`, знакомый по синхронному коду:
```js
function checkNumber(number) {
    return new Promise((resolve, reject) => {
        if (number > 0) {
            resolve("Число положительное");
        } else {
            reject("Число не положительное");
        }
    });
}

async function run() {
    try {
        const result = await checkNumber(5);
        console.log(result); // "Число положительное"

        const result2 = await checkNumber(-3);
        console.log(result2); // сюда выполнение уже не дойдёт
    } catch (error) {
        console.log("Поймали ошибку:", error); // "Поймали ошибку: Число не положительное"
    }
}

run();
```

## Последовательные асинхронные шаги
Главное преимущество `async/await` раскрывается, когда нужно выполнить несколько асинхронных операций одну за другой, где каждая следующая зависит от результата предыдущей:

```js
async function loadUserProfile(userId) {
    const user = await fetchUser(userId);          // шаг 1: получить пользователя
    const posts = await fetchUserPosts(user.id);    // шаг 2: получить его посты
    const comments = await fetchPostComments(posts[0].id); // шаг 3: комментарии к первому посту

    console.log({ user, posts, comments });
}
```
Тот же код через `.then()` потребовал бы вложенных друг в друга колбэков — с `async/await` он читается сверху вниз, как обычная последовательность шагов.

## Параллельное выполнение с await
Если несколько асинхронных операций **не зависят** друг от друга, ждать их по очереди — не оптимально. Правильнее запустить их параллельно через `Promise.all`, а затем дождаться всех разом:

```js
async function loadDashboard() {
    // Оба запроса стартуют одновременно, а не по очереди
    const [users, orders] = await Promise.all([
        fetchUsers(),
        fetchOrders(),
    ]);

    console.log(users, orders);
}
```

## Итоговое сравнение
```js
// Через Promise .then()
function loadData() {
    return fetchUser(1)
        .then(user => fetchPosts(user.id))
        .then(posts => console.log(posts))
        .catch(error => console.log(error));
}

// То же самое через async/await
async function loadData() {
    try {
        const user = await fetchUser(1);
        const posts = await fetchPosts(user.id);
        console.log(posts);
    } catch (error) {
        console.log(error);
    }
}
```
`async/await` не заменяет Promise, а строится поверх него — под капотом это всё те же промисы, просто с более понятным синтаксисом.

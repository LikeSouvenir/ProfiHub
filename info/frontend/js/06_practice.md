# Практика: страницы биографии/новостей + функции JS

Комплексное задание, объединяющее вёрстку (HTML/CSS) и JavaScript: две связанные страницы и три функции на типы данных, операторы, циклы/ветвления.

## Часть 1. Две взаимосвязанные страницы

### Условие
- **index.html** — шапка (`header`), подвал (`footer`), в центре — фото и мини-биография. В шапке — ссылка на страницу News.
- **news.html** — та же структура шапки/подвала, но в центре — несколько новостей, каждая в формате: картинка сверху, под ней текст новости, и так далее друг под другом.

### index.html
```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Обо мне</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>

    <header class="site-header">
        <div class="logo">MySite</div>
        <nav>
            <a href="index.html">Обо мне</a>
            <!-- Ссылка на соседнюю страницу News -->
            <a href="news.html">News</a>
        </nav>
    </header>

    <main class="content">
        <img src="me.jpg" alt="Моя фотография" class="profile-photo">
        <h1>Привет, я Алина</h1>
        <p>
            Изучаю веб-разработку: HTML, CSS и JavaScript. Люблю решать
            практические задачи и постепенно собирать из них полноценные проекты.
            Планирую развиваться дальше во frontend-направлении.
        </p>
    </main>

    <footer class="site-footer">
        <p>&copy; 2026 Алина. Учебный проект.</p>
    </footer>

</body>
</html>
```

### news.html
```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>News</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>

    <!-- Шапка полностью повторяет шапку index.html, чтобы навигация была на каждой странице -->
    <header class="site-header">
        <div class="logo">MySite</div>
        <nav>
            <a href="index.html">Обо мне</a>
            <a href="news.html">News</a>
        </nav>
    </header>

    <main class="content news-list">
        <!-- Каждая новость: картинка сверху, текст под ней -->
        <article class="news-item">
            <img src="news1.jpg" alt="Новость 1" class="news-image">
            <p class="news-text">Сегодня прошла первая лекция по JavaScript — разобрали типы данных и операторы.</p>
        </article>

        <article class="news-item">
            <img src="news2.jpg" alt="Новость 2" class="news-image">
            <p class="news-text">Начали разбирать циклы и ветвления на практике.</p>
        </article>

        <article class="news-item">
            <img src="news3.jpg" alt="Новость 3" class="news-image">
            <p class="news-text">Следующая тема — функции и область видимости.</p>
        </article>
    </main>

    <footer class="site-footer">
        <p>&copy; 2026 Алина. Учебный проект.</p>
    </footer>

</body>
</html>
```

### style.css (общий для обеих страниц)
```css
body {
    margin: 0;
    font-family: Arial, sans-serif;
    color: #222;
    background-color: #fafafa;
}

/* Шапка с навигацией */
.site-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 15px 30px;
    background-color: #2c3e50;
    color: white;
}

.site-header nav a {
    color: white;
    text-decoration: none;
    margin-left: 20px;
}

.site-header nav a:hover {
    text-decoration: underline;
}

/* Центральный контент */
.content {
    max-width: 600px;
    margin: 30px auto;
    padding: 0 20px;
    text-align: center;
}

.profile-photo {
    width: 150px;
    height: 150px;
    border-radius: 50%;
    object-fit: cover;
    margin-bottom: 15px;
}

/* Список новостей: каждая новость друг под другом */
.news-list {
    text-align: left;
}

.news-item {
    margin-bottom: 25px;
}

.news-image {
    width: 100%;
    border-radius: 8px;
    margin-bottom: 10px;
}

.news-text {
    font-size: 15px;
    line-height: 1.5;
}

/* Подвал */
.site-footer {
    text-align: center;
    padding: 15px;
    background-color: #eceff1;
    font-size: 13px;
    color: #777;
}
```

**Разбор:** обе страницы переиспользуют одинаковые классы (`.site-header`, `.site-footer`, `.content`) — это позволяет держать единый стиль сайта, подключая один и тот же файл `style.css`. Новости в `news.html` — это просто несколько одинаковых блоков `.news-item`, идущих один за другим внутри `<main>`, что и даёт эффект "картинка → текст → следующая новость".

---

## Часть 2. Функция конкатенации строк

### Условие
Написать функцию, которая склеивает строки. Если хотя бы одна строка пустая — вернуть `"error: empty string"`.

```js
function concatStrings(str1, str2) {
    // Проверяем, что обе строки не пустые
    if (str1 === "" || str2 === "") {
        return "error: empty string";
    }

    return str1 + str2;
}

console.log(concatStrings("Привет, ", "мир!")); // "Привет, мир!"
console.log(concatStrings("", "мир!"));         // "error: empty string"
console.log(concatStrings("Привет, ", ""));     // "error: empty string"
```

**Разбор:** условие `str1 === "" || str2 === ""` проверяет каждую строку на строгое равенство пустой строке. Если хотя бы одна пустая — функция сразу завершает работу через `return` и возвращает сообщение об ошибке, не доходя до склейки.

---

## Часть 3. Функция проверки кратности 3

### Условие
Функция принимает число. Если оно кратно 3 — сохранить в массив.

```js
const multiplesOfThree = [];

function checkMultipleOfThree(number) {
    // Оператор % возвращает остаток от деления
    // Если остаток от деления на 3 равен 0 — число кратно 3
    if (number % 3 === 0) {
        multiplesOfThree.push(number);
    }
}

checkMultipleOfThree(9);
checkMultipleOfThree(10);
checkMultipleOfThree(12);
checkMultipleOfThree(7);
checkMultipleOfThree(15);

console.log(multiplesOfThree); // [9, 12, 15]
```

**Разбор:** массив `multiplesOfThree` объявлен снаружи функции (в глобальной области видимости), чтобы результаты накапливались при каждом новом вызове функции, а не создавались заново. Внутри функции — простая проверка `number % 3 === 0` и добавление подходящего числа через `push`.

*Вариант с циклом — сразу проверяем целый массив чисел:*
```js
const numbers = [1, 3, 4, 6, 8, 9, 11, 12];
const result = [];

for (const num of numbers) {
    if (num % 3 === 0) {
        result.push(num);
    }
}

console.log(result); // [3, 6, 9, 12]
```

---

## Часть 4. Переписываем код через ?? и ??=

### Исходный код
```js
let num1 = 10,
    num2 = 20,
    result;

if (result === null || result === undefined) {
    if (num1 !== null && num1 !== undefined) {
        result = num1;
    } else {
        result = num2;
    }
}
```
Смысл кода: если `result` ещё не задан (`null` или `undefined`), присвоить ему `num1`, если тот задан, иначе — `num2`.

### Оператор нулевого слияния (`??`)
`a ?? b` возвращает `a`, если `a` **не** `null` и **не** `undefined`, иначе возвращает `b`. В отличие от `||`, оператор `??` не реагирует на другие falsy-значения вроде `0` или `""`.

### Оператор нулевого присваивания (`??=`)
`a ??= b` — это сокращённая запись `a = a ?? b`: присваивает `a` значение `b`, только если `a` сейчас `null` или `undefined`.

### Переписанный код
```js
let num1 = 10,
    num2 = 20,
    result;

// Если result ещё null/undefined — присвоить ему (num1 ?? num2)
result ??= num1 ?? num2;

console.log(result); // 10, потому что num1 задан и не равен null/undefined
```

**Разбор:**
1. `num1 ?? num2` сначала выбирает `num1`, если тот не `null`/`undefined`, иначе берёт `num2` — это заменяет весь внутренний `if/else`.
2. `result ??= (...)` присваивает результат переменной `result`, только если сама `result` изначально была `null` или `undefined` — это заменяет внешний `if`.

Весь исходный блок из 8 строк сворачивается в одну строку без потери логики.

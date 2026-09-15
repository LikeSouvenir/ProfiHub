# Модуль 3: Связка HTML/CSS и JavaScript — Эталонное решение

## index.html

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>TaskTracker</title>
    <link rel="stylesheet" href="style.css">
    <!-- defer: скрипт скачивается параллельно с разметкой, но выполняется после её разбора.
         Благодаря этому querySelector в app.js гарантированно найдёт все элементы. -->
    <script src="app.js" defer></script>
</head>
<body>

    <header class="header">
        <h1>Трекер заданий</h1>
        <p id="counter">Выполнено: 0 из 0</p>
    </header>

    <main class="container">

        <form id="task-form" class="task-form">
            <input type="text" id="task-input" placeholder="Что нужно сделать?" autocomplete="off">
            <select id="task-track">
                <option value="Solidity">Solidity</option>
                <option value="Frontend">Frontend</option>
                <option value="DevOps">DevOps</option>
            </select>
            <button type="submit">Добавить</button>
        </form>

        <input type="text" id="search" class="search" placeholder="Поиск по заданиям...">

        <div class="filters">
            <!-- data-* атрибуты хранят произвольные данные прямо в разметке;
                 из JS они доступны через element.dataset.filter -->
            <button class="filter-btn active" data-filter="all">Все</button>
            <button class="filter-btn" data-filter="active">Активные</button>
            <button class="filter-btn" data-filter="done">Выполненные</button>
        </div>

        <!-- Список изначально пуст: его наполняет скрипт -->
        <ul id="task-list" class="task-list"></ul>

        <button id="clear-done" class="clear-btn">Удалить выполненные</button>

    </main>

</body>
</html>
```

## style.css

```css
body {
    margin: 0;
    font-family: "Segoe UI", Arial, sans-serif;
    background-color: #f7f8fa;
    color: #23272f;
}

.header {
    background-color: #1b2430;
    color: #fff;
    padding: 20px 30px;
}

.header h1 { margin: 0 0 6px; font-size: 22px; }
.header p  { margin: 0; color: #9fb0c4; font-size: 14px; }

.container {
    max-width: 640px;
    margin: 30px auto;
    padding: 0 20px;
}

.task-form {
    display: flex;
    gap: 10px;
    margin-bottom: 14px;
}

.task-form input[type="text"] { flex: 1; }

.task-form input,
.task-form select,
.search {
    padding: 9px 11px;
    border: 1px solid #c8ced6;
    border-radius: 6px;
    font-family: inherit;
    font-size: 14px;
}

.search { width: 100%; box-sizing: border-box; margin-bottom: 16px; }

.task-form button {
    padding: 9px 18px;
    background-color: #4ea1ff;
    color: #fff;
    border: none;
    border-radius: 6px;
    cursor: pointer;
}

/* Класс добавляется из JS при попытке добавить пустое задание */
.error {
    border-color: #e04b4b !important;
    background-color: #fdeaea;
}

.filters { display: flex; gap: 8px; margin-bottom: 16px; }

.filter-btn {
    padding: 6px 14px;
    background-color: #fff;
    border: 1px solid #c8ced6;
    border-radius: 20px;
    cursor: pointer;
    font-size: 13px;
}

.filter-btn.active {
    background-color: #1b2430;
    border-color: #1b2430;
    color: #fff;
}

.task-list { list-style: none; padding: 0; margin: 0; }

.task-list li {
    display: flex;
    align-items: center;
    gap: 10px;
    background-color: #fff;
    border: 1px solid #e3e6ea;
    border-radius: 8px;
    padding: 10px 14px;
    margin-bottom: 8px;
}

.task-text { flex: 1; }

/* Класс done вешается на <li> при отметке чекбокса */
.task-list li.done .task-text {
    text-decoration: line-through;
    color: #9aa3b0;
}

.badge {
    font-size: 11px;
    text-transform: uppercase;
    background-color: #eef3fa;
    color: #3f6fa8;
    border-radius: 12px;
    padding: 3px 9px;
}

.delete-btn {
    border: none;
    background: none;
    color: #b4bcc7;
    font-size: 16px;
    cursor: pointer;
}

.delete-btn:hover { color: #e04b4b; }

/* Универсальный класс скрытия — используется фильтрами и поиском */
.hidden { display: none; }

.clear-btn {
    margin-top: 14px;
    padding: 8px 16px;
    background-color: #fff;
    border: 1px solid #e0a3a3;
    color: #c14a4a;
    border-radius: 6px;
    cursor: pointer;
}
```

## app.js

```js
"use strict";

// ============================================================
// 1. ПОЛУЧАЕМ ССЫЛКИ НА ЭЛЕМЕНТЫ ОДИН РАЗ ПРИ ЗАГРУЗКЕ
// ============================================================
// Искать элемент заново внутри каждого обработчика — лишняя работа для браузера

const form        = document.querySelector("#task-form");
const input       = document.querySelector("#task-input");
const trackSelect = document.querySelector("#task-track");
const list        = document.querySelector("#task-list");
const counter     = document.querySelector("#counter");
const searchInput = document.querySelector("#search");
const clearBtn    = document.querySelector("#clear-done");
const filterBtns  = document.querySelectorAll(".filter-btn"); // NodeList всех кнопок фильтра

// Текущее состояние фильтра храним в переменной модуля
let currentFilter = "all";

// ============================================================
// 2. ДОБАВЛЕНИЕ ЗАДАНИЯ (submit)
// ============================================================

form.addEventListener("submit", function (event) {
    // Без preventDefault браузер отправит форму и перезагрузит страницу
    event.preventDefault();

    const text = input.value.trim(); // trim убирает пробелы по краям

    if (text === "") {
        showInputError();
        return; // выходим из обработчика, задание не создаём
    }

    createTask(text, trackSelect.value);

    input.value = "";  // очищаем поле
    input.focus();     // и сразу возвращаем в него курсор — удобно вводить подряд

    applyFilters();
    updateCounter();
});

/** Подсветка поля при попытке добавить пустое задание */
function showInputError() {
    input.classList.add("error");

    // Через 1.5 секунды убираем подсветку
    setTimeout(function () {
        input.classList.remove("error");
    }, 1500);
}

/**
 * Создаёт элемент списка методами DOM.
 * innerHTML со склейкой строк здесь не используется: это медленнее
 * и опасно, если текст пришёл от пользователя (XSS).
 */
function createTask(text, track) {
    const li = document.createElement("li");
    // dataset пишет атрибут data-track прямо в разметку — пригодится для фильтров
    li.dataset.track = track;

    const checkbox = document.createElement("input");
    checkbox.type = "checkbox";

    const span = document.createElement("span");
    span.className = "task-text";
    span.textContent = text; // textContent вставляет текст как текст, а не как HTML

    const badge = document.createElement("span");
    badge.className = "badge";
    badge.textContent = track;

    const deleteBtn = document.createElement("button");
    deleteBtn.className = "delete-btn";
    deleteBtn.textContent = "✕";
    deleteBtn.setAttribute("aria-label", "Удалить задание");

    // append принимает сразу несколько узлов
    li.append(checkbox, span, badge, deleteBtn);
    list.append(li);
}

// ============================================================
// 3. ОТМЕТКА ВЫПОЛНЕНИЯ (change, делегирование)
// ============================================================

list.addEventListener("change", function (event) {
    if (event.target.type !== "checkbox") {
        return;
    }

    // closest поднимается вверх по дереву DOM до ближайшего подходящего предка
    const li = event.target.closest("li");
    li.classList.toggle("done", event.target.checked);

    applyFilters();
    updateCounter();
});

// ============================================================
// 4. УДАЛЕНИЕ (click, делегирование событий)
// ============================================================
// Один обработчик на весь список вместо отдельного на каждую кнопку:
// он работает и для элементов, созданных позже — событие всплывает до <ul>

list.addEventListener("click", function (event) {
    if (!event.target.classList.contains("delete-btn")) {
        return;
    }

    event.target.closest("li").remove();
    updateCounter();
});

// ============================================================
// 5. ФИЛЬТРЫ (click)
// ============================================================

filterBtns.forEach(function (btn) {
    btn.addEventListener("click", function () {
        currentFilter = btn.dataset.filter;

        // Снимаем подсветку со всех кнопок и ставим её только текущей
        filterBtns.forEach(function (b) {
            b.classList.remove("active");
        });
        btn.classList.add("active");

        applyFilters();
    });
});

// ============================================================
// 6. ЖИВОЙ ПОИСК (input)
// ============================================================
// Событие input срабатывает на каждое изменение значения,
// в отличие от change, который ждёт потери фокуса

searchInput.addEventListener("input", applyFilters);

/**
 * Единая функция видимости: учитывает и активный фильтр, и строку поиска.
 * Так две независимые механики не конфликтуют между собой.
 */
function applyFilters() {
    const query = searchInput.value.trim().toLowerCase();
    const items = list.querySelectorAll("li");

    items.forEach(function (li) {
        const isDone = li.classList.contains("done");
        const text = li.querySelector(".task-text").textContent.toLowerCase();

        const matchesFilter =
            currentFilter === "all" ||
            (currentFilter === "done" && isDone) ||
            (currentFilter === "active" && !isDone);

        // includes вернёт true и для пустой строки — поиск не мешает, пока поле пустое
        const matchesSearch = text.includes(query);

        li.classList.toggle("hidden", !(matchesFilter && matchesSearch));
    });
}

// ============================================================
// 7. СЧЁТЧИК
// ============================================================

function updateCounter() {
    const total = list.querySelectorAll("li").length;
    const done = list.querySelectorAll("li.done").length;

    counter.textContent = `Выполнено: ${done} из ${total}`;
}

// ============================================================
// 8. ОЧИСТКА ВЫПОЛНЕННЫХ
// ============================================================

clearBtn.addEventListener("click", function () {
    const doneItems = list.querySelectorAll("li.done");

    doneItems.forEach(function (li) {
        li.remove();
    });

    updateCounter();
});

// ============================================================
// 9. ГОРЯЧАЯ КЛАВИША Escape (keydown)
// ============================================================

input.addEventListener("keydown", function (event) {
    if (event.key === "Escape") {
        input.value = "";
        input.blur(); // снимаем фокус с поля
    }
});

// Первичная инициализация счётчика при загрузке страницы
updateCounter();
```

## Разбор решения

1. **`defer` вместо ручных проверок готовности DOM.** Скрипт с `defer` выполняется после полного разбора разметки, поэтому `querySelector` в начале файла уже находит все элементы. Альтернатива — подключать `<script>` перед `</body>`; вариант с `DOMContentLoaded` при наличии `defer` становится избыточным.
2. **`event.preventDefault()`.** Форма по умолчанию отправляется на сервер и перезагружает страницу, стирая весь список. Отмена поведения по умолчанию — обязательный первый шаг в обработчике `submit`.
3. **Делегирование событий.** Кнопок удаления изначально нет — они появляются динамически. Вешать обработчик на каждую новую кнопку значит дублировать логику и плодить слушатели. Вместо этого один слушатель на `<ul>` ловит всплывающее событие и по `event.target` определяет, куда именно кликнули.
4. **`closest("li")`.** Клик приходит по кнопке или чекбоксу — вложенным элементам. `closest` поднимается вверх по дереву до ближайшего `<li>`, что надёжнее цепочки `parentElement.parentElement`.
5. **`createElement` вместо `innerHTML`.** Склейка HTML-строк с пользовательским текстом открывает дорогу XSS: введённое `<img onerror=...>` выполнится как код. `textContent` вставляет текст как текст, и проблема исчезает.
6. **`classList.toggle` со вторым аргументом.** `toggle("done", flag)` добавляет класс при `true` и снимает при `false` — это заменяет `if/else` с `add`/`remove` одной строкой.
7. **Единая `applyFilters`.** Фильтр по статусу и строка поиска — две независимые механики, которые управляют одной и той же видимостью. Если каждая будет самостоятельно прятать и показывать элементы, они начнут перетирать результат друг друга. Одна функция, принимающая оба условия во внимание, снимает конфликт.
8. **`dataset`.** Атрибуты `data-filter` и `data-track` хранят данные в разметке без «магических» классов, а из JS читаются как обычные свойства объекта `dataset`.
9. **`input` против `change`.** `input` срабатывает на каждый символ (нужен для живого поиска), `change` — по завершении ввода или переключению чекбокса. Выбор правильного события избавляет от лишних перерисовок.

# Обзор объекта document

**`document`** — это встроенный объект браузера, который представляет всю HTML-страницу целиком. Через него JavaScript получает доступ к элементам разметки, может их находить, читать, изменять, создавать и удалять. Именно `document` — точка входа для работы с **DOM (Document Object Model)** — древовидным представлением HTML-документа в памяти браузера.

## Поиск элементов на странице

### getElementById — по id (возвращает один элемент)
```html
<h1 id="title">Заголовок</h1>
```
```js
const title = document.getElementById("title");
console.log(title.textContent); // "Заголовок"
```

### querySelector — по любому CSS-селектору (возвращает первый найденный)
```js
const title = document.querySelector("#title");    // по id
const firstCard = document.querySelector(".card"); // по классу — первая карточка
const firstLink = document.querySelector("a");      // по тегу — первая ссылка
```

### querySelectorAll — по CSS-селектору (возвращает ВСЕ подходящие элементы)
```js
const cards = document.querySelectorAll(".card");

// querySelectorAll возвращает NodeList — по нему можно пройтись циклом
cards.forEach(function (card) {
    console.log(card.textContent);
});
```

### getElementsByClassName / getElementsByTagName (более старый способ)
```js
const cards = document.getElementsByClassName("card"); // все элементы с классом card
const paragraphs = document.getElementsByTagName("p");  // все теги <p>
```
*В современном коде обычно используют `querySelector`/`querySelectorAll` — они гибче за счёт поддержки любых CSS-селекторов.*

## Чтение и изменение содержимого

```js
const box = document.getElementById("box");

// textContent — только текст, без учёта HTML-тегов внутри
box.textContent = "Новый текст";

// innerHTML — можно вставлять HTML-разметку, а не только текст
box.innerHTML = "<strong>Жирный текст</strong>";
```
*Осторожно с `innerHTML`: если вставлять туда текст, введённый пользователем, без проверки — это открывает возможность для XSS-атак (внедрения вредоносного кода). Для простого текста безопаснее `textContent`.*

## Работа с атрибутами
```js
const link = document.querySelector("a");

console.log(link.getAttribute("href")); // прочитать атрибут
link.setAttribute("href", "https://example.com"); // изменить атрибут
link.removeAttribute("target"); // удалить атрибут
```

## Работа со стилями (связь с CSS)
```js
const box = document.getElementById("box");

// Изменение одного CSS-свойства напрямую через JS
box.style.backgroundColor = "yellow";
box.style.fontSize = "20px";
box.style.display = "none"; // скрыть элемент
```

### Работа с классами (предпочтительный способ менять оформление)
```js
const box = document.getElementById("box");

box.classList.add("active");      // добавить класс
box.classList.remove("hidden");   // убрать класс
box.classList.toggle("dark-mode");// добавить класс, если его нет; убрать, если есть
box.classList.contains("active"); // true/false — проверить наличие класса
```
*Хорошая практика: не задавать стили напрямую через `style.*` в коде, а переключать заранее подготовленные CSS-классы через `classList` — так оформление остаётся в CSS-файле, а JS только включает/выключает готовые "режимы".*

## Создание и удаление элементов
```js
// Создаём новый элемент
const newParagraph = document.createElement("p");
newParagraph.textContent = "Я новый абзац, созданный через JS";

// Добавляем его в конец конкретного контейнера
const container = document.getElementById("container");
container.appendChild(newParagraph);

// Удаляем элемент
newParagraph.remove();
```

## Навигация по дереву DOM
```js
const item = document.querySelector(".item");

console.log(item.parentElement);     // родительский элемент
console.log(item.children);          // все дочерние элементы
console.log(item.nextElementSibling);// следующий "соседний" элемент
console.log(item.previousElementSibling); // предыдущий "соседний" элемент
```

## Пример: собираем всё вместе
```html
<button id="toggleBtn">Показать/скрыть</button>
<p id="text" class="hidden">Этот текст можно скрыть или показать</p>
```
```css
.hidden {
    display: none;
}
```
```js
const button = document.getElementById("toggleBtn");
const text = document.getElementById("text");

button.addEventListener("click", function () {
    text.classList.toggle("hidden");
});
```
При каждом клике на кнопку класс `.hidden` то добавляется, то убирается у параграфа — JS управляет **поведением**, а то, как именно выглядит скрытый/показанный текст, полностью определяет CSS.

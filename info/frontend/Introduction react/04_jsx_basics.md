# JSX, CSS и HTML файлы. Синтаксис и основы JSX

## Что такое JSX
**JSX (JavaScript XML)** — это расширение синтаксиса JavaScript, которое позволяет писать разметку, похожую на HTML, прямо внутри JS-кода. Браузер не понимает JSX напрямую — специальный компилятор (в проекте на Vite/Create React App это происходит автоматически "под капотом") превращает JSX в обычные вызовы функций JavaScript.

```jsx
// То, что пишет разработчик (JSX)
const element = <h1>Привет, мир!</h1>;

// Во что это компилируется "под капотом" (упрощённо)
const element = React.createElement('h1', null, 'Привет, мир!');
```

## Основные правила JSX

### 1. Один корневой элемент
Компонент обязан возвращать **один** родительский элемент — нельзя вернуть несколько элементов "на одном уровне":
```jsx
// ❌ Ошибка — два элемента без общего родителя
function App() {
    return (
        <h1>Заголовок</h1>
        <p>Текст</p>
    );
}

// ✅ Правильно — обёрнуто в один div
function App() {
    return (
        <div>
            <h1>Заголовок</h1>
            <p>Текст</p>
        </div>
    );
}

// ✅ Или через "пустой" фрагмент, если лишний div не нужен
function App() {
    return (
        <>
            <h1>Заголовок</h1>
            <p>Текст</p>
        </>
    );
}
```

### 2. Все теги должны быть закрыты
В отличие от обычного HTML, в JSX нельзя оставить тег без явного закрытия — даже одиночные теги вроде `<img>` или `<input>` обязательно закрываются слэшем:
```jsx
<img src="photo.jpg" alt="Фото" />
<input type="text" />
<br />
```

### 3. className вместо class
Слово `class` зарезервировано в JavaScript, поэтому CSS-класс в JSX задаётся через `className`:
```jsx
<div className="card">Карточка</div>
```

### 4. Вставка JS-выражений через { }
Внутри JSX можно вставлять любое JavaScript-выражение в фигурных скобках:
```jsx
const name = "Алина";
const isOnline = true;

function App() {
    return (
        <div>
            <h1>Привет, {name}!</h1>
            <p>{isOnline ? "В сети" : "Не в сети"}</p>
            <p>Сумма: {2 + 2}</p>
        </div>
    );
}
```

### 5. Атрибуты в camelCase
Большинство HTML-атрибутов в JSX записываются в стиле `camelCase`:
```jsx
<div onClick={handleClick} tabIndex={0}></div>
<label htmlFor="username">Имя пользователя</label>
```

## Подключение CSS к JSX-компоненту

### Обычный CSS-файл
```jsx
// App.jsx
import './App.css'

function App() {
    return <div className="container">Контент</div>;
}
```
```css
/* App.css */
.container {
    padding: 20px;
    background-color: #f5f5f5;
}
```

### Инлайн-стили (через объект JS)
```jsx
function App() {
    // Обратите внимание: двойные фигурные скобки —
    // внешние {} для вставки JS-выражения, внутренние {} — сам объект со стилями
    const boxStyle = {
        backgroundColor: "lightblue",
        padding: "10px",
        borderRadius: "8px",
    };

    return <div style={boxStyle}>Стилизованный блок</div>;
}
```
*В JSX названия CSS-свойств в инлайн-стилях пишутся в `camelCase` (`backgroundColor`, а не `background-color`), а значения указываются строками.*

## Списки в JSX
Массив элементов выводится через `map`, и каждому элементу списка обязательно нужен уникальный атрибут `key`:
```jsx
const fruits = ["Яблоко", "Банан", "Апельсин"];

function FruitList() {
    return (
        <ul>
            {fruits.map((fruit, index) => (
                <li key={index}>{fruit}</li>
            ))}
        </ul>
    );
}
```
`key` помогает React эффективно отслеживать, какие элементы списка изменились, добавились или были удалены, не пересоздавая весь список заново.

## Условный рендеринг
```jsx
function Greeting({ isLoggedIn }) {
    return (
        <div>
            {isLoggedIn ? <p>Добро пожаловать!</p> : <p>Пожалуйста, войдите</p>}

            {/* Короткая запись: показать элемент только если условие true */}
            {isLoggedIn && <p>У вас есть доступ к личному кабинету</p>}
        </div>
    );
}
```

## Пример небольшого компонента, объединяющего всё вместе
```jsx
import './Card.css'

function Card({ title, text }) {
    return (
        <div className="card">
            <h2 className="card-title">{title}</h2>
            <p className="card-text">{text}</p>
        </div>
    );
}

export default Card;
```
```css
.card {
    padding: 16px;
    border-radius: 12px;
    background-color: white;
    box-shadow: 0 2px 6px rgba(0, 0, 0, 0.1);
}

.card-title {
    margin: 0 0 8px 0;
    font-size: 18px;
}

.card-text {
    margin: 0;
    color: #555;
}
```
Такой компонент можно переиспользовать сколько угодно раз, просто передавая разные `title` и `text` — это и есть основная идея компонентного подхода в React.

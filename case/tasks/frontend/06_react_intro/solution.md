# Модуль 6: Введение в React — Эталонное решение

## Структура проекта

```
module-showcase/
├── index.html              # единственная HTML-страница: содержит <div id="root">
├── package.json            # зависимости и npm-скрипты (dev, build, preview)
├── vite.config.js          # конфигурация сборщика и плагина React
├── public/                 # статика «как есть»: favicon, robots.txt
└── src/
    ├── main.jsx            # точка входа: монтирует <App /> в #root
    ├── App.jsx             # корневой компонент, собирает страницу
    ├── App.css
    ├── index.css
    ├── data/
    │   └── modules.js
    └── components/
        ├── Header.jsx
        ├── Badge.jsx
        ├── ModuleCard.jsx
        ├── ModuleList.jsx
        └── Footer.jsx
```

## src/data/modules.js

```js
// Именованный экспорт: при импорте имя берётся в фигурные скобки
export const modules = [
    {
        id: 1,
        title: "Типы данных, циклы, функции",
        track: "Solidity",
        level: "базовый",
        lessons: 4,
        hours: 6,
        isCompleted: true,
        tags: ["uint256", "mapping", "struct"],
    },
    {
        id: 2,
        title: "Конструктор и модификаторы",
        track: "Solidity",
        level: "средний",
        lessons: 4,
        hours: 7,
        isCompleted: true,
        tags: ["constructor", "modifier", "custom errors"],
    },
    {
        id: 3,
        title: "ERC-20 и интерфейсы",
        track: "Solidity",
        level: "продвинутый",
        lessons: 4,
        hours: 9,
        isCompleted: false,
        tags: ["ERC20", "interface", "abstract"],
    },
    {
        id: 4,
        title: "HTML и CSS",
        track: "Frontend",
        level: "базовый",
        lessons: 4,
        hours: 5,
        isCompleted: true,
        tags: ["семантика", "flexbox", "box model"],
    },
    {
        id: 5,
        title: "React: основы и хуки",
        track: "Frontend",
        level: "средний",
        lessons: 4,
        hours: 8,
        isCompleted: false,
        tags: ["JSX", "props", "hooks"],
    },
    {
        id: 6,
        title: "Docker и Geth",
        track: "DevOps",
        level: "продвинутый",
        lessons: 6,
        hours: 10,
        isCompleted: false,
        tags: ["Dockerfile", "compose", "node"],
    },
];
```

## src/main.jsx

```jsx
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App.jsx";
import "./index.css";

// Точка входа приложения: находим единственный div#root в index.html
// и монтируем в него всё дерево React-компонентов
ReactDOM.createRoot(document.getElementById("root")).render(
    // StrictMode — режим разработки: подсвечивает устаревшие приёмы
    // и намеренно вызывает компоненты дважды, чтобы выявить побочные эффекты
    <React.StrictMode>
        <App />
    </React.StrictMode>
);
```

## src/components/Header.jsx

```jsx
// Пропсы деструктурируются прямо в параметрах.
// subtitle получает значение по умолчанию, если проп не передан
function Header({ title, subtitle = "База знаний для подготовки к чемпионату" }) {
    return (
        <header className="header">
            <h1 className="header__title">{title}</h1>
            <p className="header__subtitle">{subtitle}</p>
        </header>
    );
}

// Экспорт по умолчанию: при импорте имя выбирает сам импортирующий файл
export default Header;
```

## src/components/Badge.jsx

```jsx
/**
 * Универсальная «обёртка»: всё, что положили между тегами компонента,
 * приходит в специальный проп children и рендерится внутри.
 */
function Badge({ children, color = "gray" }) {
    return (
        // Класс собирается шаблонным литералом — так реализуются модификаторы
        <span className={`badge badge--${color}`}>{children}</span>
    );
}

export default Badge;
```

## src/components/ModuleCard.jsx

```jsx
import Badge from "./Badge.jsx";

function ModuleCard({ id, title, track, level, lessons, hours, isCompleted, tags }) {
    // Обычный JS-код выполняется до return — здесь удобно считать производные значения
    const avgLessonHours = (hours / lessons).toFixed(1);

    // Тернарный оператор выбирает цвет бейджа по уровню сложности
    const levelColor =
        level === "базовый" ? "green" : level === "средний" ? "blue" : "gray";

    return (
        <article className={`card ${isCompleted ? "card--done" : ""}`}>
            <div className="card__head">
                <span className="card__num">#{id}</span>

                {/* Условный рендеринг через &&: если слева false,
                    React ничего не выводит. Работает только с булевым
                    значением слева — не с числом (0 отрендерится как "0") */}
                {isCompleted && <span className="card__done">✓ Пройден</span>}
            </div>

            <h3 className="card__title">{title}</h3>

            <div className="card__badges">
                <Badge color="blue">{track}</Badge>
                {/* children здесь — текст уровня, он попадает внутрь Badge */}
                <Badge color={levelColor}>{level}</Badge>
            </div>

            <ul className="card__meta">
                <li>Уроков: {lessons}</li>
                <li>Часов: {hours}</li>
                <li>В среднем на урок: {avgLessonHours} ч</li>
            </ul>

            <div className="card__tags">
                {/* Список тегов: map возвращает массив элементов.
                    key — строковый тег, он уникален в пределах карточки */}
                {tags.map((tag) => (
                    <span className="tag" key={tag}>
                        {tag}
                    </span>
                ))}
            </div>
        </article>
    );
}

export default ModuleCard;
```

## src/components/ModuleList.jsx

```jsx
import ModuleCard from "./ModuleCard.jsx";

function ModuleList({ title, modules }) {
    return (
        <section className="section">
            <h2 className="section__title">
                {title} <span className="section__count">({modules.length})</span>
            </h2>

            {/* Тернарный оператор: либо список, либо заглушка */}
            {modules.length === 0 ? (
                <p className="empty">Модули не найдены</p>
            ) : (
                <div className="grid">
                    {modules.map((module) => (
                        // key нужен React, чтобы сопоставлять элементы между
                        // перерисовками. Индекс массива здесь не подходит: при
                        // сортировке или удалении он «переезжает» на другой
                        // объект, и React переиспользует чужой DOM-узел.
                        // id уникален и привязан к конкретной сущности
                        <ModuleCard
                            key={module.id}
                            id={module.id}
                            title={module.title}
                            track={module.track}
                            level={module.level}
                            lessons={module.lessons}
                            hours={module.hours}
                            isCompleted={module.isCompleted}
                            tags={module.tags}
                        />
                    ))}
                </div>
            )}
        </section>
    );
}

export default ModuleList;
```

## src/components/Footer.jsx

```jsx
function Footer({ year = new Date().getFullYear() }) {
    return (
        <footer className="footer">
            <p>&copy; {year} ProfiHub Team. Учебный проект.</p>
        </footer>
    );
}

export default Footer;
```

## src/App.jsx

```jsx
import Header from "./components/Header.jsx";
import ModuleList from "./components/ModuleList.jsx";
import Footer from "./components/Footer.jsx";
import { modules } from "./data/modules.js";
import "./App.css";

function App() {
    // Фильтрация по направлениям — обычный JS до return
    const solidityModules = modules.filter((m) => m.track === "Solidity");
    const frontendModules = modules.filter((m) => m.track === "Frontend");
    const devopsModules = modules.filter((m) => m.track === "DevOps");

    const completedCount = modules.filter((m) => m.isCompleted).length;
    const totalHours = modules.reduce((sum, m) => sum + m.hours, 0);
    const progressPercent = Math.round((completedCount / modules.length) * 100);

    return (
        // Компонент обязан вернуть ОДИН корневой элемент.
        // Здесь это <div className="app">, но там, где обёртка не нужна,
        // используется фрагмент <>...</> — он не создаёт лишний узел в DOM
        <div className="app">
            <Header title="ProfiHub — модули подготовки" />

            <main className="container">
                {/* Фрагмент группирует два элемента без лишнего div */}
                <>
                    <section className="summary">
                        <div className="summary__item">
                            <span className="summary__value">{modules.length}</span>
                            <span className="summary__label">модулей</span>
                        </div>
                        <div className="summary__item">
                            <span className="summary__value">{completedCount}</span>
                            <span className="summary__label">пройдено</span>
                        </div>
                        <div className="summary__item">
                            <span className="summary__value">{totalHours}</span>
                            <span className="summary__label">часов всего</span>
                        </div>
                        <div className="summary__item">
                            <span className="summary__value">{progressPercent}%</span>
                            <span className="summary__label">прогресс</span>
                        </div>
                    </section>

                    {/* Инлайн-стиль передаётся объектом: внешние скобки —
                        это выражение JSX, внутренние — литерал объекта.
                        Свойства пишутся в camelCase */}
                    <div className="progress" style={{ marginBottom: "28px" }}>
                        <div
                            className="progress__bar"
                            style={{ width: `${progressPercent}%` }}
                        />
                    </div>
                </>

                <ModuleList title="Смарт-контракты" modules={solidityModules} />
                <ModuleList title="Интерфейсы" modules={frontendModules} />
                <ModuleList title="Инфраструктура" modules={devopsModules} />
                {/* Пустой массив покажет заглушку вместо списка */}
                <ModuleList title="Hyperledger" modules={[]} />
            </main>

            <Footer />
        </div>
    );
}

export default App;
```

## src/App.css

```css
.app { min-height: 100vh; display: flex; flex-direction: column; }

.header {
    background-color: #1b2430;
    color: #fff;
    padding: 28px 30px;
}
.header__title    { margin: 0 0 6px; font-size: 24px; }
.header__subtitle { margin: 0; color: #9fb0c4; font-size: 14px; }

.container { flex: 1; max-width: 1000px; margin: 30px auto; padding: 0 20px; width: 100%; box-sizing: border-box; }

.summary { display: flex; gap: 16px; flex-wrap: wrap; margin-bottom: 16px; }
.summary__item {
    flex: 1 1 140px;
    background-color: #fff;
    border: 1px solid #e3e6ea;
    border-radius: 10px;
    padding: 16px;
    text-align: center;
}
.summary__value { display: block; font-size: 26px; font-weight: bold; color: #1b2430; }
.summary__label { font-size: 13px; color: #6b7482; }

.progress { height: 8px; background-color: #e3e6ea; border-radius: 4px; overflow: hidden; }
.progress__bar { height: 100%; background-color: #4ea1ff; }

.section__title { font-size: 18px; margin: 24px 0 14px; }
.section__count { color: #9aa3b0; font-weight: normal; }

.grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(260px, 1fr)); gap: 16px; }

.card {
    background-color: #fff;
    border: 1px solid #e3e6ea;
    border-radius: 10px;
    padding: 16px;
}
.card--done { border-color: #a8dcb5; background-color: #f6fcf8; }
.card__head { display: flex; justify-content: space-between; align-items: center; }
.card__num  { color: #9aa3b0; font-size: 13px; }
.card__done { color: #2e8b57; font-size: 12px; font-weight: bold; }
.card__title { margin: 8px 0 12px; font-size: 16px; }
.card__badges { display: flex; gap: 6px; margin-bottom: 12px; }
.card__meta { list-style: none; padding: 0; margin: 0 0 12px; font-size: 13px; color: #6b7482; }
.card__tags { display: flex; flex-wrap: wrap; gap: 6px; }

.tag { font-size: 11px; background-color: #f0f2f5; border-radius: 4px; padding: 3px 7px; color: #55606f; }

.badge { font-size: 11px; text-transform: uppercase; border-radius: 12px; padding: 3px 9px; }
.badge--blue  { background-color: #eef3fa; color: #3f6fa8; }
.badge--green { background-color: #eaf7ee; color: #2e8b57; }
.badge--gray  { background-color: #f0f2f5; color: #6b7482; }

.empty { color: #9aa3b0; font-style: italic; }

.footer { background-color: #eceff1; text-align: center; padding: 18px; font-size: 13px; color: #6b7482; }
```

## Разбор решения

1. **Что делает Vite.** `index.html` — единственная страница приложения; в ней есть пустой `<div id="root">` и подключение `main.jsx` как модуля. `main.jsx` монтирует дерево React в этот div. Всё, что видит пользователь, создаётся JavaScript-ом в рантайме. Папка `public` отдаётся как есть (путь `/logo.png`), а `src/assets` проходит через сборщик и попадает в бандл с хешем в имени.
2. **Один корневой элемент.** Функция не может вернуть два значения, а JSX компилируется в вызовы функций — отсюда требование единственного корня. Когда обёртка не нужна в DOM, используется фрагмент `<>...</>`: он группирует элементы, но не создаёт узел.
3. **`className` вместо `class`.** `class` — зарезервированное слово JavaScript, поэтому JSX использует `className`. По той же причине `for` у `<label>` превращается в `htmlFor`.
4. **Фигурные скобки — это выражение.** Внутри `{}` можно писать любое JS-выражение, но не инструкцию: тернарный оператор работает, а `if` — нет. Поэтому условный рендеринг делается тернарником или `&&`.
5. **`&&` и ловушка с нулём.** `{isCompleted && <span/>}` не выведет ничего при `false`. Но `{lessons && <span/>}` при `lessons === 0` выведет на странице «0», потому что React рендерит числа. Слева от `&&` всегда должно стоять именно булево значение.
6. **Props — односторонний поток.** Данные идут сверху вниз: `App` читает `modules.js` и раздаёт отфильтрованные массивы дочерним компонентам. `ModuleList` и `ModuleCard` ничего не знают об источнике данных и потому переиспользуемы — тот же `ModuleList` работает и со списком Solidity, и с пустым массивом.
7. **`key` — это `id`, а не индекс.** React сопоставляет элементы между перерисовками по `key`. Индекс массива привязан к позиции, а не к сущности: после сортировки или удаления элемент под индексом 2 станет другим объектом, и React переиспользует для него старый DOM-узел вместе с состоянием. `id` уникален и стабилен, поэтому ошибок не возникает.
8. **`props.children`.** `Badge` не знает заранее, что в нём покажут: содержимое между открывающим и закрывающим тегами приходит в `children`. Это основной способ делать компоненты-обёртки — карточки, модальные окна, контейнеры.
9. **Значения по умолчанию.** `function Header({ title, subtitle = "..." })` — обычный параметр по умолчанию при деструктуризации. Он читается прямо в сигнатуре и заменяет устаревший `defaultProps`.
10. **Вычисления до `return`.** Фильтры, `reduce` и производные значения считаются в теле функции обычным JavaScript-ом. В JSX остаётся только подстановка результатов — разметка не засоряется логикой.

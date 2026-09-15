# Модуль 8: React Router и Context API — Эталонное решение

## src/data/content.js

```js
export const tracks = [
    { slug: "solidity", title: "Смарт-контракты", description: "Solidity, ERC-стандарты, Hardhat" },
    { slug: "frontend", title: "Интерфейсы",      description: "HTML, CSS, JavaScript, React" },
    { slug: "devops",   title: "Инфраструктура",  description: "Docker, Geth, CI" },
];

export const modules = [
    {
        id: 1, slug: "solidity-basics", trackSlug: "solidity",
        title: "Типы данных, циклы, функции", level: "базовый",
        lessons: [
            { id: 11, title: "Типы данных", minutes: 40 },
            { id: 12, title: "Функции",     minutes: 45 },
            { id: 13, title: "Циклы",       minutes: 30 },
        ],
    },
    {
        id: 2, slug: "solidity-modifiers", trackSlug: "solidity",
        title: "Конструктор и модификаторы", level: "средний",
        lessons: [
            { id: 21, title: "Constructor",           minutes: 35 },
            { id: 22, title: "Ключевые слова и ошибки", minutes: 50 },
            { id: 23, title: "require и modifier",     minutes: 45 },
        ],
    },
    {
        id: 3, slug: "solidity-erc20", trackSlug: "solidity",
        title: "ERC-20 и интерфейсы", level: "продвинутый",
        lessons: [
            { id: 31, title: "Интерфейсы",          minutes: 40 },
            { id: 32, title: "Абстрактные контракты", minutes: 40 },
            { id: 33, title: "Обзор ERC-20",         minutes: 55 },
        ],
    },
    {
        id: 4, slug: "html-css", trackSlug: "frontend",
        title: "HTML и CSS", level: "базовый",
        lessons: [
            { id: 41, title: "Базовые теги", minutes: 35 },
            { id: 42, title: "Селекторы CSS", minutes: 40 },
        ],
    },
    {
        id: 5, slug: "react-basics", trackSlug: "frontend",
        title: "React: основы и хуки", level: "средний",
        lessons: [
            { id: 51, title: "JSX",        minutes: 40 },
            { id: 52, title: "useState",   minutes: 45 },
            { id: 53, title: "useEffect",  minutes: 50 },
        ],
    },
    {
        id: 6, slug: "docker-geth", trackSlug: "devops",
        title: "Docker и Geth", level: "продвинутый",
        lessons: [
            { id: 61, title: "Dockerfile",     minutes: 45 },
            { id: 62, title: "docker compose", minutes: 50 },
            { id: 63, title: "Запуск ноды",    minutes: 60 },
        ],
    },
];
```

## src/context/ContentContext.jsx

```jsx
import { createContext, useContext, useMemo, useCallback } from "react";
import { tracks, modules } from "../data/content.js";

// 1. Создаём контекст. Значение по умолчанию null — по нему
//    мы определим, что хук вызвали вне провайдера
const ContentContext = createContext(null);

// 2. Провайдер: компонент, который кладёт данные в контекст
//    и рендерит внутри себя всё дерево приложения
export function ContentProvider({ children }) {
    const getTrackBySlug = useCallback(
        (slug) => tracks.find((track) => track.slug === slug),
        []
    );

    const getModulesByTrack = useCallback(
        (trackSlug) => modules.filter((module) => module.trackSlug === trackSlug),
        []
    );

    const getModuleBySlug = useCallback(
        (slug) => modules.find((module) => module.slug === slug),
        []
    );

    // useMemo не даёт создавать новый объект на каждом рендере.
    // Без него любая перерисовка провайдера меняла бы ссылку value,
    // и КАЖДЫЙ потребитель контекста перерисовывался бы впустую
    const value = useMemo(
        () => ({ tracks, modules, getTrackBySlug, getModulesByTrack, getModuleBySlug }),
        [getTrackBySlug, getModulesByTrack, getModuleBySlug]
    );

    return <ContentContext.Provider value={value}>{children}</ContentContext.Provider>;
}

// 3. Собственный хук доступа: скрывает useContext и даёт
//    понятную ошибку вместо «Cannot read properties of null»
export function useContent() {
    const context = useContext(ContentContext);

    if (context === null) {
        throw new Error("useContent должен использоваться внутри <ContentProvider>");
    }

    return context;
}
```

## src/context/FavoritesContext.jsx

```jsx
import { createContext, useContext, useState, useMemo, useCallback } from "react";

const FavoritesContext = createContext(null);

export function FavoritesProvider({ children }) {
    const [favoriteIds, setFavoriteIds] = useState([]);

    // useCallback сохраняет одну и ту же ссылку на функцию между рендерами.
    // Иначе каждая перерисовка создавала бы новую функцию, менялась бы
    // ссылка value, и useMemo ниже терял бы смысл
    const toggleFavorite = useCallback((id) => {
        setFavoriteIds((prev) =>
            prev.includes(id) ? prev.filter((item) => item !== id) : [...prev, id]
        );
    }, []);

    const isFavorite = useCallback((id) => favoriteIds.includes(id), [favoriteIds]);

    const clearFavorites = useCallback(() => setFavoriteIds([]), []);

    const value = useMemo(
        () => ({
            favoriteIds,
            favoritesCount: favoriteIds.length,
            toggleFavorite,
            isFavorite,
            clearFavorites,
        }),
        [favoriteIds, toggleFavorite, isFavorite, clearFavorites]
    );

    return <FavoritesContext.Provider value={value}>{children}</FavoritesContext.Provider>;
}

export function useFavorites() {
    const context = useContext(FavoritesContext);

    if (context === null) {
        throw new Error("useFavorites должен использоваться внутри <FavoritesProvider>");
    }

    return context;
}
```

## src/main.jsx

```jsx
import React from "react";
import ReactDOM from "react-dom/client";
import { BrowserRouter } from "react-router-dom";
import { ContentProvider } from "./context/ContentContext.jsx";
import { FavoritesProvider } from "./context/FavoritesContext.jsx";
import App from "./App.jsx";
import "./index.css";

ReactDOM.createRoot(document.getElementById("root")).render(
    <React.StrictMode>
        {/* Провайдеры оборачивают роутер: контекст доступен на всех страницах.
            BrowserRouter даёт «чистые» адреса вида /tracks/solidity через
            History API. HashRouter пишет /#/tracks/solidity и не требует
            настройки сервера — он нужен для статического хостинга,
            который не умеет отдавать index.html на любой путь */}
        <ContentProvider>
            <FavoritesProvider>
                <BrowserRouter>
                    <App />
                </BrowserRouter>
            </FavoritesProvider>
        </ContentProvider>
    </React.StrictMode>
);
```

## src/App.jsx

```jsx
import { Routes, Route } from "react-router-dom";
import Layout from "./components/Layout.jsx";
import HomePage from "./pages/HomePage.jsx";
import TracksPage from "./pages/TracksPage.jsx";
import TrackPage from "./pages/TrackPage.jsx";
import ModulePage from "./pages/ModulePage.jsx";
import FavoritesPage from "./pages/FavoritesPage.jsx";
import AboutPage from "./pages/AboutPage.jsx";
import NotFoundPage from "./pages/NotFoundPage.jsx";
import "./App.css";

function App() {
    return (
        // Routes выбирает ОДИН наиболее подходящий Route и рендерит его
        <Routes>
            {/* Родительский маршрут задаёт общий каркас страницы.
                Дочерние маршруты подставляются в его <Outlet /> */}
            <Route path="/" element={<Layout />}>
                {/* index — маршрут, совпадающий с путём родителя ("/") */}
                <Route index element={<HomePage />} />
                <Route path="tracks" element={<TracksPage />} />
                {/* Двоеточие объявляет динамический сегмент;
                    значение читается хуком useParams */}
                <Route path="tracks/:trackSlug" element={<TrackPage />} />
                <Route path="modules/:moduleSlug" element={<ModulePage />} />
                <Route path="favorites" element={<FavoritesPage />} />
                <Route path="about" element={<AboutPage />} />
                {/* Звёздочка ловит всё, что не совпало выше.
                    Этот маршрут всегда идёт последним */}
                <Route path="*" element={<NotFoundPage />} />
            </Route>
        </Routes>
    );
}

export default App;
```

## src/components/Layout.jsx

```jsx
import { Outlet } from "react-router-dom";
import Navbar from "./Navbar.jsx";

function Layout() {
    return (
        <div className="app">
            <Navbar />

            <main className="container">
                {/* Outlet — точка подстановки текущего дочернего маршрута.
                    Шапка и подвал при переходах не перемонтируются */}
                <Outlet />
            </main>

            <footer className="footer">
                <p>&copy; {new Date().getFullYear()} ProfiHub. База знаний.</p>
            </footer>
        </div>
    );
}

export default Layout;
```

## src/components/Navbar.jsx

```jsx
import { NavLink } from "react-router-dom";
import { useFavorites } from "../context/FavoritesContext.jsx";

function Navbar() {
    // Данные берутся напрямую из контекста.
    // Без него счётчик пришлось бы тащить пропсами через App и Layout,
    // которые сами по себе о избранном ничего не знают — это и есть
    // prop drilling, который решает Context API
    const { favoritesCount } = useFavorites();

    // NavLink принимает className функцией и сам сообщает,
    // активна ли ссылка для текущего адреса
    const linkClass = ({ isActive }) =>
        isActive ? "nav__link nav__link--active" : "nav__link";

    return (
        <header className="navbar">
            <div className="navbar__logo">ProfiHub</div>

            <nav className="nav">
                {/* end нужен главной: без него "/" считалась бы активной
                    на всех вложенных путях, потому что все они с неё начинаются */}
                <NavLink to="/" className={linkClass} end>
                    Главная
                </NavLink>
                <NavLink to="/tracks" className={linkClass}>
                    Направления
                </NavLink>
                <NavLink to="/favorites" className={linkClass}>
                    Избранное
                    {favoritesCount > 0 && <span className="counter">{favoritesCount}</span>}
                </NavLink>
                <NavLink to="/about" className={linkClass}>
                    О проекте
                </NavLink>
            </nav>
        </header>
    );
}

export default Navbar;
```

## src/pages/HomePage.jsx

```jsx
import { Link, useNavigate } from "react-router-dom";
import { useContent } from "../context/ContentContext.jsx";
import { useFavorites } from "../context/FavoritesContext.jsx";

function HomePage() {
    const { tracks, modules } = useContent();
    const { favoritesCount } = useFavorites();

    // useNavigate даёт программную навигацию — переход из обработчика,
    // а не по клику на ссылку
    const navigate = useNavigate();

    const handleContinue = () => {
        const nextModule = modules.find((module) => module.level === "средний") ?? modules[0];
        navigate(`/modules/${nextModule.slug}`);
    };

    return (
        <section>
            <h1>База знаний ProfiHub</h1>
            <p className="lead">
                {modules.length} модулей по {tracks.length} направлениям.
                В избранном: {favoritesCount}.
            </p>

            <button type="button" className="btn btn--primary" onClick={handleContinue}>
                Продолжить обучение
            </button>

            <h2>Направления</h2>
            <div className="grid">
                {tracks.map((track) => (
                    // Link рендерит <a>, но перехватывает клик и меняет адрес
                    // через History API — без перезагрузки страницы
                    <Link key={track.slug} to={`/tracks/${track.slug}`} className="card card--link">
                        <h3>{track.title}</h3>
                        <p>{track.description}</p>
                    </Link>
                ))}
            </div>
        </section>
    );
}

export default HomePage;
```

## src/pages/TracksPage.jsx

```jsx
import { useSearchParams, Link } from "react-router-dom";
import { useContent } from "../context/ContentContext.jsx";

const LEVELS = ["все", "базовый", "средний", "продвинутый"];

function TracksPage() {
    const { modules } = useContent();

    // useSearchParams хранит состояние фильтров в адресной строке.
    // Благодаря этому страницу можно перезагрузить или отправить ссылкой,
    // и фильтры сохранятся — обычный useState так не умеет
    const [searchParams, setSearchParams] = useSearchParams();

    const query = searchParams.get("q") ?? "";
    const level = searchParams.get("level") ?? "все";

    const updateParam = (key, value) => {
        const next = new URLSearchParams(searchParams);

        // Пустое значение убираем из URL, чтобы адрес не засорялся
        if (value === "" || value === "все") {
            next.delete(key);
        } else {
            next.set(key, value);
        }

        setSearchParams(next);
    };

    const visibleModules = modules.filter((module) => {
        const matchesQuery = module.title.toLowerCase().includes(query.trim().toLowerCase());
        const matchesLevel = level === "все" || module.level === level;
        return matchesQuery && matchesLevel;
    });

    return (
        <section>
            <h1>Все модули</h1>

            <div className="filter-bar">
                <input
                    type="text"
                    className="input"
                    value={query}
                    onChange={(event) => updateParam("q", event.target.value)}
                    placeholder="Поиск по названию..."
                />

                <div className="filters">
                    {LEVELS.map((item) => (
                        <button
                            key={item}
                            type="button"
                            className={level === item ? "chip chip--active" : "chip"}
                            onClick={() => updateParam("level", item)}
                        >
                            {item}
                        </button>
                    ))}
                </div>
            </div>

            {visibleModules.length === 0 ? (
                <p className="empty">Ничего не найдено</p>
            ) : (
                <div className="grid">
                    {visibleModules.map((module) => (
                        <Link
                            key={module.id}
                            to={`/modules/${module.slug}`}
                            className="card card--link"
                        >
                            <h3>{module.title}</h3>
                            <span className="badge">{module.level}</span>
                            <p className="muted">Уроков: {module.lessons.length}</p>
                        </Link>
                    ))}
                </div>
            )}
        </section>
    );
}

export default TracksPage;
```

## src/pages/TrackPage.jsx

```jsx
import { useParams, Navigate, Link } from "react-router-dom";
import { useContent } from "../context/ContentContext.jsx";

function TrackPage() {
    // useParams возвращает объект динамических сегментов маршрута.
    // Имя ключа совпадает с тем, что указано после двоеточия в path
    const { trackSlug } = useParams();
    const { getTrackBySlug, getModulesByTrack } = useContent();

    const track = getTrackBySlug(trackSlug);

    // Направления с таким slug нет — уводим на 404.
    // replace убирает несуществующий адрес из истории,
    // чтобы кнопка «назад» не вернула пользователя на ошибку
    if (!track) {
        return <Navigate to="/404" replace />;
    }

    const trackModules = getModulesByTrack(trackSlug);

    return (
        <section>
            <Link to="/tracks" className="back-link">← Ко всем направлениям</Link>

            <h1>{track.title}</h1>
            <p className="lead">{track.description}</p>

            <div className="grid">
                {trackModules.map((module) => (
                    <Link key={module.id} to={`/modules/${module.slug}`} className="card card--link">
                        <h3>{module.title}</h3>
                        <span className="badge">{module.level}</span>
                    </Link>
                ))}
            </div>
        </section>
    );
}

export default TrackPage;
```

## src/pages/ModulePage.jsx

```jsx
import { useParams, useNavigate, Link } from "react-router-dom";
import { useContent } from "../context/ContentContext.jsx";
import { useFavorites } from "../context/FavoritesContext.jsx";
import LessonList from "../components/LessonList.jsx";
import NotFoundPage from "./NotFoundPage.jsx";

function ModulePage() {
    const { moduleSlug } = useParams();
    const navigate = useNavigate();
    const { getModuleBySlug, getTrackBySlug } = useContent();
    const { isFavorite, toggleFavorite } = useFavorites();

    const module = getModuleBySlug(moduleSlug);

    // Альтернатива редиректу: отрисовать 404 прямо здесь,
    // сохранив адрес в строке браузера
    if (!module) {
        return <NotFoundPage />;
    }

    const track = getTrackBySlug(module.trackSlug);
    const totalMinutes = module.lessons.reduce((sum, lesson) => sum + lesson.minutes, 0);
    const favorite = isFavorite(module.id);

    return (
        <section>
            {/* navigate(-1) — шаг назад по истории браузера */}
            <button type="button" className="back-link" onClick={() => navigate(-1)}>
                ← Назад
            </button>

            <h1>{module.title}</h1>

            <p className="lead">
                Направление: <Link to={`/tracks/${track.slug}`}>{track.title}</Link> ·{" "}
                уровень: {module.level} · {totalMinutes} минут
            </p>

            <button
                type="button"
                className={favorite ? "btn btn--active" : "btn"}
                onClick={() => toggleFavorite(module.id)}
            >
                {favorite ? "★ В избранном" : "☆ В избранное"}
            </button>

            <LessonList lessons={module.lessons} />
        </section>
    );
}

export default ModulePage;
```

## src/components/LessonList.jsx

```jsx
function LessonList({ lessons }) {
    return (
        <ol className="lesson-list">
            {lessons.map((lesson) => (
                <li key={lesson.id} className="lesson">
                    <span>{lesson.title}</span>
                    <span className="muted">{lesson.minutes} мин</span>
                </li>
            ))}
        </ol>
    );
}

export default LessonList;
```

## src/pages/FavoritesPage.jsx

```jsx
import { Link } from "react-router-dom";
import { useContent } from "../context/ContentContext.jsx";
import { useFavorites } from "../context/FavoritesContext.jsx";

function FavoritesPage() {
    const { modules } = useContent();
    const { favoriteIds, clearFavorites, toggleFavorite } = useFavorites();

    const favoriteModules = modules.filter((module) => favoriteIds.includes(module.id));

    return (
        <section>
            <h1>Избранное</h1>

            {favoriteModules.length === 0 ? (
                <p className="empty">
                    Пока пусто. <Link to="/tracks">Выберите модуль</Link>
                </p>
            ) : (
                <>
                    <button type="button" className="btn" onClick={clearFavorites}>
                        Очистить всё
                    </button>

                    <div className="grid">
                        {favoriteModules.map((module) => (
                            <article key={module.id} className="card">
                                <Link to={`/modules/${module.slug}`}>
                                    <h3>{module.title}</h3>
                                </Link>
                                <span className="badge">{module.level}</span>
                                <button
                                    type="button"
                                    className="btn btn--small"
                                    onClick={() => toggleFavorite(module.id)}
                                >
                                    Убрать
                                </button>
                            </article>
                        ))}
                    </div>
                </>
            )}
        </section>
    );
}

export default FavoritesPage;
```

## src/pages/AboutPage.jsx

```jsx
function AboutPage() {
    return (
        <section>
            <h1>О проекте</h1>
            <p>
                ProfiHub — учебная база знаний для подготовки к чемпионату
                по блокчейн-разработке. Материалы разделены по направлениям и модулям.
            </p>
        </section>
    );
}

export default AboutPage;
```

## src/pages/NotFoundPage.jsx

```jsx
import { Link } from "react-router-dom";

function NotFoundPage() {
    return (
        <section className="not-found">
            <h1>404</h1>
            <p>Такой страницы нет. Возможно, адрес введён с ошибкой.</p>
            <Link to="/" className="btn btn--primary">На главную</Link>
        </section>
    );
}

export default NotFoundPage;
```

## Разбор решения

1. **`pages/` против `components/`.** В `pages/` лежит то, что напрямую сопоставлено маршруту и отвечает за целую экранную форму: собирает данные и компонует блоки. В `components/` — переиспользуемые куски интерфейса, которые ничего не знают о роутинге. Разделение помогает сразу видеть, сколько у приложения экранов.
2. **`BrowserRouter` против `HashRouter`.** Первый использует History API и даёт чистые адреса `/tracks/solidity`, но требует, чтобы сервер отдавал `index.html` на любой путь — иначе прямое открытие адреса вернёт 404. Второй пишет адрес после решётки (`/#/tracks/solidity`) и работает на любом статическом хостинге без настройки.
3. **Вложенные маршруты и `Outlet`.** Родительский `Route` с `element={<Layout />}` задаёт каркас: шапка и подвал рендерятся один раз и не перемонтируются при переходах. `Outlet` — место, куда React Router подставляет текущий дочерний маршрут. Без вложенности `Navbar` пришлось бы дублировать в каждой странице.
4. **`index` вместо `path=""`.** Маршрут `index` срабатывает, когда адрес точно совпадает с путём родителя. Это идиоматичный способ задать «главную» внутри общего каркаса.
5. **`Link` вместо `<a href>`.** Обычная ссылка перезагружает документ и стирает всё состояние React. `Link` перехватывает клик и меняет адрес через History API — приложение остаётся живым, во вкладке Network нет запроса документа.
6. **`NavLink` и `end`.** `NavLink` сам сообщает, активна ли ссылка, через аргумент `isActive` в `className`. Флаг `end` нужен главной: путь `/` является префиксом всех остальных, и без `end` она подсвечивалась бы всегда.
7. **Динамические параметры.** Сегмент `:trackSlug` в `path` превращается в ключ объекта `useParams()`. Значение всегда строка — если в маршруте передан числовой id, его нужно приводить через `Number()`.
8. **`useNavigate` против `Link`.** `Link` — для навигации по клику пользователя. `useNavigate` — для навигации из кода: после отправки формы, при редиректе, для `navigate(-1)` (шаг назад по истории). Флаг `replace` заменяет текущую запись в истории вместо добавления новой — так несуществующий адрес не остаётся в истории.
9. **`useSearchParams` для фильтров.** Состояние фильтров живёт в адресной строке, а не в `useState`. Поэтому ссылку `/tracks?q=erc&level=средний` можно отправить коллеге или обновить страницу, не потеряв выбор. Пустые значения из URL удаляются, чтобы адрес оставался читаемым.
10. **Порядок маршрутов и `*`.** Маршрут `path="*"` совпадает с любым адресом, поэтому ставится последним. React Router v6 выбирает наиболее специфичное совпадение, но держать `*` в конце — правило, которое избавляет от неожиданностей.
11. **Context решает prop drilling.** Счётчик избранного нужен `Navbar`, а состояние живёт в провайдере на самом верху. Без контекста значение пришлось бы пробрасывать через `App` и `Layout`, которые сами по себе об избранном ничего не знают. Контекст даёт прямой доступ из любого места дерева.
12. **Зачем `useMemo` и `useCallback` в провайдере.** Значение контекста — объект. Если создавать его заново на каждом рендере, ссылка будет меняться постоянно, и React перерисует всех потребителей, даже когда данные фактически те же. `useMemo` фиксирует объект, а `useCallback` — функции внутри него, иначе мемоизация объекта не имела бы смысла.
13. **Собственный хук вместо голого `useContext`.** `useContent()` прячет детали реализации и проверяет, что вызов происходит внутри провайдера. Ошибка «useContent должен использоваться внутри ContentProvider» гораздо полезнее, чем падение на чтении свойства у `null` где-то в глубине компонента.
14. **Два контекста, а не один.** Содержимое базы знаний статично, избранное меняется часто. Если положить их в один провайдер, каждое нажатие «в избранное» перерисовывало бы всех потребителей контента. Разделение по частоте изменений — базовый приём оптимизации работы с контекстом.

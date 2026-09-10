# Что такое React Router DOM

**react-router-dom** — библиотека, которая добавляет в React-приложение маршрутизацию: возможность показывать разные компоненты в зависимости от текущего URL-адреса в браузере, без полной перезагрузки страницы.

## Основные компоненты библиотеки

| Компонент/хук | Назначение |
|---|---|
| `BrowserRouter` | Оборачивает всё приложение, включает роутинг на основе истории браузера |
| `Routes` | Контейнер, внутри которого перечисляются все возможные маршруты |
| `Route` | Один конкретный маршрут: связка "путь → компонент" |
| `Link` | Замена обычного `<a>` — переход между страницами без перезагрузки |
| `useNavigate` | Хук для программного перехода на другую страницу (например, после отправки формы) |
| `useParams` | Хук для получения динамических частей URL (например, `/product/5` → `id = 5`) |

## Базовая настройка

**main.jsx**
```jsx
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import { BrowserRouter } from 'react-router-dom'
import App from './App.jsx'
import './index.css'

createRoot(document.getElementById('root')).render(
    <StrictMode>
        {/* BrowserRouter оборачивает всё приложение один раз, в самом корне */}
        <BrowserRouter>
            <App />
        </BrowserRouter>
    </StrictMode>,
)
```

**App.jsx**
```jsx
import { Routes, Route } from 'react-router-dom'
import HomePage from './pages/HomePage.jsx'
import AboutPage from './pages/AboutPage.jsx'

function App() {
    return (
        <Routes>
            {/* path — адрес в браузере, element — что показывать по этому адресу */}
            <Route path="/" element={<HomePage />} />
            <Route path="/about" element={<AboutPage />} />
        </Routes>
    );
}

export default App;
```

## Переход между страницами через Link
```jsx
import { Link } from 'react-router-dom'

function Navbar() {
    return (
        <nav>
            <Link to="/">Главная</Link>
            <Link to="/about">О нас</Link>
        </nav>
    );
}
```
`Link` рендерится в обычный `<a>` тег в итоговом HTML, но перехватывает клик и меняет страницу через JavaScript, **без** полной перезагрузки — приложение остаётся SPA, просто внутри неё меняется отображаемый компонент.

*Важно: обычный `<a href="/about">` тоже сработает, но вызовет полную перезагрузку страницы браузером — весь смысл SPA-роутинга в этом случае теряется. Внутри React-приложения для переходов между "страницами" всегда используется `Link` (или `useNavigate`), а не обычный `<a>`.*

## Динамические маршруты
```jsx
<Route path="/product/:id" element={<ProductPage />} />
```
```jsx
import { useParams } from 'react-router-dom'

function ProductPage() {
    const { id } = useParams(); // достаём id прямо из URL
    return <h1>Товар №{id}</h1>;
}
```
Если открыть `/product/5`, компонент `ProductPage` получит `id === "5"` через `useParams()`.

## Маршрут "страница не найдена" (404)
```jsx
<Routes>
    <Route path="/" element={<HomePage />} />
    <Route path="/about" element={<AboutPage />} />

    {/* "*" ловит абсолютно любой путь, не подошедший под предыдущие маршруты */}
    <Route path="*" element={<NotFoundPage />} />
</Routes>
```

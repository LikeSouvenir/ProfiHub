# Изменения в файловой системе. Создание новой страницы

## Как меняется структура проекта с появлением роутинга
До роутинга весь интерфейс обычно помещался в один `App.jsx`. С появлением нескольких "страниц" в проекте принято выделять отдельную папку `pages/` — по одному файлу на каждую страницу приложения:

```
src/
├── components/         # Переиспользуемые кусочки интерфейса (кнопки, карточки, навигация)
│   ├── Navbar.jsx
│   └── ProductCard.jsx
├── pages/               # Компоненты, каждый из которых — отдельная "страница" приложения
│   ├── HomePage.jsx
│   ├── AboutPage.jsx
│   └── NotFoundPage.jsx
├── context/             # Контексты приложения (см. следующий конспект)
│   └── ProductsContext.jsx
├── App.jsx              # Здесь описываются маршруты (Routes/Route)
└── main.jsx             # Здесь подключается BrowserRouter
```

**Разница между `components/` и `pages/`:**
- `components/` — небольшие, переиспользуемые кусочки UI, которые могут встречаться на нескольких страницах сразу (например, `Navbar` показывается и на главной, и на странице "О нас").
- `pages/` — компоненты верхнего уровня, каждый из которых соответствует ровно одному маршруту (`path`) в `Routes`, и обычно сам собирает более мелкие компоненты внутри себя.

## Создание новой страницы: пошагово

### Шаг 1. Создать файл в pages/
```jsx
// src/pages/AboutPage.jsx
function AboutPage() {
    return (
        <div className="about-page">
            <h1>О нас</h1>
            <p>Мы делаем обучающую платформу для будущих разработчиков.</p>
        </div>
    );
}

export default AboutPage;
```

### Шаг 2. Подключить страницу в App.jsx через маршрут
```jsx
import { Routes, Route } from 'react-router-dom'
import HomePage from './pages/HomePage.jsx'
import AboutPage from './pages/AboutPage.jsx'

function App() {
    return (
        <Routes>
            <Route path="/" element={<HomePage />} />
            <Route path="/about" element={<AboutPage />} />
        </Routes>
    );
}

export default App;
```

### Шаг 3. Добавить ссылку на новую страницу в навигацию
```jsx
// src/components/Navbar.jsx
import { Link } from 'react-router-dom'

function Navbar() {
    return (
        <nav className="navbar">
            <Link to="/">Главная</Link>
            <Link to="/about">О нас</Link>
        </nav>
    );
}

export default Navbar;
```

### Шаг 4. Разместить Navbar так, чтобы он был виден на всех страницах
```jsx
// App.jsx
import { Routes, Route } from 'react-router-dom'
import Navbar from './components/Navbar.jsx'
import HomePage from './pages/HomePage.jsx'
import AboutPage from './pages/AboutPage.jsx'

function App() {
    return (
        <>
            {/* Navbar рендерится один раз, вне Routes — поэтому виден на любой странице */}
            <Navbar />

            <Routes>
                <Route path="/" element={<HomePage />} />
                <Route path="/about" element={<AboutPage />} />
            </Routes>
        </>
    );
}

export default App;
```

Именно такое разделение — общие элементы (навигация, футер) выносятся **за пределы** `Routes`, а то, что должно меняться в зависимости от URL, помещается **внутрь** `Routes` — и есть основной принцип построения многостраничных React-приложений.

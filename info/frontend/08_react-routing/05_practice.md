# Практика: веб-приложение магазина с категориями и Context

## Условие
Создать веб-приложение магазина. На основной странице — карточки с товарами. Информацию о товарах хранить в контексте и выводить пользователю. Разбить товары по категориям и отображать всё по категории.

## Структура проекта
```
src/
├── components/
│   ├── Navbar.jsx
│   └── ProductCard.jsx
├── context/
│   └── ProductsContext.jsx
├── pages/
│   ├── HomePage.jsx
│   └── CategoryPage.jsx
├── App.jsx
└── main.jsx
```

## context/ProductsContext.jsx
```jsx
import { createContext } from 'react'

export const ProductsContext = createContext(null);

export function ProductsProvider({ children }) {
    // Все товары магазина с указанием категории — хранятся централизованно, в одном месте
    const products = [
        { id: 1, name: "Клавиатура механическая", category: "Электроника", price: 45 },
        { id: 2, name: "Наушники беспроводные", category: "Электроника", price: 60 },
        { id: 3, name: "Роман 'Мастер и Маргарита'", category: "Книги", price: 12 },
        { id: 4, name: "Сборник рассказов Чехова", category: "Книги", price: 9 },
        { id: 5, name: "Кроссовки беговые", category: "Одежда", price: 70 },
        { id: 6, name: "Куртка демисезонная", category: "Одежда", price: 120 },
    ];

    // Список уникальных категорий, вычисленный из массива товаров
    const categories = [...new Set(products.map(p => p.category))];

    return (
        <ProductsContext.Provider value={{ products, categories }}>
            {children}
        </ProductsContext.Provider>
    );
}
```

## components/ProductCard.jsx
```jsx
function ProductCard({ product }) {
    return (
        <div className="product-card">
            <h3 className="product-name">{product.name}</h3>
            <p className="product-price">{product.price} $</p>
            <span className="product-category">{product.category}</span>
        </div>
    );
}

export default ProductCard;
```

## components/Navbar.jsx
```jsx
import { useContext } from 'react'
import { Link } from 'react-router-dom'
import { ProductsContext } from '../context/ProductsContext.jsx'

function Navbar() {
    // Получаем список категорий из контекста, чтобы построить по нему навигацию
    const { categories } = useContext(ProductsContext);

    return (
        <nav className="navbar">
            <Link to="/">Все товары</Link>

            {categories.map(category => (
                <Link key={category} to={`/category/${category}`}>
                    {category}
                </Link>
            ))}
        </nav>
    );
}

export default Navbar;
```

## pages/HomePage.jsx — все товары, сгруппированные по категориям
```jsx
import { useContext } from 'react'
import { ProductsContext } from '../context/ProductsContext.jsx'
import ProductCard from '../components/ProductCard.jsx'

function HomePage() {
    const { products, categories } = useContext(ProductsContext);

    return (
        <div className="home-page">
            <h1>Каталог товаров</h1>

            {categories.map(category => (
                <section key={category} className="category-section">
                    <h2>{category}</h2>

                    <div className="products-grid">
                        {products
                            .filter(product => product.category === category)
                            .map(product => (
                                <ProductCard key={product.id} product={product} />
                            ))}
                    </div>
                </section>
            ))}
        </div>
    );
}

export default HomePage;
```

## pages/CategoryPage.jsx — товары только одной категории (по маршруту)
```jsx
import { useContext } from 'react'
import { useParams } from 'react-router-dom'
import { ProductsContext } from '../context/ProductsContext.jsx'
import ProductCard from '../components/ProductCard.jsx'

function CategoryPage() {
    const { products } = useContext(ProductsContext);
    const { categoryName } = useParams(); // берём имя категории прямо из URL

    const filteredProducts = products.filter(
        product => product.category === categoryName
    );

    return (
        <div className="category-page">
            <h1>Категория: {categoryName}</h1>

            <div className="products-grid">
                {filteredProducts.length > 0 ? (
                    filteredProducts.map(product => (
                        <ProductCard key={product.id} product={product} />
                    ))
                ) : (
                    <p>В этой категории пока нет товаров</p>
                )}
            </div>
        </div>
    );
}

export default CategoryPage;
```

## App.jsx
```jsx
import { Routes, Route } from 'react-router-dom'
import Navbar from './components/Navbar.jsx'
import HomePage from './pages/HomePage.jsx'
import CategoryPage from './pages/CategoryPage.jsx'
import './App.css'

function App() {
    return (
        <>
            <Navbar />

            <Routes>
                <Route path="/" element={<HomePage />} />
                <Route path="/category/:categoryName" element={<CategoryPage />} />
            </Routes>
        </>
    );
}

export default App;
```

## main.jsx
```jsx
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import { BrowserRouter } from 'react-router-dom'
import { ProductsProvider } from './context/ProductsContext.jsx'
import App from './App.jsx'
import './index.css'

createRoot(document.getElementById('root')).render(
    <StrictMode>
        <BrowserRouter>
            <ProductsProvider>
                <App />
            </ProductsProvider>
        </BrowserRouter>
    </StrictMode>,
)
```

## App.css (базовая стилизация карточек и сетки)
```css
.navbar {
    display: flex;
    gap: 15px;
    padding: 15px 30px;
    background-color: #2c3e50;
}

.navbar a {
    color: white;
    text-decoration: none;
}

.category-section {
    margin: 20px 30px;
}

.products-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
    gap: 16px;
}

.product-card {
    padding: 16px;
    border-radius: 10px;
    background-color: #f8f9fa;
    box-shadow: 0 2px 6px rgba(0, 0, 0, 0.08);
}

.product-name {
    margin: 0 0 8px 0;
    font-size: 16px;
}

.product-price {
    font-weight: bold;
    color: #2c3e50;
}

.product-category {
    font-size: 12px;
    color: #888;
}
```

## Разбор решения
1. **`ProductsContext` + `ProductsProvider`** хранят весь список товаров и вычисленный список уникальных категорий (через `new Set`) в одном месте, доступном любому компоненту через `useContext` — без передачи `products` вручную через `props` на каждом уровне.
2. **`HomePage`** проходит по списку `categories`, и для каждой категории рендерит отдельный `<section>` с заголовком-названием категории и отфильтрованными (`.filter`) товарами именно этой категории — так реализуется требование "разбить товар по категориям и отображать всё по категории" прямо на главной странице.
3. **`CategoryPage`** — отдельный маршрут `/category/:categoryName`, использующий `useParams()` для получения названия категории напрямую из адресной строки, и тот же `products` из контекста, отфильтрованный уже по конкретной категории — позволяет открыть, например, `/category/Книги` и увидеть только книги.
4. **`Navbar`** тоже берёт данные из того же самого контекста (`categories`), автоматически строя ссылки на все категории — если позже добавить новый товар с новой категорией в `ProductsProvider`, ссылка на неё в навигации появится сама, без ручного изменения `Navbar`.

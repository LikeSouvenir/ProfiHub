# Практика: магазин на fakestoreapi.com с сортировкой

## Условие
Создать веб-приложение магазина, заполнить данными из API по ссылке `https://fakestoreapi.com/products`, разбить данные по категориям так, чтобы каждая категория была на отдельной странице, а также добавить сортировку по алфавиту и по цене.

## О самом API
`GET https://fakestoreapi.com/products` возвращает JSON-массив товаров вида:
```json
[
    {
        "id": 1,
        "title": "Fjallraven Backpack",
        "price": 109.95,
        "description": "...",
        "category": "men's clothing",
        "image": "https://fakestoreapi.com/img/....jpg",
        "rating": { "rate": 3.9, "count": 120 }
    }
]
```

## Структура проекта
```
src/
├── components/
│   ├── AppNavbar.jsx
│   ├── ProductCard.jsx
│   └── SortControl.jsx
├── context/
│   └── ProductsContext.jsx
├── pages/
│   └── CategoryPage.jsx
├── App.jsx
└── main.jsx
```

## Установка библиотек
```bash
npm install react-bootstrap bootstrap react-router-dom
```

## context/ProductsContext.jsx — загрузка данных из API
```jsx
import { createContext, useState, useEffect } from 'react'

export const ProductsContext = createContext(null);

export function ProductsProvider({ children }) {
    const [products, setProducts] = useState([]);
    const [loading, setLoading] = useState(true);

    useEffect(() => {
        // Запрос к API выполняется один раз, при монтировании провайдера
        fetch("https://fakestoreapi.com/products")
            .then(response => response.json())
            .then(data => {
                setProducts(data);
                setLoading(false);
            })
            .catch(error => {
                console.error("Ошибка загрузки товаров:", error);
                setLoading(false);
            });
    }, []);

    // Уникальные категории вычисляются из полученных с сервера товаров
    const categories = [...new Set(products.map(p => p.category))];

    return (
        <ProductsContext.Provider value={{ products, categories, loading }}>
            {children}
        </ProductsContext.Provider>
    );
}
```

## components/AppNavbar.jsx — навигация по категориям
```jsx
import { useContext } from 'react'
import { LinkContainer } from 'react-router-bootstrap'
import Navbar from 'react-bootstrap/Navbar'
import Nav from 'react-bootstrap/Nav'
import { ProductsContext } from '../context/ProductsContext.jsx'

function AppNavbar() {
    const { categories } = useContext(ProductsContext);

    return (
        <Navbar bg="dark" variant="dark" expand="lg" className="mb-4">
            <Navbar.Brand className="ms-3">FakeShop</Navbar.Brand>
            <Navbar.Toggle aria-controls="main-nav" />
            <Navbar.Collapse id="main-nav">
                <Nav>
                    {categories.map(category => (
                        <LinkContainer key={category} to={`/category/${category}`}>
                            <Nav.Link>{category}</Nav.Link>
                        </LinkContainer>
                    ))}
                </Nav>
            </Navbar.Collapse>
        </Navbar>
    );
}

export default AppNavbar;
```
*Если библиотека `react-router-bootstrap` не установлена, `Nav.Link` можно заменить на обычный `Link` из `react-router-dom`, обернув его в `as={Link} to={...}`.*

## components/SortControl.jsx — выбор сортировки
```jsx
import Form from 'react-bootstrap/Form'

function SortControl({ sortBy, onChange }) {
    return (
        <Form.Select
            value={sortBy}
            onChange={(e) => onChange(e.target.value)}
            style={{ maxWidth: '250px' }}
            className="mb-4"
        >
            <option value="none">Без сортировки</option>
            <option value="alphabet">По алфавиту (А-Я)</option>
            <option value="price-asc">По цене (сначала дешёвые)</option>
            <option value="price-desc">По цене (сначала дорогие)</option>
        </Form.Select>
    );
}

export default SortControl;
```

## components/ProductCard.jsx
```jsx
import Card from 'react-bootstrap/Card'

function ProductCard({ product }) {
    return (
        <Card className="h-100">
            <Card.Img
                variant="top"
                src={product.image}
                style={{ height: '200px', objectFit: 'contain', padding: '10px' }}
            />
            <Card.Body className="d-flex flex-column">
                <Card.Title style={{ fontSize: '16px' }}>{product.title}</Card.Title>
                <Card.Text className="mt-auto fw-bold">{product.price} $</Card.Text>
            </Card.Body>
        </Card>
    );
}

export default ProductCard;
```

## pages/CategoryPage.jsx — страница категории с сортировкой
```jsx
import { useContext, useState, useMemo } from 'react'
import { useParams } from 'react-router-dom'
import Container from 'react-bootstrap/Container'
import Row from 'react-bootstrap/Row'
import Col from 'react-bootstrap/Col'
import { ProductsContext } from '../context/ProductsContext.jsx'
import ProductCard from '../components/ProductCard.jsx'
import SortControl from '../components/SortControl.jsx'

function CategoryPage() {
    const { products, loading } = useContext(ProductsContext);
    const { categoryName } = useParams();
    const [sortBy, setSortBy] = useState("none");

    // useMemo — пересчитываем отсортированный список только тогда,
    // когда реально меняются products, categoryName или sortBy
    const sortedProducts = useMemo(() => {
        const filtered = products.filter(p => p.category === categoryName);

        // Создаём копию массива перед сортировкой — sort мутирует исходный массив
        const sorted = [...filtered];

        if (sortBy === "alphabet") {
            sorted.sort((a, b) => a.title.localeCompare(b.title));
        } else if (sortBy === "price-asc") {
            sorted.sort((a, b) => a.price - b.price);
        } else if (sortBy === "price-desc") {
            sorted.sort((a, b) => b.price - a.price);
        }

        return sorted;
    }, [products, categoryName, sortBy]);

    if (loading) {
        return <Container><p>Загрузка товаров...</p></Container>;
    }

    return (
        <Container>
            <h1 className="text-capitalize mb-3">{categoryName}</h1>

            <SortControl sortBy={sortBy} onChange={setSortBy} />

            <Row xs={1} md={3} className="g-4">
                {sortedProducts.map(product => (
                    <Col key={product.id}>
                        <ProductCard product={product} />
                    </Col>
                ))}
            </Row>
        </Container>
    );
}

export default CategoryPage;
```

## App.jsx
```jsx
import { Routes, Route, Navigate } from 'react-router-dom'
import AppNavbar from './components/AppNavbar.jsx'
import CategoryPage from './pages/CategoryPage.jsx'

function App() {
    return (
        <>
            <AppNavbar />
            <Routes>
                {/* Главная страница перенаправляет на первую доступную категорию */}
                <Route path="/" element={<Navigate to="/category/electronics" />} />
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
import 'bootstrap/dist/css/bootstrap.min.css'
import { ProductsProvider } from './context/ProductsContext.jsx'
import App from './App.jsx'

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

## Разбор решения
1. **Загрузка из API** происходит один раз в `ProductsProvider` через `fetch` внутри `useEffect` с пустым массивом зависимостей — полученные данные и вычисленный из них список `categories` становятся доступны всему приложению через контекст, без повторных запросов на каждой странице.
2. **Каждая категория — отдельная страница**, потому что маршрут задан динамически: `/category/:categoryName`. При переходе по ссылке из `AppNavbar` меняется только `categoryName` в URL, а `CategoryPage` заново фильтрует общий список `products` из контекста под конкретную категорию.
3. **Сортировка по алфавиту** использует `localeCompare` — строковый метод, корректно сравнивающий строки с учётом алфавитного порядка (в том числе для не латинских алфавитов).
4. **Сортировка по цене** — сравнение чисел через вычитание (`a.price - b.price` — по возрастанию, `b.price - a.price` — по убыванию), стандартный приём для числового `sort` в JavaScript.
5. **`useMemo`** оборачивает фильтрацию и сортировку, чтобы они пересчитывались только при реальном изменении зависимостей (`products`, `categoryName`, `sortBy`), а не при каждом рендере компонента — что особенно полезно, если список товаров большой.
6. Все визуальные элементы (`Navbar`, `Card`, `Form.Select`, сетка `Row`/`Col`) взяты из **React-Bootstrap**, что даёт готовый адаптивный внешний вид без написания собственного CSS.

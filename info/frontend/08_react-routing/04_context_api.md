# Создание первого Routing + createContext, useContext, ContextProvider

## Проблема: передача данных через много уровней (prop drilling)
Если данные нужны глубоко вложенному компоненту, их приходится передавать через `props` на каждом промежуточном уровне, даже если сам промежуточный компонент эти данные не использует — это называется **prop drilling** и усложняет код при росте приложения:
```jsx
<App>
    <Page products={products}>
        <ProductList products={products}>
            <ProductCard product={products[0]} /> {/* наконец-то тут они реально нужны */}
        </ProductList>
    </Page>
</App>
```

## Что решает Context API
**Context** — встроенный в React механизм, который позволяет "прокинуть" данные сразу в любой компонент дерева, минуя промежуточные уровни, без явной передачи через `props` на каждом шаге.

## Шаг 1. Создание контекста: createContext
```jsx
// src/context/ProductsContext.jsx
import { createContext } from 'react'

// createContext создаёт объект контекста с необязательным значением по умолчанию
export const ProductsContext = createContext(null);
```

## Шаг 2. Провайдер: ContextProvider
Каждый контекст, созданный через `createContext`, автоматически получает встроенный компонент `.Provider` — именно он "раздаёт" значение контекста всем компонентам внутри себя:
```jsx
// src/context/ProductsContext.jsx
import { createContext, useState } from 'react'

export const ProductsContext = createContext(null);

export function ProductsProvider({ children }) {
    const [products] = useState([
        { id: 1, name: "Клавиатура", category: "Электроника", price: 45 },
        { id: 2, name: "Роман 'Мастер и Маргарита'", category: "Книги", price: 12 },
        { id: 3, name: "Кроссовки", category: "Одежда", price: 60 },
    ]);

    // value — это то, что будет доступно всем потомкам через useContext
    return (
        <ProductsContext.Provider value={{ products }}>
            {children}
        </ProductsContext.Provider>
    );
}
```
`children` — специальный проп, через который React передаёт всё, что было вложено внутрь компонента при его использовании (см. пример подключения ниже).

## Шаг 3. Подключение провайдера на верхнем уровне приложения
```jsx
// main.jsx
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import { BrowserRouter } from 'react-router-dom'
import { ProductsProvider } from './context/ProductsContext.jsx'
import App from './App.jsx'

createRoot(document.getElementById('root')).render(
    <StrictMode>
        <BrowserRouter>
            {/* Оборачиваем App в провайдер — теперь любой компонент внутри App
                может получить доступ к products через useContext */}
            <ProductsProvider>
                <App />
            </ProductsProvider>
        </BrowserRouter>
    </StrictMode>,
)
```

## Шаг 4. Чтение данных из контекста: useContext
```jsx
// src/pages/HomePage.jsx
import { useContext } from 'react'
import { ProductsContext } from '../context/ProductsContext.jsx'

function HomePage() {
    // useContext достаёт value, переданное в ближайший .Provider выше по дереву
    const { products } = useContext(ProductsContext);

    return (
        <div>
            <h1>Каталог товаров</h1>
            {products.map(product => (
                <p key={product.id}>{product.name} — {product.price}$</p>
            ))}
        </div>
    );
}

export default HomePage;
```
Важно: `HomePage` получил доступ к `products` **без единого props** между `App` и `HomePage` — контекст полностью решает проблему prop drilling.

## Полная схема первого роутинга с контекстом
```
main.jsx
└── BrowserRouter
    └── ProductsProvider          ← контекст оборачивает всё приложение
        └── App.jsx
            ├── Navbar             (общий для всех страниц)
            └── Routes
                ├── "/"      → HomePage    (читает products через useContext)
                └── "/about" → AboutPage   (контекст products тут не нужен, но доступен)
```

## Когда использовать Context, а когда — обычные props
- **Обычные props** — когда данные нужны только одному конкретному дочернему компоненту, находящемуся рядом.
- **Context** — когда одни и те же данные нужны сразу многим компонентам на разных уровнях вложенности приложения (список товаров, текущий пользователь, выбранная тема оформления и т.д.).

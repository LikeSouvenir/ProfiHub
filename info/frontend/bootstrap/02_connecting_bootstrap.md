# Подключение Bootstrap к веб-приложению

## Установка
В уже существующем React-проекте (Vite):
```bash
npm install react-bootstrap bootstrap
```
Устанавливаются сразу два пакета:
- `bootstrap` — сами CSS-стили Bootstrap (сетка, цвета, отступы, готовые классы).
- `react-bootstrap` — React-компоненты, использующие эти стили.

## Подключение CSS-файла Bootstrap
CSS нужно импортировать один раз, в самой "верхней" точке приложения — обычно в `main.jsx`:
```jsx
// main.jsx
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import 'bootstrap/dist/css/bootstrap.min.css' // подключаем стили Bootstrap
import App from './App.jsx'
import './index.css'

createRoot(document.getElementById('root')).render(
    <StrictMode>
        <App />
    </StrictMode>,
)
```
После этого импорта во всём приложении становятся доступны как готовые CSS-классы Bootstrap (`className="container"`, `className="btn btn-primary"`), так и компоненты из `react-bootstrap`.

## Проверка подключения
```jsx
// App.jsx
import Button from 'react-bootstrap/Button'

function App() {
    return (
        <div className="p-4">
            <Button variant="primary">Кнопка Bootstrap</Button>
        </div>
    );
}

export default App;
```
Если кнопка отображается синей, со скруглёнными углами и характерным для Bootstrap стилем — подключение прошло успешно.

## Два способа импорта компонентов

### Импорт отдельного компонента (рекомендуется)
```jsx
import Button from 'react-bootstrap/Button'
import Card from 'react-bootstrap/Card'
```
Такой способ импортирует только конкретный компонент, что делает финальную сборку приложения немного легче (меньше неиспользуемого кода).

### Импорт из общего пакета
```jsx
import { Button, Card } from 'react-bootstrap'
```
Работает так же, но менее оптимален с точки зрения размера итоговой сборки в некоторых конфигурациях сборщика — на практике для небольших/учебных проектов разница почти не заметна.

## Использование сетки Bootstrap (Container, Row, Col)
Сетка Bootstrap доступна в виде готовых компонентов, повторяющих обычные классы `container`/`row`/`col`:
```jsx
import Container from 'react-bootstrap/Container'
import Row from 'react-bootstrap/Row'
import Col from 'react-bootstrap/Col'

function Layout() {
    return (
        <Container>
            <Row>
                <Col md={4}>Первая колонка</Col>
                <Col md={4}>Вторая колонка</Col>
                <Col md={4}>Третья колонка</Col>
            </Row>
        </Container>
    );
}
```
`md={4}` означает: на средних и более широких экранах эта колонка займёт 4 из 12 условных единиц сетки Bootstrap (то есть треть ширины) — три такие колонки как раз заполнят строку целиком.

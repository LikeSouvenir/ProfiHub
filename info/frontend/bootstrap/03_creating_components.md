# Создание компонентов

## Составные компоненты Bootstrap (dot notation)
Многие компоненты React-Bootstrap состоят из нескольких связанных частей, которые обращаются через точку — например, `Card` и его "подкомпоненты":
```jsx
import Card from 'react-bootstrap/Card'

function ProductCard() {
    return (
        <Card style={{ width: '18rem' }}>
            <Card.Img variant="top" src="product.jpg" />
            <Card.Body>
                <Card.Title>Название товара</Card.Title>
                <Card.Text>
                    Краткое описание товара для покупателя.
                </Card.Text>
                <Card.Footer>150 $</Card.Footer>
            </Card.Body>
        </Card>
    );
}
```

## Кнопки (Button)
```jsx
import Button from 'react-bootstrap/Button'

function Buttons() {
    return (
        <>
            <Button variant="primary">Основная</Button>
            <Button variant="secondary">Вторичная</Button>
            <Button variant="danger">Опасное действие</Button>
            <Button variant="outline-primary">Контурная</Button>
            <Button variant="primary" disabled>Недоступна</Button>
        </>
    );
}
```
`variant` — готовый набор цветовых тем Bootstrap, определяющих внешний вид кнопки без необходимости писать собственный CSS.

## Навигация (Navbar + Nav)
```jsx
import Navbar from 'react-bootstrap/Navbar'
import Nav from 'react-bootstrap/Nav'
import { LinkContainer } from 'react-router-bootstrap' // объединяет react-router-dom и Bootstrap

function AppNavbar() {
    return (
        <Navbar bg="dark" variant="dark" expand="lg">
            <Navbar.Brand href="/">MyShop</Navbar.Brand>
            <Navbar.Toggle aria-controls="main-navbar" />
            <Navbar.Collapse id="main-navbar">
                <Nav className="me-auto">
                    <LinkContainer to="/">
                        <Nav.Link>Главная</Nav.Link>
                    </LinkContainer>
                    <LinkContainer to="/about">
                        <Nav.Link>О нас</Nav.Link>
                    </LinkContainer>
                </Nav>
            </Navbar.Collapse>
        </Navbar>
    );
}
```
`Navbar.Toggle`/`Navbar.Collapse` автоматически реализуют "гамбургер"-меню на узких экранах — без единой строчки написанного вручную JS для мобильной адаптации.

## Формы (Form)
```jsx
import Form from 'react-bootstrap/Form'
import Button from 'react-bootstrap/Button'

function SearchForm() {
    return (
        <Form>
            <Form.Group className="mb-3">
                <Form.Label>Поиск товара</Form.Label>
                <Form.Control type="text" placeholder="Введите название" />
            </Form.Group>

            <Form.Select className="mb-3">
                <option value="">Все категории</option>
                <option value="electronics">Электроника</option>
                <option value="clothing">Одежда</option>
            </Form.Select>

            <Button type="submit">Найти</Button>
        </Form>
    );
}
```

## Сетка карточек товаров (Row + Col + Card)
```jsx
import Container from 'react-bootstrap/Container'
import Row from 'react-bootstrap/Row'
import Col from 'react-bootstrap/Col'
import Card from 'react-bootstrap/Card'

function ProductGrid({ products }) {
    return (
        <Container>
            <Row xs={1} md={3} className="g-4">
                {products.map(product => (
                    <Col key={product.id}>
                        <Card>
                            <Card.Img variant="top" src={product.image} />
                            <Card.Body>
                                <Card.Title>{product.title}</Card.Title>
                                <Card.Text>{product.price} $</Card.Text>
                            </Card.Body>
                        </Card>
                    </Col>
                ))}
            </Row>
        </Container>
    );
}
```
`xs={1} md={3}` — адаптивность прямо через пропсы компонента: одна карточка в строке на мобильных экранах, три — на средних и более широких, без написания медиа-запросов вручную. `g-4` — стандартный Bootstrap-класс отступов между колонками сетки (gap).

## Комбинирование собственных стилей с Bootstrap
Компоненты React-Bootstrap принимают обычные пропсы React (`className`, `style`), поэтому их легко дополнять своими стилями поверх готовых:
```jsx
<Card className="my-custom-card" style={{ borderRadius: '16px' }}>
    ...
</Card>
```
```css
.my-custom-card {
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}
```

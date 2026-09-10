# Взаимодействие с компонентами Bootstrap

Компоненты React-Bootstrap — обычные React-компоненты, поэтому взаимодействие с ними строится на тех же принципах: `useState` для хранения состояния и обработчики событий (`onClick`, `onChange`, `onSubmit`) для реакции на действия пользователя.

## Кнопка + состояние
```jsx
import { useState } from 'react'
import Button from 'react-bootstrap/Button'

function LikeButton() {
    const [liked, setLiked] = useState(false);

    return (
        <Button
            variant={liked ? "danger" : "outline-danger"}
            onClick={() => setLiked(!liked)}
        >
            {liked ? "❤️ Нравится" : "🤍 Добавить в избранное"}
        </Button>
    );
}
```
Здесь `variant` меняется динамически в зависимости от состояния `liked` — внешний вид кнопки полностью управляется React-состоянием, а не статичными CSS-классами.

## Форма: controlled-компоненты (Form.Control)
Как и обычный `<input>`, `Form.Control` из React-Bootstrap можно сделать **управляемым (controlled)** — его значение полностью контролируется состоянием React:
```jsx
import { useState } from 'react'
import Form from 'react-bootstrap/Form'

function SearchInput() {
    const [query, setQuery] = useState("");

    return (
        <Form.Control
            type="text"
            placeholder="Поиск..."
            value={query}                              // значение берётся из state
            onChange={(e) => setQuery(e.target.value)} // при каждом вводе — обновляем state
        />
    );
}
```
`e.target.value` — стандартное свойство события ввода, содержащее текущий текст поля (то же самое поведение, что и у обычного HTML `<input>`).

## Выпадающий список: Form.Select
```jsx
import { useState } from 'react'
import Form from 'react-bootstrap/Form'

function CategoryFilter({ categories, onSelect }) {
    const [selected, setSelected] = useState("all");

    function handleChange(e) {
        const value = e.target.value;
        setSelected(value);
        onSelect(value); // сообщаем родительскому компоненту о выборе
    }

    return (
        <Form.Select value={selected} onChange={handleChange}>
            <option value="all">Все категории</option>
            {categories.map(category => (
                <option key={category} value={category}>
                    {category}
                </option>
            ))}
        </Form.Select>
    );
}
```

## Модальное окно (Modal): открытие/закрытие через состояние
```jsx
import { useState } from 'react'
import Button from 'react-bootstrap/Button'
import Modal from 'react-bootstrap/Modal'

function ProductDetails({ product }) {
    const [show, setShow] = useState(false);

    return (
        <>
            <Button onClick={() => setShow(true)}>Подробнее</Button>

            <Modal show={show} onHide={() => setShow(false)}>
                <Modal.Header closeButton>
                    <Modal.Title>{product.title}</Modal.Title>
                </Modal.Header>
                <Modal.Body>{product.description}</Modal.Body>
                <Modal.Footer>
                    <Button variant="secondary" onClick={() => setShow(false)}>
                        Закрыть
                    </Button>
                </Modal.Footer>
            </Modal>
        </>
    );
}
```
`show` — булево состояние React, полностью управляющее видимостью модального окна; `onHide` — обработчик, который React-Bootstrap вызывает сам при клике вне окна или на крестик закрытия.

## Форма отправки (Form + onSubmit)
```jsx
import { useState } from 'react'
import Form from 'react-bootstrap/Form'
import Button from 'react-bootstrap/Button'

function ContactForm() {
    const [email, setEmail] = useState("");

    function handleSubmit(e) {
        e.preventDefault(); // отменяем стандартную перезагрузку страницы браузером
        console.log("Отправлен email:", email);
    }

    return (
        <Form onSubmit={handleSubmit}>
            <Form.Group className="mb-3">
                <Form.Label>Email</Form.Label>
                <Form.Control
                    type="email"
                    value={email}
                    onChange={(e) => setEmail(e.target.value)}
                    required
                />
            </Form.Group>
            <Button type="submit">Отправить</Button>
        </Form>
    );
}
```
`e.preventDefault()` необходим здесь так же, как и в обычном HTML/JS — без него браузер попытается сам отправить и перезагрузить страницу, что для SPA-приложения на React нежелательно.

## Итог
Компоненты React-Bootstrap не отличаются по принципу работы от обычных HTML-элементов внутри React: та же связка `useState` + `value`/`onChange` для форм, обычные обработчики `onClick` для кнопок, и обычные пропсы для условного управления внешним видом (`variant`, `show` и т.д.) — Bootstrap лишь добавляет готовую стилизацию и удобные составные компоненты поверх привычных React-паттернов.

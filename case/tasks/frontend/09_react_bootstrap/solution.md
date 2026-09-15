# Модуль 9: React-Bootstrap — Эталонное решение

## src/main.jsx

```jsx
import React from "react";
import ReactDOM from "react-dom/client";
import { BrowserRouter } from "react-router-dom";
import { TasksProvider } from "./context/TasksContext.jsx";
import App from "./App.jsx";

// Порядок импортов важен: сначала Bootstrap, затем собственные стили.
// CSS применяется в порядке подключения, и наши правила должны
// идти последними, чтобы перекрывать стандартные при совпадении веса
import "bootstrap/dist/css/bootstrap.min.css";
import "./App.css";

ReactDOM.createRoot(document.getElementById("root")).render(
    <React.StrictMode>
        <TasksProvider>
            <BrowserRouter>
                <App />
            </BrowserRouter>
        </TasksProvider>
    </React.StrictMode>
);
```

**Почему `react-bootstrap`, а не `bootstrap.bundle.js`.** Обычный Bootstrap управляет компонентами императивно: атрибуты `data-bs-toggle` навешивают обработчики на DOM-узлы напрямую. React же сам владеет DOM и перерисовывает его при изменении состояния — сторонний скрипт, который параллельно меняет те же узлы, приводит к рассинхронизации: модалка «залипает» открытой, дропдаун не закрывается после перерисовки. `react-bootstrap` переписывает те же компоненты на React, и их видимость управляется обычным состоянием (`show`, `onHide`), то есть теми же правилами, что и остальной интерфейс.

**Два способа импорта.**

```jsx
import { Button, Card } from "react-bootstrap";   // читается удобнее
import Button from "react-bootstrap/Button";       // точечный импорт
```

Современные сборщики (Vite, Webpack 5) применяют tree-shaking, и на размер бандла оба варианта влияют одинаково. Точечный импорт остаётся полезен при старых сборщиках без tree-shaking и слегка ускоряет холодный старт дев-сервера.

## src/context/TasksContext.jsx

```jsx
import { createContext, useContext, useState, useEffect, useMemo, useCallback } from "react";

const TasksContext = createContext(null);

const API_URL = "https://fakestoreapi.com/products";

export function TasksProvider({ children }) {
    const [tasks, setTasks] = useState([]);
    const [isLoading, setIsLoading] = useState(true);
    const [error, setError] = useState(null);
    const [selected, setSelected] = useState([]);
    const [reloadKey, setReloadKey] = useState(0);

    useEffect(() => {
        let cancelled = false;

        // Асинхронная функция объявляется внутри: сам useEffect
        // не может быть async — он обязан вернуть либо функцию
        // очистки, либо ничего, а async всегда возвращает промис
        async function loadTasks() {
            setIsLoading(true);
            setError(null);

            try {
                const response = await fetch(API_URL);

                // fetch не бросает ошибку на статусах 4xx и 5xx —
                // проверять response.ok нужно вручную
                if (!response.ok) {
                    throw new Error(`Сервер ответил ${response.status}`);
                }

                const data = await response.json();
                if (!cancelled) setTasks(data);
            } catch (err) {
                if (!cancelled) setError(err.message);
            } finally {
                if (!cancelled) setIsLoading(false);
            }
        }

        loadTasks();

        return () => {
            cancelled = true; // компонент размонтирован — setState не вызываем
        };
    }, [reloadKey]);

    const reload = useCallback(() => setReloadKey((prev) => prev + 1), []);

    const toggleSelect = useCallback((id) => {
        setSelected((prev) =>
            prev.includes(id) ? prev.filter((item) => item !== id) : [...prev, id]
        );
    }, []);

    const isSelected = useCallback((id) => selected.includes(id), [selected]);
    const clearSelected = useCallback(() => setSelected([]), []);

    // Список категорий выводится из данных, а не хранится отдельно.
    // Set убирает дубликаты, spread возвращает его обратно в массив
    const categories = useMemo(
        () => [...new Set(tasks.map((task) => task.category))],
        [tasks]
    );

    const value = useMemo(
        () => ({
            tasks, categories, isLoading, error, reload,
            selected, toggleSelect, isSelected, clearSelected,
        }),
        [tasks, categories, isLoading, error, reload, selected, toggleSelect, isSelected, clearSelected]
    );

    return <TasksContext.Provider value={value}>{children}</TasksContext.Provider>;
}

export function useTasks() {
    const context = useContext(TasksContext);

    if (context === null) {
        throw new Error("useTasks должен использоваться внутри <TasksProvider>");
    }

    return context;
}
```

## src/components/AppNavbar.jsx

```jsx
import { Navbar, Nav, Container, Form, Button, Badge } from "react-bootstrap";
import { Link, NavLink, useNavigate } from "react-router-dom";
import { useState } from "react";
import { useTasks } from "../context/TasksContext.jsx";

function AppNavbar() {
    const { categories, selected } = useTasks();
    const [query, setQuery] = useState("");
    const navigate = useNavigate();

    const handleSearch = (event) => {
        event.preventDefault();
        navigate(`/?q=${encodeURIComponent(query.trim())}`);
    };

    return (
        // expand="lg": на экранах уже lg меню сворачивается в бургер.
        // Toggle и Collapse работают на состоянии React, без JS-бандла Bootstrap
        <Navbar bg="dark" variant="dark" expand="lg" sticky="top">
            <Container>
                {/* as={Link} подменяет внутренний тег компонента.
                    Без него Navbar.Brand отрендерит обычный <a href>,
                    и клик перезагрузит всё приложение. С as={Link}
                    навигация идёт через роутер, стили Bootstrap сохраняются */}
                <Navbar.Brand as={Link} to="/">
                    ProfiHub<span className="text-info">Store</span>
                </Navbar.Brand>

                <Navbar.Toggle aria-controls="main-nav" />

                <Navbar.Collapse id="main-nav">
                    <Nav className="me-auto">
                        <Nav.Link as={NavLink} to="/" end>
                            Все задания
                        </Nav.Link>

                        {categories.map((category) => (
                            <Nav.Link
                                key={category}
                                as={NavLink}
                                to={`/category/${encodeURIComponent(category)}`}
                            >
                                {category}
                            </Nav.Link>
                        ))}
                    </Nav>

                    {/* d-flex — утилитарный класс Bootstrap: выстраивает
                        поле и кнопку в строку без собственного CSS */}
                    <Form className="d-flex me-3" onSubmit={handleSearch}>
                        <Form.Control
                            type="search"
                            placeholder="Поиск задания"
                            className="me-2"
                            value={query}
                            onChange={(event) => setQuery(event.target.value)}
                        />
                        <Button variant="outline-info" type="submit">
                            Найти
                        </Button>
                    </Form>

                    <Nav.Link as={NavLink} to="/cart">
                        Выбрано{" "}
                        <Badge bg="info" pill>
                            {selected.length}
                        </Badge>
                    </Nav.Link>
                </Navbar.Collapse>
            </Container>
        </Navbar>
    );
}

export default AppNavbar;
```

## src/components/LoadingState.jsx

```jsx
import { Spinner, Alert, Button, Container } from "react-bootstrap";

function LoadingState({ isLoading, error, onRetry }) {
    if (isLoading) {
        return (
            // py-5 и text-center — утилитарные классы Bootstrap:
            // вертикальные отступы и центрирование без своего CSS
            <Container className="py-5 text-center">
                <Spinner animation="border" variant="primary" role="status">
                    {/* visually-hidden оставляет текст для скринридеров,
                        скрывая его визуально */}
                    <span className="visually-hidden">Загрузка...</span>
                </Spinner>
                <p className="mt-3 text-muted">Загружаем задания...</p>
            </Container>
        );
    }

    if (error) {
        return (
            <Container className="py-5">
                <Alert variant="danger">
                    <Alert.Heading>Не удалось загрузить данные</Alert.Heading>
                    <p className="mb-3">{error}</p>
                    <Button variant="outline-danger" onClick={onRetry}>
                        Повторить
                    </Button>
                </Alert>
            </Container>
        );
    }

    return null;
}

export default LoadingState;
```

## src/components/TaskCard.jsx

```jsx
import { Card, Button, Badge, Stack } from "react-bootstrap";
import { useTasks } from "../context/TasksContext.jsx";

function TaskCard({ task, onShowDetails }) {
    const { isSelected, toggleSelect } = useTasks();
    const selected = isSelected(task.id);

    // Обрезаем длинное описание, чтобы карточки не разъезжались
    const shortText =
        task.description.length > 80 ? `${task.description.slice(0, 80)}...` : task.description;

    return (
        // h-100 растягивает карточку на всю высоту колонки,
        // d-flex flex-column вместе с mt-auto у подвала прижимает
        // кнопки к низу — так все карточки в ряду выглядят одинаково
        <Card className="h-100 d-flex flex-column task-card shadow-sm">
            <Card.Img variant="top" src={task.image} alt={task.title} className="task-card__img" />

            <Card.Body className="d-flex flex-column">
                <Stack direction="horizontal" gap={2} className="mb-2">
                    <Badge bg="secondary">{task.category}</Badge>
                    <Badge bg="warning" text="dark">
                        ★ {task.rating?.rate ?? "—"}
                    </Badge>
                </Stack>

                <Card.Title className="fs-6">{task.title}</Card.Title>
                <Card.Text className="text-muted small">{shortText}</Card.Text>

                {/* mt-auto занимает всё свободное место сверху,
                    прижимая блок к нижнему краю карточки */}
                <div className="mt-auto pt-2">
                    <div className="fw-bold mb-2">{Math.round(task.price)} баллов</div>

                    <Stack direction="horizontal" gap={2}>
                        <Button
                            variant="outline-primary"
                            size="sm"
                            onClick={() => onShowDetails(task)}
                        >
                            Подробнее
                        </Button>

                        {/* variant меняется в зависимости от состояния —
                            декларативно, без ручного переключения классов */}
                        <Button
                            variant={selected ? "success" : "outline-secondary"}
                            size="sm"
                            onClick={() => toggleSelect(task.id)}
                        >
                            {selected ? "✓ Выбрано" : "Выбрать"}
                        </Button>
                    </Stack>
                </div>
            </Card.Body>
        </Card>
    );
}

export default TaskCard;
```

## src/components/SortControl.jsx

```jsx
import { Form } from "react-bootstrap";

export const SORT_OPTIONS = [
    { value: "title-asc",   label: "Название: А → Я" },
    { value: "points-asc",  label: "Баллы: по возрастанию" },
    { value: "points-desc", label: "Баллы: по убыванию" },
    { value: "rating-desc", label: "Рейтинг: сначала высокий" },
];

/** Сортировка КОПИИ массива: sort мутирует исходный массив на месте */
export function sortTasks(tasks, sortBy) {
    const copy = [...tasks];

    switch (sortBy) {
        case "title-asc":
            return copy.sort((a, b) => a.title.localeCompare(b.title));
        case "points-asc":
            return copy.sort((a, b) => a.price - b.price);
        case "points-desc":
            return copy.sort((a, b) => b.price - a.price);
        case "rating-desc":
            return copy.sort((a, b) => (b.rating?.rate ?? 0) - (a.rating?.rate ?? 0));
        default:
            return copy;
    }
}

function SortControl({ value, onChange }) {
    return (
        <Form.Group controlId="sort-select" className="mb-0">
            <Form.Label className="visually-hidden">Сортировка</Form.Label>
            {/* Контролируемый компонент: значение приходит из состояния */}
            <Form.Select value={value} onChange={(event) => onChange(event.target.value)}>
                {SORT_OPTIONS.map((option) => (
                    <option key={option.value} value={option.value}>
                        {option.label}
                    </option>
                ))}
            </Form.Select>
        </Form.Group>
    );
}

export default SortControl;
```

## src/components/TaskModal.jsx

```jsx
import { Modal, Button, Badge, ListGroup, Image } from "react-bootstrap";
import { useTasks } from "../context/TasksContext.jsx";

function TaskModal({ task, show, onHide }) {
    const { isSelected, toggleSelect } = useTasks();

    // Пока задание не выбрано, содержимого нет — модалку не рендерим
    if (!task) return null;

    const selected = isSelected(task.id);

    return (
        // Видимость управляется пропом show из состояния React.
        // В обычном Bootstrap то же самое делалось бы разметкой:
        // data-bs-toggle="modal" data-bs-target="#id" — то есть
        // скрипт сам менял бы классы в DOM за спиной у React
        <Modal show={show} onHide={onHide} size="lg" centered>
            <Modal.Header closeButton>
                <Modal.Title className="fs-5">{task.title}</Modal.Title>
            </Modal.Header>

            <Modal.Body>
                <div className="text-center mb-3">
                    <Image src={task.image} alt={task.title} fluid style={{ maxHeight: "220px" }} />
                </div>

                <p>{task.description}</p>

                <ListGroup variant="flush">
                    <ListGroup.Item className="d-flex justify-content-between">
                        <span>Направление</span>
                        <Badge bg="secondary">{task.category}</Badge>
                    </ListGroup.Item>
                    <ListGroup.Item className="d-flex justify-content-between">
                        <span>Баллы</span>
                        <strong>{Math.round(task.price)}</strong>
                    </ListGroup.Item>
                    <ListGroup.Item className="d-flex justify-content-between">
                        <span>Рейтинг</span>
                        <span>
                            ★ {task.rating?.rate ?? "—"} ({task.rating?.count ?? 0} оценок)
                        </span>
                    </ListGroup.Item>
                </ListGroup>
            </Modal.Body>

            <Modal.Footer>
                <Button variant="secondary" onClick={onHide}>
                    Закрыть
                </Button>
                <Button
                    variant={selected ? "success" : "primary"}
                    onClick={() => toggleSelect(task.id)}
                >
                    {selected ? "✓ Убрать из выбранных" : "Выбрать задание"}
                </Button>
            </Modal.Footer>
        </Modal>
    );
}

export default TaskModal;
```

## src/components/PaginationBar.jsx

```jsx
import { Pagination } from "react-bootstrap";

function PaginationBar({ currentPage, totalPages, onPageChange }) {
    // Одна страница — панель не нужна
    if (totalPages <= 1) return null;

    const pages = Array.from({ length: totalPages }, (_, index) => index + 1);

    return (
        <Pagination className="justify-content-center mt-4">
            <Pagination.Prev
                disabled={currentPage === 1}
                onClick={() => onPageChange(currentPage - 1)}
            />

            {pages.map((page) => (
                <Pagination.Item
                    key={page}
                    active={page === currentPage}
                    onClick={() => onPageChange(page)}
                >
                    {page}
                </Pagination.Item>
            ))}

            <Pagination.Next
                disabled={currentPage === totalPages}
                onClick={() => onPageChange(currentPage + 1)}
            />
        </Pagination>
    );
}

export default PaginationBar;
```

## src/pages/CatalogPage.jsx

```jsx
import { useState, useEffect } from "react";
import { Container, Row, Col, Form } from "react-bootstrap";
import { useSearchParams } from "react-router-dom";
import { useTasks } from "../context/TasksContext.jsx";
import TaskCard from "../components/TaskCard.jsx";
import SortControl, { sortTasks } from "../components/SortControl.jsx";
import TaskModal from "../components/TaskModal.jsx";
import LoadingState from "../components/LoadingState.jsx";
import PaginationBar from "../components/PaginationBar.jsx";

const PER_PAGE = 8;

function CatalogPage() {
    const { tasks, isLoading, error, reload } = useTasks();
    const [searchParams] = useSearchParams();

    const [sortBy, setSortBy] = useState("title-asc");
    const [page, setPage] = useState(1);
    const [modalTask, setModalTask] = useState(null); // null = модалка закрыта

    const query = (searchParams.get("q") ?? "").trim().toLowerCase();

    // При смене поиска или сортировки возвращаемся на первую страницу,
    // иначе пользователь окажется на пустой третьей странице из двух
    useEffect(() => {
        setPage(1);
    }, [query, sortBy]);

    if (isLoading || error) {
        return <LoadingState isLoading={isLoading} error={error} onRetry={reload} />;
    }

    // Порядок операций: сначала фильтр, потом сортировка, потом срез страницы
    const filtered = tasks.filter((task) => task.title.toLowerCase().includes(query));
    const sorted = sortTasks(filtered, sortBy);
    const totalPages = Math.ceil(sorted.length / PER_PAGE);
    const visible = sorted.slice((page - 1) * PER_PAGE, page * PER_PAGE);

    return (
        <Container className="py-4">
            <Row className="align-items-center mb-4">
                <Col md={8}>
                    <h1 className="h4 mb-0">
                        Каталог заданий{" "}
                        <span className="text-muted fs-6">({sorted.length})</span>
                    </h1>
                    {query && <p className="text-muted small mb-0">Поиск: «{query}»</p>}
                </Col>
                <Col md={4} className="mt-3 mt-md-0">
                    <SortControl value={sortBy} onChange={setSortBy} />
                </Col>
            </Row>

            {/* Сетка: 1 колонка на телефоне, 2 на планшете,
                3 на среднем экране, 4 на большом. g-4 — отступы между ячейками */}
            <Row className="g-4">
                {visible.map((task) => (
                    <Col key={task.id} xs={12} sm={6} md={4} lg={3}>
                        <TaskCard task={task} onShowDetails={setModalTask} />
                    </Col>
                ))}
            </Row>

            {visible.length === 0 && (
                <p className="text-center text-muted py-5">Ничего не найдено</p>
            )}

            <PaginationBar currentPage={page} totalPages={totalPages} onPageChange={setPage} />

            {/* show вычисляется из состояния: есть задание — окно открыто */}
            <TaskModal
                task={modalTask}
                show={modalTask !== null}
                onHide={() => setModalTask(null)}
            />
        </Container>
    );
}

export default CatalogPage;
```

## src/pages/CategoryPage.jsx

```jsx
import { useState } from "react";
import { Container, Row, Col, Breadcrumb } from "react-bootstrap";
import { useParams, Link } from "react-router-dom";
import { useTasks } from "../context/TasksContext.jsx";
import TaskCard from "../components/TaskCard.jsx";
import SortControl, { sortTasks } from "../components/SortControl.jsx";
import TaskModal from "../components/TaskModal.jsx";
import LoadingState from "../components/LoadingState.jsx";

function CategoryPage() {
    const { categoryName } = useParams();
    const { tasks, isLoading, error, reload } = useTasks();
    const [sortBy, setSortBy] = useState("points-desc");
    const [modalTask, setModalTask] = useState(null);

    if (isLoading || error) {
        return <LoadingState isLoading={isLoading} error={error} onRetry={reload} />;
    }

    // Параметр приходит закодированным (пробелы как %20) — декодируем
    const category = decodeURIComponent(categoryName);
    const categoryTasks = sortTasks(
        tasks.filter((task) => task.category === category),
        sortBy
    );

    return (
        <Container className="py-4">
            <Breadcrumb>
                <Breadcrumb.Item linkAs={Link} linkProps={{ to: "/" }}>
                    Все задания
                </Breadcrumb.Item>
                <Breadcrumb.Item active>{category}</Breadcrumb.Item>
            </Breadcrumb>

            <Row className="align-items-center mb-4">
                <Col md={8}>
                    <h1 className="h4 mb-0 text-capitalize">{category}</h1>
                </Col>
                <Col md={4} className="mt-3 mt-md-0">
                    <SortControl value={sortBy} onChange={setSortBy} />
                </Col>
            </Row>

            <Row className="g-4">
                {categoryTasks.map((task) => (
                    <Col key={task.id} xs={12} sm={6} md={4} lg={3}>
                        <TaskCard task={task} onShowDetails={setModalTask} />
                    </Col>
                ))}
            </Row>

            <TaskModal
                task={modalTask}
                show={modalTask !== null}
                onHide={() => setModalTask(null)}
            />
        </Container>
    );
}

export default CategoryPage;
```

## src/pages/CartPage.jsx

```jsx
import { useState } from "react";
import { Container, Row, Col, Card, Form, Button, Alert, ListGroup, Badge } from "react-bootstrap";
import { Link } from "react-router-dom";
import { useTasks } from "../context/TasksContext.jsx";

const EMPTY_FORM = { teamName: "", email: "", track: "solidity", agree: false };

function CartPage() {
    const { tasks, selected, toggleSelect, clearSelected } = useTasks();

    const [form, setForm] = useState(EMPTY_FORM);
    const [errors, setErrors] = useState({});
    const [submitted, setSubmitted] = useState(false);

    const selectedTasks = tasks.filter((task) => selected.includes(task.id));
    const totalPoints = selectedTasks.reduce((sum, task) => sum + Math.round(task.price), 0);

    const handleChange = (event) => {
        const { name, value, type, checked } = event.target;

        setForm((prev) => ({
            ...prev,
            // У чекбокса значимое поле — checked, а не value
            [name]: type === "checkbox" ? checked : value,
        }));
    };

    const handleSubmit = (event) => {
        event.preventDefault();

        const nextErrors = {};
        if (form.teamName.trim().length < 2) nextErrors.teamName = "Минимум 2 символа";
        if (!form.email.includes("@")) nextErrors.email = "Укажите корректный email";
        if (!form.agree) nextErrors.agree = "Нужно подтвердить участие";
        if (selectedTasks.length === 0) nextErrors.tasks = "Выберите хотя бы одно задание";

        if (Object.keys(nextErrors).length > 0) {
            setErrors(nextErrors);
            setSubmitted(false);
            return;
        }

        setErrors({});
        setSubmitted(true);
    };

    return (
        <Container className="py-4">
            <h1 className="h4 mb-4">Выбранные задания</h1>

            <Row className="g-4">
                <Col lg={7}>
                    {selectedTasks.length === 0 ? (
                        <Alert variant="secondary">
                            Пока ничего не выбрано. <Alert.Link as={Link} to="/">Открыть каталог</Alert.Link>
                        </Alert>
                    ) : (
                        <ListGroup>
                            {selectedTasks.map((task) => (
                                <ListGroup.Item
                                    key={task.id}
                                    className="d-flex justify-content-between align-items-center"
                                >
                                    <div className="me-3">
                                        <div className="small fw-semibold">{task.title}</div>
                                        <Badge bg="secondary">{task.category}</Badge>
                                    </div>
                                    <div className="d-flex align-items-center gap-3">
                                        <strong>{Math.round(task.price)}</strong>
                                        <Button
                                            variant="outline-danger"
                                            size="sm"
                                            onClick={() => toggleSelect(task.id)}
                                        >
                                            Убрать
                                        </Button>
                                    </div>
                                </ListGroup.Item>
                            ))}
                        </ListGroup>
                    )}

                    {selectedTasks.length > 0 && (
                        <Button variant="link" className="px-0 mt-2" onClick={clearSelected}>
                            Очистить всё
                        </Button>
                    )}
                </Col>

                <Col lg={5}>
                    <Card bg="light" className="mb-4">
                        <Card.Body className="d-flex justify-content-between align-items-center">
                            <span>Заданий: {selectedTasks.length}</span>
                            <span className="fs-5 fw-bold">{totalPoints} баллов</span>
                        </Card.Body>
                    </Card>

                    <Card>
                        <Card.Body>
                            <Card.Title className="fs-6 mb-3">Заявка команды</Card.Title>

                            {submitted && (
                                <Alert variant="success" onClose={() => setSubmitted(false)} dismissible>
                                    Заявка от «{form.teamName}» принята: {selectedTasks.length} заданий
                                    на {totalPoints} баллов.
                                </Alert>
                            )}

                            {errors.tasks && <Alert variant="warning">{errors.tasks}</Alert>}

                            <Form onSubmit={handleSubmit} noValidate>
                                <Form.Group className="mb-3" controlId="teamName">
                                    <Form.Label>Название команды</Form.Label>
                                    <Form.Control
                                        name="teamName"
                                        value={form.teamName}
                                        onChange={handleChange}
                                        // isInvalid включает красную рамку и показывает Feedback
                                        isInvalid={Boolean(errors.teamName)}
                                        placeholder="ProfiHub Team"
                                    />
                                    <Form.Control.Feedback type="invalid">
                                        {errors.teamName}
                                    </Form.Control.Feedback>
                                </Form.Group>

                                <Form.Group className="mb-3" controlId="email">
                                    <Form.Label>Email капитана</Form.Label>
                                    <Form.Control
                                        type="email"
                                        name="email"
                                        value={form.email}
                                        onChange={handleChange}
                                        isInvalid={Boolean(errors.email)}
                                        placeholder="captain@example.com"
                                    />
                                    <Form.Control.Feedback type="invalid">
                                        {errors.email}
                                    </Form.Control.Feedback>
                                    <Form.Text className="text-muted">
                                        На эту почту придёт подтверждение участия.
                                    </Form.Text>
                                </Form.Group>

                                <Form.Group className="mb-3" controlId="track">
                                    <Form.Label>Основное направление</Form.Label>
                                    <Form.Select name="track" value={form.track} onChange={handleChange}>
                                        <option value="solidity">Смарт-контракты</option>
                                        <option value="frontend">Интерфейсы</option>
                                        <option value="devops">Инфраструктура</option>
                                    </Form.Select>
                                </Form.Group>

                                <Form.Group className="mb-3" controlId="agree">
                                    <Form.Check
                                        type="checkbox"
                                        name="agree"
                                        checked={form.agree}
                                        onChange={handleChange}
                                        isInvalid={Boolean(errors.agree)}
                                        label="Подтверждаю участие команды"
                                        feedback={errors.agree}
                                        feedbackType="invalid"
                                    />
                                </Form.Group>

                                <Button type="submit" variant="primary" className="w-100">
                                    Отправить заявку
                                </Button>
                            </Form>
                        </Card.Body>
                    </Card>
                </Col>
            </Row>
        </Container>
    );
}

export default CartPage;
```

## src/App.jsx

```jsx
import { Routes, Route } from "react-router-dom";
import AppNavbar from "./components/AppNavbar.jsx";
import CatalogPage from "./pages/CatalogPage.jsx";
import CategoryPage from "./pages/CategoryPage.jsx";
import CartPage from "./pages/CartPage.jsx";

function App() {
    return (
        <>
            <AppNavbar />

            <Routes>
                <Route path="/" element={<CatalogPage />} />
                <Route path="/category/:categoryName" element={<CategoryPage />} />
                <Route path="/cart" element={<CartPage />} />
                <Route path="*" element={<CatalogPage />} />
            </Routes>
        </>
    );
}

export default App;
```

## src/App.css

```css
/* Собственные классы ДОПОЛНЯЮТ компоненты Bootstrap через проп className.
   Переопределять встроенные классы (.card, .btn) напрямую не стоит:
   это ломает все остальные места, где они используются, и ваши правки
   исчезнут при обновлении версии Bootstrap. Отдельный класс на конкретный
   компонент изолирует изменения и не трогает библиотеку. */

.task-card {
    transition: transform 0.15s ease, box-shadow 0.15s ease;
}

.task-card:hover {
    transform: translateY(-4px);
    box-shadow: 0 8px 20px rgba(0, 0, 0, 0.12) !important;
}

/* Изображения товаров в API разного размера и пропорций.
   Фиксированная высота + object-fit: contain выравнивают карточки,
   не искажая картинки */
.task-card__img {
    height: 180px;
    object-fit: contain;
    padding: 16px;
    background-color: #ffffff;
}
```

## Разбор решения

1. **Одинаковая высота карточек.** Самая частая проблема сетки товаров: заголовки разной длины «ломают» ряд. Решение из трёх частей: `h-100` растягивает `Card` на высоту колонки, `d-flex flex-column` делает тело карточки колонкой, `mt-auto` у нижнего блока съедает свободное пространство и прижимает кнопки к низу.
2. **Адаптивная сетка.** `xs={12} sm={6} md={4} lg={3}` — это 1 / 2 / 3 / 4 колонки. Bootstrap работает по принципу mobile-first: значение действует начиная с указанной ширины и выше, пока его не перебьёт следующий брейкпоинт. Отступы задаются классом `g-4` у `Row`, а не `margin` у колонок.
3. **`as={Link}`.** Компоненты `Navbar.Brand` и `Nav.Link` по умолчанию рендерят `<a href>`, а это полная перезагрузка приложения. Проп `as` подменяет внутренний тег на роутерный `Link`, сохраняя при этом все классы Bootstrap. Тот же приём работает с `Breadcrumb.Item` (через `linkAs`) и `Alert.Link`.
4. **Модалка на состоянии.** `show={modalTask !== null}` — окно открыто ровно тогда, когда есть что показывать. Одна переменная состояния заменяет и флаг видимости, и хранилище данных. В классическом Bootstrap видимость управлялась бы атрибутами `data-bs-toggle`, то есть сторонним скриптом, меняющим DOM за спиной React.
5. **Контролируемая форма одним объектом.** Одно состояние `form` и один `handleChange` на все поля. Чекбокс обрабатывается отдельной веткой: у него значимое свойство — `checked`, а не `value`.
6. **Валидация средствами Bootstrap.** Проп `isInvalid` включает красную рамку и показывает вложенный `Form.Control.Feedback type="invalid"`. Атрибут `noValidate` у `<Form>` отключает браузерную валидацию, чтобы не конфликтовать с собственной.
7. **Порядок фильтр → сортировка → срез.** Сначала отбираем подходящие элементы, потом сортируем результат, и только потом режем на страницы. Обратный порядок дал бы сортировку внутри одной страницы вместо сортировки всего списка.
8. **Сброс страницы при смене фильтров.** Без `useEffect`, возвращающего `page` на первую, пользователь после поиска окажется на третьей странице списка из одной — визуально пустой экран без объяснения.
9. **`fetch` и `response.ok`.** `fetch` отклоняет промис только при сетевом сбое; ответ со статусом 404 или 500 считается успешным. Проверка `response.ok` и ручной `throw` переводят HTTP-ошибку в ветку `catch`.
10. **`useEffect` не может быть `async`.** Эффект обязан вернуть либо функцию очистки, либо ничего, а `async`-функция всегда возвращает промис. Поэтому асинхронная логика объявляется внутри эффекта отдельной функцией и сразу вызывается.
11. **Категории выводятся, а не хранятся.** `[...new Set(tasks.map(t => t.category))]` вычисляется из загруженных данных через `useMemo`. Отдельное состояние для категорий пришлось бы синхронизировать вручную — лишний источник рассогласования.
12. **Свои классы дополняют, а не переопределяют.** `.task-card` навешивается через `className` поверх стандартного `.card`. Правка самого `.card` затронула бы все карточки приложения и потерялась бы при обновлении Bootstrap; отдельный класс изолирует изменения. Единственный `!important` в решении нужен, чтобы перебить `shadow-sm` при наведении — утилитарные классы Bootstrap имеют высокий приоритет.

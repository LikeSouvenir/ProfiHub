# Модуль 9: React-Bootstrap — Практическое задание

## Проект: Витрина заданий с загрузкой из API (TaskStore)

### Цель

Освоить подключение React-Bootstrap к проекту, сетку (`Container`, `Row`, `Col`), готовые компоненты (`Navbar`, `Nav`, `Card`, `Button`, `Badge`, `Form`, `Spinner`, `Alert`, `Modal`, `Pagination`, `ListGroup`), составные компоненты через dot notation (`Card.Body`, `Form.Group`, `Modal.Header`), контролируемые формы на `Form.Control` и `Form.Select`, управление состоянием компонентов Bootstrap из React, а также комбинирование утилитарных классов Bootstrap с собственными стилями.

Проект объединяет всё изученное: роутинг, контекст, хуки и загрузку данных из внешнего API.

### Техническое задание

1. **Установка и подключение:**

   ```bash
   npm install react-bootstrap bootstrap react-router-dom
   ```

   - CSS Bootstrap импортируется **один раз** в `main.jsx`: `import "bootstrap/dist/css/bootstrap.min.css";`

   - В `README.md` поясните, почему в React используется `react-bootstrap`, а не подключение `bootstrap.bundle.js` напрямую (конфликт императивного jQuery-подхода с декларативным React; управление состоянием модалок и дропдаунов).

   - Покажите оба способа импорта компонентов и объясните, какой предпочтительнее для размера бандла:

     ```jsx
     import { Button, Card } from "react-bootstrap";   // общий импорт
     import Button from "react-bootstrap/Button";       // точечный импорт
     ```

2. **Структура:**

   ```
   src/
   ├── components/
   │   ├── AppNavbar.jsx
   │   ├── TaskCard.jsx
   │   ├── SortControl.jsx
   │   ├── TaskModal.jsx
   │   ├── LoadingState.jsx
   │   └── PaginationBar.jsx
   ├── context/
   │   └── TasksContext.jsx
   ├── pages/
   │   ├── CatalogPage.jsx
   │   ├── CategoryPage.jsx
   │   └── CartPage.jsx
   ├── App.jsx
   └── main.jsx
   ```

3. **Загрузка данных (`TasksContext.jsx`):**

   - Данные берутся из `https://fakestoreapi.com/products` — товары трактуются как «задания чемпионата» (поля `id`, `title`, `price` → баллы, `category` → направление, `image`, `rating`, `description`).

   - Загрузка в `useEffect` при монтировании через `async/await` + `try/catch/finally`.

   - Состояния: `tasks`, `isLoading`, `error`, а также производный список категорий (`[...new Set(tasks.map(t => t.category))]`).

   - Контекст также хранит «корзину заданий» (выбранные для выполнения): `selected`, `toggleSelect`, `isSelected`, `clearSelected`.

4. **`AppNavbar.jsx`:**

   - Компонент `Navbar` с `bg="dark"`, `variant="dark"`, `expand="lg"` и рабочей кнопкой-бургером (`Navbar.Toggle` + `Navbar.Collapse`).

   - `Navbar.Brand` оборачивает роутерную ссылку через `as={Link}` — поясните комментарием, зачем нужен проп `as`.

   - Пункты меню — категории, полученные из контекста; активный пункт подсвечивается.

   - Справа — `Badge` с количеством выбранных заданий и `Form` с полем поиска (`Form.Control` + `Button`), собранная через `d-flex`.

5. **`LoadingState.jsx`:**

   - Во время загрузки — `Spinner` с `animation="border"` внутри отцентрованного блока.

   - При ошибке — `Alert` с `variant="danger"`, текстом ошибки и кнопкой «Повторить».

6. **Сетка и карточки (`CatalogPage.jsx`, `TaskCard.jsx`):**

   - `Container` → `Row` → `Col` с адаптивными брейкпоинтами: `xs={12} sm={6} md={4} lg={3}`.

   - У `Row` — `g-4` для отступов между колонками.

   - `TaskCard` использует `Card`, `Card.Img`, `Card.Body`, `Card.Title`, `Card.Text`, `Card.Footer`.

   - Все карточки одинаковой высоты: `className="h-100 d-flex flex-column"` + `mt-auto` у подвала.

   - `Badge` с категорией и `Badge bg="warning"` с рейтингом.

   - Длинное описание обрезается до 80 символов с многоточием.

   - Две кнопки: «Подробнее» (открывает модальное окно) и «Выбрать / Убрать» (`variant` меняется в зависимости от состояния).

7. **`SortControl.jsx`:**

   - `Form.Select` с вариантами: по названию (А→Я), по баллам (возр.), по баллам (убыв.), по рейтингу.

   - Контролируемый компонент: `value` из состояния, изменение через `onChange`.

   - Сортировка выполняется **на копии массива**, исходные данные не мутируются.

8. **`TaskModal.jsx`:**

   - `Modal` с `show`, `onHide`, `size="lg"`, `centered`.

   - Внутри: `Modal.Header closeButton`, `Modal.Title`, `Modal.Body` (изображение, полное описание, `ListGroup` с характеристиками), `Modal.Footer` с кнопками.

   - Открытие и закрытие управляются состоянием React — поясните комментарием, что в обычном Bootstrap это делал бы `data-bs-toggle`.

9. **`PaginationBar.jsx`:**

   - Компонент `Pagination` с `Pagination.Prev`, `Pagination.Item`, `Pagination.Next`.

   - По 8 заданий на страницу; активный элемент подсвечивается через `active={...}`.

   - При смене фильтра, поиска или сортировки текущая страница сбрасывается на первую.

10. **Форма выбора (`CartPage.jsx`):**

    - `Form` с `Form.Group`, `Form.Label`, `Form.Control`, `Form.Select`, `Form.Check` (чекбокс согласия), `Form.Text` с подсказкой.

    - Валидация через `Form.Control.Feedback` и проп `isInvalid`.

    - Обработчик `onSubmit` с `preventDefault`, вывод результата в `Alert variant="success"`.

    - Суммарное количество баллов выбранных заданий выводится в `Card` с `bg="light"`.

11. **Собственные стили поверх Bootstrap:**

    - Минимум два кастомных класса в `App.css`, дополняющих стандартные компоненты (например, `hover`-эффект карточки и фиксированная высота изображения через `object-fit: contain`).

    - Поясните, почему свои классы дописываются к Bootstrap-компонентам через проп `className`, а не переопределяют его встроенные классы.

### Критерии приёмки

- CSS Bootstrap импортирован ровно один раз, до собственных стилей.
- Модальное окно и бургер-меню работают без подключения JS-бандла Bootstrap.
- Карточки в ряду одинаковой высоты независимо от длины заголовка.
- Поиск, фильтр по категории, сортировка и пагинация работают совместно и не конфликтуют.
- На ширине телефона сетка перестраивается в одну колонку.
- Исходный массив заданий нигде не мутируется.

# Модуль 1: HTML и CSS — Эталонное решение

## index.html

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ProfiHub Team — команда чемпионата</title>
    <!-- Стили вынесены в отдельный файл: разметка отвечает за структуру, CSS — за оформление -->
    <link rel="stylesheet" href="style.css">
</head>
<body>

    <!-- ШАПКА: логотип слева, навигация справа -->
    <header class="site-header">
        <div class="logo">ProfiHub<span>Team</span></div>
        <nav class="main-nav">
            <!-- Якорные ссылки ведут на id секций ниже по странице -->
            <a href="#about">О команде</a>
            <a href="#members">Состав</a>
            <a href="#schedule">Расписание</a>
            <a href="#contact">Контакты</a>
        </nav>
    </header>

    <main class="container">

        <!-- СЕКЦИЯ 1: описание команды + список стека -->
        <section id="about" class="section">
            <h2>О команде</h2>
            <p>
                Мы готовимся к чемпионату по блокчейн-разработке в составе трёх человек.
                Каждый участник отвечает за своё направление: смарт-контракты, интерфейс
                и инфраструктуру. Материалы, конспекты и практические задания мы собираем
                в общей базе знаний ProfiHub.
            </p>
            <h3>Наш стек</h3>
            <ul class="stack-list">
                <li>Solidity &amp; Hardhat</li>
                <li>JavaScript (ES6+)</li>
                <li>React + React Router</li>
                <li>Docker &amp; Geth</li>
                <li>Git &amp; GitHub</li>
            </ul>
        </section>

        <!-- СЕКЦИЯ 2: карточки участников в ряд через flex -->
        <section id="members" class="section">
            <h2>Состав</h2>
            <div class="members-grid">
                <article class="member-card">
                    <img src="img/member1.jpg" alt="Фото Алины" class="avatar">
                    <h3>Алина Смирнова</h3>
                    <p class="role">Solidity-разработчик</p>
                </article>

                <article class="member-card">
                    <img src="img/member2.jpg" alt="Фото Ивана" class="avatar">
                    <h3>Иван Петров</h3>
                    <p class="role">Frontend-разработчик</p>
                </article>

                <article class="member-card">
                    <img src="img/member3.jpg" alt="Фото Марии" class="avatar">
                    <h3>Мария Ким</h3>
                    <p class="role">DevOps-инженер</p>
                </article>
            </div>
        </section>

        <!-- СЕКЦИЯ 3: таблица расписания -->
        <section id="schedule" class="section">
            <h2>Расписание подготовки</h2>
            <table class="schedule-table">
                <thead>
                    <tr>
                        <th>День</th>
                        <th>Модуль</th>
                        <th>Время</th>
                    </tr>
                </thead>
                <tbody>
                    <tr>
                        <td>Понедельник</td>
                        <td>Solidity: типы данных и функции</td>
                        <td>10:00 — 13:00</td>
                    </tr>
                    <tr>
                        <td>Вторник</td>
                        <td>HTML / CSS: вёрстка интерфейса</td>
                        <td>10:00 — 13:00</td>
                    </tr>
                    <tr>
                        <td>Среда</td>
                        <td>JavaScript: DOM и события</td>
                        <td>14:00 — 17:00</td>
                    </tr>
                    <tr>
                        <td>Четверг</td>
                        <td>Web3.js: подключение к контракту</td>
                        <td>14:00 — 17:00</td>
                    </tr>
                    <tr>
                        <td>Пятница</td>
                        <td>Разбор кейса прошлого финала</td>
                        <td>10:00 — 16:00</td>
                    </tr>
                </tbody>
            </table>
        </section>

        <!-- СЕКЦИЯ 4: форма обратной связи -->
        <section id="contact" class="section">
            <h2>Связаться с нами</h2>
            <form class="contact-form">
                <!-- label связан с полем через for="id-поля": клик по подписи ставит фокус в поле -->
                <label for="name">Имя</label>
                <input type="text" id="name" name="name" placeholder="Как к вам обращаться">

                <label for="email">Email</label>
                <!-- required включает браузерную валидацию: пустое поле не даст отправить форму -->
                <input type="email" id="email" name="email" placeholder="you@example.com" required>

                <label for="direction">Направление</label>
                <select id="direction" name="direction">
                    <option value="solidity">Solidity</option>
                    <option value="frontend">Frontend</option>
                    <option value="devops">DevOps</option>
                </select>

                <label for="message">Сообщение</label>
                <textarea id="message" name="message" rows="4" placeholder="Коротко опишите вопрос"></textarea>

                <button type="submit" class="submit-btn">Отправить</button>
            </form>
        </section>

    </main>

    <footer class="site-footer">
        <p>&copy; 2026 ProfiHub Team. Учебный проект.</p>
        <p><a href="mailto:team@profihub.dev">team@profihub.dev</a></p>
    </footer>

</body>
</html>
```

## style.css

```css
/* ===== БАЗОВЫЕ СТИЛИ ===== */

/* Сбрасываем отступы по умолчанию и задаём общий шрифт для всей страницы */
body {
    margin: 0;
    font-family: "Segoe UI", Arial, sans-serif;
    color: #23272f;
    background-color: #f7f8fa;
    line-height: 1.6;
}

/* Групповой селектор: одинаковый отступ снизу для всех заголовков */
h1, h2, h3 {
    margin: 0 0 12px;
}

/* Ограничиваем ширину контента и центрируем его по горизонтали */
.container {
    max-width: 900px;
    margin: 0 auto;
    padding: 0 20px;
}

/* ===== ШАПКА ===== */

.site-header {
    display: flex;                   /* логотип и меню в одну строку */
    justify-content: space-between;  /* разводим их по краям */
    align-items: center;             /* выравниваем по вертикали */
    padding: 16px 30px;
    background-color: #1b2430;
    color: #ffffff;
}

.logo {
    font-size: 20px;
    font-weight: bold;
    letter-spacing: 0.5px;
}

/* Вложенный span внутри логотипа красим в акцентный цвет */
.logo span {
    color: #4ea1ff;
}

.main-nav a {
    color: #d6dde6;
    text-decoration: none;
    margin-left: 22px;
    font-size: 15px;
}

/* Псевдокласс :hover — стиль применяется только при наведении курсора */
.main-nav a:hover {
    color: #4ea1ff;
    text-decoration: underline;
}

/* ===== СЕКЦИИ ===== */

.section {
    padding: 36px 0;
    border-bottom: 1px solid #e3e6ea;
}

/* Селектор по id: выделяем первую секцию чуть большим отступом сверху */
#about {
    padding-top: 48px;
}

.stack-list {
    list-style: square;
    padding-left: 22px;
}

/* ===== КАРТОЧКИ УЧАСТНИКОВ ===== */

.members-grid {
    display: flex;
    gap: 20px;          /* одинаковое расстояние между карточками */
    flex-wrap: wrap;    /* на узком экране карточки перенесутся на новую строку */
}

.member-card {
    flex: 1 1 220px;    /* карточки растягиваются поровну, минимальная ширина 220px */
    background-color: #ffffff;
    border: 1px solid #e3e6ea;
    border-radius: 10px;
    padding: 20px;
    text-align: center;
}

.avatar {
    width: 96px;
    height: 96px;
    border-radius: 50%;   /* квадратное изображение превращается в круг */
    object-fit: cover;    /* картинка заполняет квадрат без искажения пропорций */
    margin-bottom: 12px;
}

.role {
    color: #6b7482;
    font-size: 14px;
    margin: 0;
}

/* ===== ТАБЛИЦА ===== */

.schedule-table {
    width: 100%;
    border-collapse: collapse;  /* схлопываем двойные границы между ячейками */
}

/* Групповой селектор: общие правила для ячеек шапки и тела таблицы */
.schedule-table th,
.schedule-table td {
    border: 1px solid #d5dae0;
    padding: 10px 12px;
    text-align: left;
}

.schedule-table thead th {
    background-color: #1b2430;
    color: #ffffff;
}

/* Чередование фона строк улучшает читаемость таблицы */
.schedule-table tbody tr:nth-child(even) {
    background-color: #f0f2f5;
}

/* ===== ФОРМА ===== */

.contact-form {
    display: flex;
    flex-direction: column;  /* поля идут друг под другом */
    max-width: 480px;
}

.contact-form label {
    margin-top: 14px;
    margin-bottom: 4px;
    font-size: 14px;
    font-weight: bold;
}

/* Групповой селектор для всех типов полей ввода */
.contact-form input,
.contact-form select,
.contact-form textarea {
    padding: 9px 11px;
    border: 1px solid #c8ced6;
    border-radius: 6px;
    font-family: inherit;  /* поля наследуют шрифт страницы, а не системный */
    font-size: 14px;
}

.submit-btn {
    margin-top: 20px;
    padding: 11px 20px;
    background-color: #4ea1ff;
    color: #ffffff;
    border: none;
    border-radius: 6px;
    font-size: 15px;
    cursor: pointer;
    align-self: flex-start;
}

.submit-btn:hover {
    background-color: #2f88ec;
}

/* ===== ПОДВАЛ ===== */

.site-footer {
    text-align: center;
    padding: 20px;
    background-color: #eceff1;
    color: #6b7482;
    font-size: 13px;
}

.site-footer a {
    color: #2f88ec;
    text-decoration: none;
}

.site-footer a:hover {
    text-decoration: underline;
}
```

## Разбор решения

1. **Семантика вместо `div`.** Структурные блоки размечены тегами `header`, `nav`, `main`, `section`, `article`, `footer`. `div` остался только там, где нужен чисто технический контейнер (`.members-grid`, `.logo`) — это ровно тот случай, для которого `div` и задуман.
2. **Якорная навигация.** Атрибут `id` у секции превращает её в цель ссылки: `href="#about"` прокручивает страницу к `<section id="about">`. Никакого JavaScript для этого не нужно.
3. **Flexbox в двух ролях.** В шапке `justify-content: space-between` разводит логотип и меню по краям; в `.members-grid` `flex: 1 1 220px` вместе с `flex-wrap` даёт карточки равной ширины, которые сами переносятся на узком экране.
4. **Box model.** `padding` задаёт внутреннее «дыхание» карточек, шапки и ячеек таблицы; `margin: 0 auto` у `.container` центрирует контент; `border` + `border-radius` формируют визуальную рамку карточки.
5. **`border-radius: 50%` + `object-fit: cover`.** Первое делает квадратное изображение кругом, второе обрезает картинку по центру вместо растягивания — стандартный приём для аватаров.
6. **Три вида селекторов.** По тегу (`body`, `h1, h2, h3`), по классу (`.member-card`), по id (`#about`), плюс псевдоклассы `:hover` и `:nth-child(even)`.
7. **Доступность формы.** Каждому полю соответствует `<label for="id">`: клик по подписи переводит фокус в поле, а скринридер корректно озвучивает форму. Атрибут `required` у email включает встроенную валидацию браузера без единой строки JS.
8. **`font-family: inherit` у полей.** По умолчанию `input` и `textarea` используют системный шрифт и визуально выбиваются из страницы — наследование возвращает их к общему оформлению.

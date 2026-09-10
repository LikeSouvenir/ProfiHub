# Разбор практического задания: Страница-визитка

Соберём небольшую одностраничную визитку, используя базовые теги HTML и CSS-стили из прошлых конспектов: семантическую структуру, заголовки, изображение, список навыков, ссылки и простую стилизацию через внешний файл CSS.

## Условие задачи
Создать страницу личной визитки из двух файлов — `index.html` и `style.css` — с блоками:
- Шапка с именем и фото.
- Раздел "Обо мне".
- Список навыков.
- Ссылки на соцсети/почту.
- Подвал с копирайтом.

## Решение с подробными комментариями

**index.html**
```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Визитка — Алина Смирнова</title>
    <!-- Подключаем внешний файл стилей -->
    <link rel="stylesheet" href="style.css">
</head>
<body>

    <!-- Семантический тег header — шапка страницы -->
    <header class="header">
        <img src="avatar.jpg" alt="Фото Алины" class="avatar">
        <h1>Алина Смирнова</h1>
        <p class="subtitle">Frontend-разработчик</p>
    </header>

    <!-- Основной контент страницы -->
    <main>
        <section class="about">
            <h2>Обо мне</h2>
            <p>
                Занимаюсь разработкой веб-интерфейсов уже 2 года.
                Люблю чистый код и аккуратный дизайн.
            </p>
        </section>

        <section class="skills">
            <h2>Навыки</h2>
            <ul>
                <li>HTML5 &amp; CSS3</li>
                <li>JavaScript</li>
                <li>Адаптивная вёрстка</li>
                <li>Git &amp; GitHub</li>
            </ul>
        </section>

        <section class="contacts">
            <h2>Связаться со мной</h2>
            <a href="mailto:alina@example.com">alina@example.com</a><br>
            <a href="https://github.com/example" target="_blank">GitHub</a>
        </section>
    </main>

    <!-- Семантический тег footer — подвал страницы -->
    <footer class="footer">
        <p>&copy; 2026 Алина Смирнова. Все права защищены.</p>
    </footer>

</body>
</html>
```

**style.css**
```css
/* Сбрасываем стандартные отступы у body, чтобы контент начинался вплотную к краям */
body {
    margin: 0;
    font-family: Arial, sans-serif;
    color: #333;
    background-color: #fafafa;
}

/* Шапка: центрируем содержимое и добавляем фон */
.header {
    text-align: center;
    padding: 40px 20px;
    background-color: #2c3e50;
    color: white;
}

.avatar {
    width: 120px;
    height: 120px;
    border-radius: 50%; /* делаем фото круглым */
    object-fit: cover;
    margin-bottom: 15px;
}

.subtitle {
    color: #cfd8dc;
    font-size: 16px;
}

/* Основной контент: ограничиваем ширину и центрируем по странице */
main {
    max-width: 600px;
    margin: 30px auto;
    padding: 0 20px;
}

.about, .skills, .contacts {
    margin-bottom: 30px;
}

h2 {
    border-bottom: 2px solid #2c3e50;
    padding-bottom: 6px;
}

.skills ul {
    list-style: square;
    padding-left: 20px;
}

.contacts a {
    color: #2c3e50;
    text-decoration: none;
    font-weight: bold;
}

.contacts a:hover {
    text-decoration: underline;
}

/* Подвал */
.footer {
    text-align: center;
    padding: 15px;
    background-color: #eceff1;
    font-size: 13px;
    color: #777;
}
```

## Что здесь используется
1. **Семантические теги** `<header>`, `<main>`, `<section>`, `<footer>` — структура страницы понятна сразу по разметке, без единого "безликого" `<div>`.
2. **Классы (`class`)** — каждому блоку присвоен свой класс, чтобы применить к нему собственные стили в `style.css`.
3. **Box model** — используются `padding` (внутренние отступы шапки и подвала) и `margin` (расстояние между блоками, центрирование `main` через `margin: 30px auto`).
4. **Псевдокласс `:hover`** — ссылки в разделе контактов подчёркиваются только при наведении.
5. **`border-radius: 50%`** на квадратной картинке `120×120` превращает её в круг — частый приём для аватаров.

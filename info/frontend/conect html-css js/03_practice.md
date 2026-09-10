# Практика: делаем страницу интерактивной

Возьмём страницу биографии из предыдущей темы и добавим ей интерактивность через `document`, обработчики событий и работу с классами/стилями.

## Условие
На странице `index.html`:
1. Кнопка переключения темы (светлая/тёмная) — по клику меняет класс у `<body>`.
2. Кнопка "Показать больше", которая раскрывает скрытый по умолчанию блок с дополнительной информацией о себе.
3. Счётчик, показывающий, сколько раз пользователь нажал на аватар (для тренировки `addEventListener` и работы с `textContent`).

## HTML
```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>Обо мне</title>
    <link rel="stylesheet" href="style.css">
</head>
<body id="pageBody">

    <header class="site-header">
        <div class="logo">MySite</div>
        <button id="themeBtn">Тёмная тема</button>
    </header>

    <main class="content">
        <img src="me.jpg" alt="Моё фото" class="profile-photo" id="avatar">
        <p>Кликов по фото: <span id="clickCount">0</span></p>

        <h1>Привет, я Алина</h1>
        <p>Изучаю веб-разработку: HTML, CSS и JavaScript.</p>

        <button id="moreBtn">Показать больше</button>

        <!-- Скрыт по умолчанию через CSS-класс .hidden -->
        <p id="moreInfo" class="hidden">
            Дополнительно: люблю разбирать открытые проекты на GitHub
            и участвовать в хакатонах по блокчейну.
        </p>
    </main>

    <footer class="site-footer">
        <p>&copy; 2026 Алина</p>
    </footer>

    <script src="script.js" defer></script>
</body>
</html>
```

## CSS
```css
.hidden {
    display: none;
}

/* Класс, который будет переключаться на body для тёмной темы */
.dark-theme {
    background-color: #1a1a1a;
    color: #f0f0f0;
}

.dark-theme .site-header {
    background-color: #000;
}
```

## JavaScript
```js
// Ждём полной загрузки документа, чтобы все элементы точно существовали
document.addEventListener("DOMContentLoaded", function () {

    // ==== 1. Переключение темы ====
    const themeBtn = document.getElementById("themeBtn");
    const body = document.getElementById("pageBody");

    themeBtn.addEventListener("click", function () {
        // toggle: добавит класс, если его нет, уберёт, если есть
        body.classList.toggle("dark-theme");

        // Меняем текст на кнопке в зависимости от текущей темы
        if (body.classList.contains("dark-theme")) {
            themeBtn.textContent = "Светлая тема";
        } else {
            themeBtn.textContent = "Тёмная тема";
        }
    });

    // ==== 2. Показать/скрыть дополнительную информацию ====
    const moreBtn = document.getElementById("moreBtn");
    const moreInfo = document.getElementById("moreInfo");

    moreBtn.addEventListener("click", function () {
        moreInfo.classList.toggle("hidden");

        moreBtn.textContent = moreInfo.classList.contains("hidden")
            ? "Показать больше"
            : "Скрыть";
    });

    // ==== 3. Счётчик кликов по аватару ====
    const avatar = document.getElementById("avatar");
    const clickCountEl = document.getElementById("clickCount");
    let clickCount = 0;

    avatar.addEventListener("click", function () {
        clickCount++;
        clickCountEl.textContent = clickCount;
    });

});
```

## Разбор решения
1. **Переключение темы:** `classList.toggle("dark-theme")` на `<body>` включает/выключает CSS-класс, вся визуальная логика (цвета, фон) остаётся в CSS — JS только переключает "режим". Текст на кнопке синхронизируется с текущим состоянием через `classList.contains(...)`.
2. **Показать больше:** используется тот же приём `toggle`, но уже на конкретном параграфе — класс `.hidden` управляет свойством `display: none`.
3. **Счётчик кликов:** переменная `clickCount` хранится в замыкании внутри обработчика `DOMContentLoaded` (см. конспект про области видимости и замыкания), а `clickCountEl.textContent = clickCount` каждый раз обновляет число, отображаемое пользователю, находя нужный элемент через `getElementById`.

Всё взаимодействие построено по одному и тому же принципу: **найти элемент → повесить `addEventListener` → внутри обработчика изменить DOM** (текст, атрибут, класс или стиль).

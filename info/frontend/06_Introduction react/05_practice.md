# Практика: собственный проект — форма и карточка в App.jsx

## Условие
Создать собственный React-проект. В файле `App.jsx` сверстать:
- `<input>` с `placeholder="username"`;
- кнопку `submit`;
- сделать кнопку и input закруглёнными;
- создать карточку с заголовком и текстом на свою тему.

## Шаг 1. Создание проекта
```bash
npm create vite@latest my-app -- --template react
cd my-app
npm install
npm run dev
```

## Шаг 2. App.jsx
```jsx
import './App.css'

function App() {
    return (
        <div className="app">

            {/* Форма с полем ввода и кнопкой отправки */}
            <form className="form">
                <input
                    type="text"
                    placeholder="username"
                    className="input-field"
                />
                <button type="submit" className="submit-btn">
                    Submit
                </button>
            </form>

            {/* Карточка с заголовком и текстом на свободную тему */}
            <div className="card">
                <h2 className="card-title">Изучаю React</h2>
                <p className="card-text">
                    React — это библиотека для построения пользовательских интерфейсов
                    из переиспользуемых компонентов. Она позволяет описывать, как должен
                    выглядеть интерфейс в зависимости от текущего состояния приложения,
                    а всю остальную работу по обновлению страницы React берёт на себя.
                </p>
            </div>

        </div>
    );
}

export default App;
```

## Шаг 3. App.css
```css
.app {
    max-width: 500px;
    margin: 60px auto;
    padding: 0 20px;
    font-family: Arial, sans-serif;
    display: flex;
    flex-direction: column;
    gap: 30px;
}

/* Форма: input и кнопка в одну строку */
.form {
    display: flex;
    gap: 10px;
}

/* Закруглённый input */
.input-field {
    flex: 1;
    padding: 12px 18px;
    font-size: 15px;
    border: 1px solid #ccc;
    border-radius: 25px; /* делает поле ввода овальным/закруглённым */
    outline: none;
}

.input-field:focus {
    border-color: #4a90d9;
}

/* Закруглённая кнопка submit */
.submit-btn {
    padding: 12px 24px;
    font-size: 15px;
    color: white;
    background-color: #4a90d9;
    border: none;
    border-radius: 25px; /* та же величина скругления, что и у input, для единого стиля */
    cursor: pointer;
}

.submit-btn:hover {
    background-color: #3a7bc0;
}

/* Карточка с заголовком и текстом */
.card {
    padding: 20px;
    background-color: #f8f9fa;
    border-radius: 12px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
}

.card-title {
    margin: 0 0 10px 0;
    font-size: 20px;
    color: #222;
}

.card-text {
    margin: 0;
    font-size: 15px;
    line-height: 1.6;
    color: #555;
}
```

## Разбор решения
1. **`<input placeholder="username">`** — атрибут `placeholder` в JSX пишется точно так же, как в обычном HTML, поскольку является стандартным HTML-атрибутом (в отличие от `class`/`for`, которые в JSX превращаются в `className`/`htmlFor`).
2. **Закругление input и кнопки** достигается одним и тем же CSS-свойством `border-radius: 25px` — чем больше значение относительно высоты элемента, тем более "овальной" становится форма. Одинаковое значение для обоих элементов создаёт визуально согласованный, единый стиль формы.
3. **Карточка** — обычный `<div className="card">` с вложенными `<h2>` (заголовок) и `<p>` (текст), стилизованный через `border-radius`, `box-shadow` и внутренние отступы `padding` — те же принципы box model, что и в CSS-теме, только применённые внутри JSX-компонента.
4. Вся верстка и стили лежат в двух файлах: `App.jsx` отвечает за структуру и содержимое, `App.css` — за внешний вид, что напрямую продолжает принцип разделения HTML/CSS из более ранних тем, только теперь внутри React-компонента.

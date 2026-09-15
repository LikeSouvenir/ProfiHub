# Модуль 7: Базовый React — состояние и эффекты — Эталонное решение

## package.json

```json
{
    "name": "prep-tracker",
    "private": true,
    "version": "1.0.0",
    "type": "module",
    "scripts": {
        "dev": "vite",
        "build": "vite build",
        "preview": "vite preview"
    },
    "dependencies": {
        "react": "^18.3.1",
        "react-dom": "^18.3.1"
    },
    "devDependencies": {
        "@vitejs/plugin-react": "^4.3.1",
        "vite": "^5.4.0"
    }
}
```

**Разбор полей:**

- `type: "module"` — файлы `.js` трактуются как ES-модули, работает `import` вместо `require`.
- `scripts` — ярлыки для команд: `npm run dev` вместо `./node_modules/.bin/vite`.
- `dependencies` — попадают в итоговый бандл и нужны в рантайме (`react`). `devDependencies` — нужны только при разработке и сборке (`vite`, плагины, линтеры) и в продакшен-бандл не входят.
- `^18.3.1` — разрешает обновления до `18.x.x`, но не до `19`. `~18.3.1` — только патчи `18.3.x`. Точная версия без символа фиксирует ровно её.
- `package-lock.json` фиксирует конкретные версии всего дерева зависимостей, включая транзитивные. Благодаря ему установка на другой машине даёт идентичный результат.
- `npm install` — ставит зависимости, при необходимости обновляет lock-файл. `npm ci` — ставит строго по lock-файлу, предварительно удалив `node_modules`; используется в CI ради воспроизводимости. `npm run build` — запускает сборку продакшен-бандла в папку `dist`.

## src/utils/taskUtils.js

```js
// Именованные экспорты: импортируются как import { calcProgress } from "..."

export const calcProgress = (tasks) => {
    if (tasks.length === 0) return 0;
    const done = tasks.filter((task) => task.isDone).length;
    return Math.round((done / tasks.length) * 100);
};

export const filterTasks = (tasks, filter, search) => {
    const query = search.trim().toLowerCase();

    return tasks.filter((task) => {
        const matchesFilter =
            filter === "all" ||
            (filter === "done" && task.isDone) ||
            (filter === "active" && !task.isDone);

        const matchesSearch = task.title.toLowerCase().includes(query);

        return matchesFilter && matchesSearch;
    });
};

// sort мутирует массив, поэтому сортируем копию
export const sortTasks = (tasks, order = "desc") =>
    [...tasks].sort((a, b) => (order === "desc" ? b.points - a.points : a.points - b.points));

export const formatTime = (totalSeconds) => {
    const minutes = String(Math.floor(totalSeconds / 60)).padStart(2, "0");
    const seconds = String(totalSeconds % 60).padStart(2, "0");
    return `${minutes}:${seconds}`;
};
```

## src/api/fakeApi.js

```js
const INITIAL_TASKS = [
    { id: 1, title: "Реестр студентов",      track: "Solidity", points: 10, isDone: true },
    { id: 2, title: "Ролевой доступ",        track: "Solidity", points: 25, isDone: false },
    { id: 3, title: "Лендинг команды",       track: "Frontend", points: 10, isDone: true },
    { id: 4, title: "Трекер заданий",        track: "Frontend", points: 20, isDone: false },
    { id: 5, title: "Сборка ноды в Docker",  track: "DevOps",   points: 35, isDone: false },
];

/** Имитация сетевого запроса: задержка + вероятность сбоя */
export const fetchTasks = () =>
    new Promise((resolve, reject) => {
        setTimeout(() => {
            if (Math.random() < 0.15) {
                reject(new Error("Сервер недоступен"));
                return;
            }
            resolve(INITIAL_TASKS.map((task) => ({ ...task })));
        }, 800);
    });
```

## src/hooks/useLocalTasks.js

```js
import { useState, useEffect, useCallback } from "react";
import { fetchTasks } from "../api/fakeApi.js";

/**
 * Собственный хук: вся работа с массивом заданий вынесена сюда.
 * App остаётся композицией компонентов без логики данных.
 */
export function useLocalTasks() {
    const [tasks, setTasks] = useState([]);
    const [isLoading, setIsLoading] = useState(true);
    const [error, setError] = useState(null);
    // Счётчик перезагрузок: его изменение перезапускает эффект
    const [reloadKey, setReloadKey] = useState(0);

    useEffect(() => {
        // Флаг отмены: если компонент размонтируется раньше ответа,
        // мы не станем вызывать setState для уже удалённого компонента
        let cancelled = false;

        setIsLoading(true);
        setError(null);

        fetchTasks()
            .then((data) => {
                if (!cancelled) setTasks(data);
            })
            .catch((err) => {
                if (!cancelled) setError(err.message);
            })
            .finally(() => {
                if (!cancelled) setIsLoading(false);
            });

        // Функция очистки вызывается при размонтировании
        // и перед каждым повторным запуском эффекта
        return () => {
            cancelled = true;
        };
    }, [reloadKey]); // эффект перезапускается при смене reloadKey

    const reload = useCallback(() => setReloadKey((prev) => prev + 1), []);

    const addTask = useCallback((task) => {
        // Функциональная форма: React передаёт актуальное предыдущее
        // значение — это безопасно при нескольких обновлениях подряд
        setTasks((prev) => [
            ...prev,
            { ...task, id: Date.now(), isDone: false },
        ]);
    }, []);

    const toggleTask = useCallback((id) => {
        // map создаёт НОВЫЙ массив, а для нужного элемента — НОВЫЙ объект.
        // Прямая мутация (task.isDone = !task.isDone) не сработала бы:
        // ссылка на массив осталась бы прежней, React сравнил бы её
        // по Object.is, не увидел изменений и не стал бы перерисовывать
        setTasks((prev) =>
            prev.map((task) => (task.id === id ? { ...task, isDone: !task.isDone } : task))
        );
    }, []);

    const deleteTask = useCallback((id) => {
        setTasks((prev) => prev.filter((task) => task.id !== id));
    }, []);

    const updatePoints = useCallback((id, delta) => {
        setTasks((prev) =>
            prev.map((task) =>
                task.id === id
                    ? { ...task, points: Math.max(1, Math.min(100, task.points + delta)) }
                    : task
            )
        );
    }, []);

    return { tasks, isLoading, error, reload, addTask, toggleTask, deleteTask, updatePoints };
}
```

## src/components/TaskForm.jsx

```jsx
import { useState } from "react";

const EMPTY_FORM = { title: "", track: "Solidity", points: 10 };

function TaskForm({ onAdd }) {
    // Всё состояние формы — в одном объекте вместо трёх useState
    const [form, setForm] = useState(EMPTY_FORM);
    const [errors, setErrors] = useState({});

    // Один обработчик на все поля: имя поля берётся из атрибута name
    const handleChange = (event) => {
        const { name, value, type } = event.target;

        setForm((prev) => ({
            ...prev,
            // Вычисляемый ключ: имя свойства определяется в рантайме.
            // input всегда отдаёт строку, поэтому число приводим явно
            [name]: type === "number" ? Number(value) : value,
        }));
    };

    const handleSubmit = (event) => {
        // В React форма тоже перезагружает страницу по умолчанию
        event.preventDefault();

        const nextErrors = {};
        if (form.title.trim() === "") nextErrors.title = "Введите название задания";
        if (form.points < 1 || form.points > 100) nextErrors.points = "Баллы: от 1 до 100";

        if (Object.keys(nextErrors).length > 0) {
            setErrors(nextErrors);
            return;
        }

        onAdd({ ...form, title: form.title.trim() });
        setForm(EMPTY_FORM); // сброс формы
        setErrors({});
    };

    return (
        <form className="task-form" onSubmit={handleSubmit}>
            <div className="field">
                {/* Контролируемый компонент: value приходит из состояния,
                    а onChange возвращает изменения обратно в состояние.
                    Источник истины один — переменная form */}
                <input
                    type="text"
                    name="title"
                    value={form.title}
                    onChange={handleChange}
                    placeholder="Название задания"
                    className={errors.title ? "input input--error" : "input"}
                />
                {errors.title && <span className="field__error">{errors.title}</span>}
            </div>

            <select name="track" value={form.track} onChange={handleChange} className="input">
                <option value="Solidity">Solidity</option>
                <option value="Frontend">Frontend</option>
                <option value="DevOps">DevOps</option>
            </select>

            <div className="field">
                <input
                    type="number"
                    name="points"
                    value={form.points}
                    onChange={handleChange}
                    min="1"
                    max="100"
                    className={errors.points ? "input input--error" : "input"}
                />
                {errors.points && <span className="field__error">{errors.points}</span>}
            </div>

            <button type="submit" className="btn btn--primary">Добавить</button>
        </form>
    );
}

export default TaskForm;
```

## src/components/TaskItem.jsx

```jsx
function TaskItem({ task, onToggle, onDelete, onUpdatePoints }) {
    return (
        <li className={task.isDone ? "task task--done" : "task"}>
            <input
                type="checkbox"
                checked={task.isDone}
                // Колбэк пришёл из App — дочерний компонент не владеет состоянием,
                // он только сообщает наверх, что произошло
                onChange={() => onToggle(task.id)}
            />

            <span className="task__title">{task.title}</span>
            <span className="badge">{task.track}</span>

            <div className="task__points">
                <button type="button" onClick={() => onUpdatePoints(task.id, -5)}>−</button>
                <span>{task.points}</span>
                <button type="button" onClick={() => onUpdatePoints(task.id, +5)}>+</button>
            </div>

            <button type="button" className="task__delete" onClick={() => onDelete(task.id)}>
                ✕
            </button>
        </li>
    );
}

export default TaskItem;
```

## src/components/TaskList.jsx

```jsx
import TaskItem from "./TaskItem.jsx";

function TaskList({ tasks, onToggle, onDelete, onUpdatePoints }) {
    if (tasks.length === 0) {
        // Ранний return вместо тернарника в JSX — код читается проще
        return <p className="empty">Заданий не найдено</p>;
    }

    return (
        <ul className="task-list">
            {tasks.map((task) => (
                <TaskItem
                    key={task.id}
                    task={task}
                    onToggle={onToggle}
                    onDelete={onDelete}
                    onUpdatePoints={onUpdatePoints}
                />
            ))}
        </ul>
    );
}

export default TaskList;
```

## src/components/FilterBar.jsx

```jsx
const FILTERS = [
    { value: "all",    label: "Все"        },
    { value: "active", label: "Активные"   },
    { value: "done",   label: "Выполненные"},
];

function FilterBar({ filter, onFilterChange, search, onSearchChange }) {
    return (
        <div className="filter-bar">
            <input
                type="text"
                className="input"
                value={search}
                onChange={(event) => onSearchChange(event.target.value)}
                placeholder="Поиск по названию..."
            />

            <div className="filters">
                {FILTERS.map((item) => (
                    <button
                        key={item.value}
                        type="button"
                        className={filter === item.value ? "chip chip--active" : "chip"}
                        onClick={() => onFilterChange(item.value)}
                    >
                        {item.label}
                    </button>
                ))}
            </div>
        </div>
    );
}

export default FilterBar;
```

## src/components/StatsPanel.jsx

```jsx
import { calcProgress, sortTasks } from "../utils/taskUtils.js";

function StatsPanel({ tasks }) {
    // Производные значения считаются при каждом рендере.
    // Дублировать их в useState нельзя: состояние рассинхронизируется
    // с источником, и появятся баги «цифры не совпадают со списком»
    const total = tasks.length;
    const done = tasks.filter((task) => task.isDone).length;
    const progress = calcProgress(tasks);
    const totalPoints = tasks.reduce((sum, task) => sum + task.points, 0);
    const hardest = total > 0 ? sortTasks(tasks)[0] : null;

    return (
        <section className="stats">
            <div className="stats__row">
                <div className="stat"><b>{total}</b><span>всего</span></div>
                <div className="stat"><b>{done}</b><span>выполнено</span></div>
                <div className="stat"><b>{progress}%</b><span>прогресс</span></div>
                <div className="stat"><b>{totalPoints}</b><span>баллов</span></div>
            </div>

            <div className="progress">
                <div className="progress__bar" style={{ width: `${progress}%` }} />
            </div>

            {hardest && (
                <p className="stats__hint">
                    Самое дорогое: <b>{hardest.title}</b> ({hardest.points} баллов)
                </p>
            )}
        </section>
    );
}

export default StatsPanel;
```

## src/components/Timer.jsx

```jsx
import { useState, useEffect } from "react";
import { formatTime } from "../utils/taskUtils.js";

function Timer() {
    const [seconds, setSeconds] = useState(0);
    const [isRunning, setIsRunning] = useState(false);

    useEffect(() => {
        // Пока таймер на паузе, интервал вообще не создаётся
        if (!isRunning) {
            return;
        }

        const intervalId = setInterval(() => {
            // Функциональная форма обязательна: без неё замыкание
            // «запомнит» seconds = 0 с момента создания интервала,
            // и счётчик застрянет на единице
            setSeconds((prev) => prev + 1);
        }, 1000);

        // Функция очистки. Без clearInterval при каждом переключении
        // isRunning создавался бы новый интервал, а старые продолжали бы
        // работать — счётчик начал бы ускоряться, а после размонтирования
        // компонента остался бы висеть в памяти
        return () => clearInterval(intervalId);
    }, [isRunning]); // эффект перезапускается только при смене режима

    return (
        <section className="timer">
            <span className="timer__value">{formatTime(seconds)}</span>

            <button type="button" className="btn" onClick={() => setIsRunning((prev) => !prev)}>
                {isRunning ? "Пауза" : "Старт"}
            </button>

            <button
                type="button"
                className="btn"
                onClick={() => {
                    setIsRunning(false);
                    setSeconds(0);
                }}
            >
                Сброс
            </button>
        </section>
    );
}

export default Timer;
```

## src/App.jsx

```jsx
import { useState, useEffect } from "react";
import { useLocalTasks } from "./hooks/useLocalTasks.js";
import { filterTasks } from "./utils/taskUtils.js";
import TaskForm from "./components/TaskForm.jsx";
import TaskList from "./components/TaskList.jsx";
import FilterBar from "./components/FilterBar.jsx";
import StatsPanel from "./components/StatsPanel.jsx";
import Timer from "./components/Timer.jsx";
import "./App.css";

function App() {
    // Логика данных инкапсулирована в собственном хуке
    const { tasks, isLoading, error, reload, addTask, toggleTask, deleteTask, updatePoints } =
        useLocalTasks();

    // Состояние интерфейса живёт здесь: и FilterBar, и TaskList
    // зависят от filter/search, поэтому оно поднято до общего родителя
    const [filter, setFilter] = useState("all");
    const [search, setSearch] = useState("");

    // Эффект с зависимостью: реагирует на каждое изменение массива задач
    useEffect(() => {
        const activeCount = tasks.filter((task) => !task.isDone).length;
        document.title = `PrepTracker (${activeCount} активных)`;
    }, [tasks]);

    // Производное значение: не состояние, а результат вычисления при рендере
    const visibleTasks = filterTasks(tasks, filter, search);

    if (isLoading) {
        return <div className="loader">Загружаем задания...</div>;
    }

    if (error) {
        return (
            <div className="error-box">
                <p>Ошибка: {error}</p>
                <button type="button" className="btn btn--primary" onClick={reload}>
                    Повторить
                </button>
            </div>
        );
    }

    return (
        <div className="app">
            <header className="header">
                <h1>Трекер подготовки</h1>
                <Timer />
            </header>

            <main className="container">
                <StatsPanel tasks={tasks} />

                <TaskForm onAdd={addTask} />

                <FilterBar
                    filter={filter}
                    onFilterChange={setFilter}
                    search={search}
                    onSearchChange={setSearch}
                />

                <TaskList
                    tasks={visibleTasks}
                    onToggle={toggleTask}
                    onDelete={deleteTask}
                    onUpdatePoints={updatePoints}
                />
            </main>
        </div>
    );
}

export default App;
```

## Разбор решения

1. **Почему мутация не работает.** React решает, нужна ли перерисовка, сравнивая старое и новое значение состояния по ссылке (`Object.is`). `tasks.push(...)` меняет содержимое, но ссылка на массив остаётся прежней — сравнение даёт «равны», и перерисовки не будет. Поэтому каждое обновление создаёт новый массив (`[...prev, item]`, `map`, `filter`) и, при изменении элемента, новый объект (`{ ...task, isDone: !task.isDone }`).
2. **Функциональная форма `setState`.** `setTasks((prev) => ...)` получает гарантированно актуальное значение. Прямая форма `setTasks([...tasks, item])` опирается на переменную из замыкания: при нескольких обновлениях в одном обработчике или внутри `setInterval` она окажется устаревшей. В `Timer` это видно особенно наглядно — без функциональной формы счётчик застрял бы на единице.
3. **Массив зависимостей `useEffect`.** Пустой `[]` — эффект выполняется один раз при монтировании (загрузка данных). `[tasks]` — при каждом изменении задач (обновление `document.title`). `[isRunning]` — при смене режима таймера. Отсутствие массива вовсе означало бы запуск после каждого рендера, что здесь вызвало бы бесконечный цикл.
4. **Функция очистки.** Возвращаемая из `useEffect` функция вызывается перед повторным запуском эффекта и при размонтировании. Для `setInterval` это обязательно: иначе каждое переключение плодило бы новый интервал поверх старого, счётчик ускорялся бы, а после ухода со страницы таймер продолжал бы работать и обращаться к несуществующему компоненту.
5. **Флаг отмены в асинхронном эффекте.** Запрос длится 800 мс — за это время пользователь может уйти со страницы. `cancelled = true` в функции очистки не даёт вызвать `setState` для размонтированного компонента.
6. **Подъём состояния наверх.** `filter` нужен и `FilterBar` (подсветить активную кнопку), и `TaskList` (показать нужные элементы). Общий родитель — `App`, поэтому состояние живёт там, а вниз идут данные и колбэки. Дочерние компоненты остаются «глупыми» и переиспользуемыми.
7. **Производные значения не хранятся в состоянии.** `visibleTasks`, `progress`, `totalPoints` вычисляются при рендере из `tasks`. Если бы они лежали в `useState`, пришлось бы синхронизировать их вручную при каждом изменении — источник рассогласования, при котором статистика отстаёт от списка.
8. **Один объект состояния для формы.** Вместо трёх `useState` — один объект и один обработчик. Вычисляемый ключ `[name]: value` связывает атрибут `name` поля с полем объекта, и добавление нового поля не требует нового обработчика.
9. **Контролируемые компоненты.** `value` приходит из состояния, `onChange` возвращает изменения обратно. Единственный источник истины — переменная состояния, поэтому программный сброс формы (`setForm(EMPTY_FORM)`) мгновенно очищает все поля.
10. **Собственный хук.** `useLocalTasks` инкапсулирует состояние, загрузку и все операции. `App` превращается в композицию компонентов, а логику можно переиспользовать в другом месте приложения или покрыть тестами отдельно от разметки.

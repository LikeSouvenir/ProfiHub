# Модуль 4: Продвинутые объекты и Promise — Эталонное решение

## taskApi.js

```js
"use strict";

// ============================================================
// 1. ЗАМОРОЖЕННАЯ КОНФИГУРАЦИЯ
// ============================================================

const CONFIG = Object.freeze({
    baseUrl: "https://api.profihub.local/tasks",
    timeout: 5000,
    retries: 3,
});

// Object.freeze запрещает добавление, удаление и изменение свойств.
// В нестрогом режиме присваивание молча игнорируется, в "use strict" — бросает TypeError.
try {
    CONFIG.timeout = 99999;
} catch (error) {
    console.log("Попытка изменить CONFIG отклонена:", error.message);
}
console.log("timeout остался прежним:", CONFIG.timeout); // 5000

// ============================================================
// 2. БАЗА ДАННЫХ В ПАМЯТИ
// ============================================================

const tasksDb = [
    { id: 1, title: "Реестр студентов",     track: "solidity", difficulty: "easy",   points: 10, isSolved: true,  author: { name: "Алина", group: "BC-21" } },
    { id: 2, title: "Ролевой доступ",       track: "solidity", difficulty: "medium", points: 25, isSolved: false, author: { name: "Алина", group: "BC-21" } },
    { id: 3, title: "Временной депозитарий",track: "solidity", difficulty: "hard",   points: 40, isSolved: false, author: { name: "Данияр", group: "BC-22" } },
    { id: 4, title: "Лендинг команды",      track: "frontend", difficulty: "easy",   points: 10, isSolved: true,  author: { name: "Иван", group: "BC-21" } },
    { id: 5, title: "Трекер заданий",       track: "frontend", difficulty: "medium", points: 20, isSolved: false, author: { name: "Иван", group: "BC-21" } },
    { id: 6, title: "Сборка ноды в Docker", track: "devops",   difficulty: "hard",   points: 35, isSolved: false, author: { name: "Мария", group: "BC-22" } },
];

// ============================================================
// 3. СЕРВИС: МЕТОДЫ ОБЪЕКТА И this
// ============================================================

const taskService = {
    db: tasksDb,

    // Сокращённый синтаксис метода: getAll() вместо getAll: function ()
    getAll() {
        // Spread создаёт поверхностную копию: внешний код не сможет случайно
        // испортить исходный массив через push/splice
        return [...this.db];
    },

    getByTrack(track) {
        return this.db.filter((task) => task.track === track);
    },

    getTotalPoints() {
        // reduce сворачивает массив в одно значение; 0 — начальное значение аккумулятора
        return this.db.reduce((sum, task) => sum + task.points, 0);
    },

    getHardest() {
        return this.db.reduce((max, task) => (task.points > max.points ? task : max));
    },

    getSolvedCount() {
        return this.db.filter((task) => task.isSolved).length;
    },

    // sort мутирует массив, поэтому сортируем копию, а не this.db
    getSortedByPoints() {
        return [...this.db].sort((a, b) => b.points - a.points);
    },
};

console.log("\n=== Сервис ===");
console.log("Всего заданий:", taskService.getAll().length);
console.log("Solidity-заданий:", taskService.getByTrack("solidity").length);
console.log("Сумма баллов:", taskService.getTotalPoints());
console.log("Самое дорогое:", taskService.getHardest().title);
console.log("Решено:", taskService.getSolvedCount());

// some / every — быстрые проверки без ручного цикла
console.log("Есть нерешённые hard:", tasksDb.some((t) => t.difficulty === "hard" && !t.isSolved));
console.log("Все задания дороже 5 баллов:", tasksDb.every((t) => t.points > 5));

// find возвращает первый подходящий элемент или undefined
console.log("Первое frontend-задание:", tasksDb.find((t) => t.track === "frontend")?.title);

// ============================================================
// 4. ДЕСТРУКТУРИЗАЦИЯ
// ============================================================

/**
 * Объект разбирается прямо в списке параметров, включая вложенный author.
 * Внутри функции сразу доступны title, points и name — без task.author.name.
 */
function describeTask({ title, points, author: { name } }) {
    return `"${title}" — ${points} баллов, автор: ${name}`;
}

console.log("\n=== Деструктуризация ===");
console.log(describeTask(tasksDb[2]));

// Переименование (title -> taskTitle) и значение по умолчанию для отсутствующего поля
const { title: taskTitle, comment = "без комментария" } = tasksDb[0];
console.log(`${taskTitle} (${comment})`);

// Массив: берём первый элемент, пропускаем второй, остальные собираем в others
const [first, , ...others] = tasksDb;
console.log("Первое:", first.title, "| остальных:", others.length);

// Деструктуризация в цикле — частый приём при переборе массива объектов
for (const { id, title, difficulty } of tasksDb) {
    console.log(`#${id} ${title} [${difficulty}]`);
}

// ============================================================
// 5. SPREAD И REST
// ============================================================

/**
 * Иммутабельное обновление: свойства из changes перекрывают одноимённые
 * свойства task, а исходный объект остаётся нетронутым.
 */
function updateTask(task, changes) {
    return { ...task, ...changes };
}

const original = tasksDb[1];
const updated = updateTask(original, { isSolved: true, points: 30 });

console.log("\n=== Spread ===");
console.log("Оригинал:", original.isSolved, original.points); // false 25 — не изменился
console.log("Копия:   ", updated.isSolved, updated.points);   // true 30

// Rest-параметр собирает любое число аргументов в массив
function sumPoints(...values) {
    return values.reduce((sum, value) => sum + value, 0);
}

console.log("Сумма произвольных баллов:", sumPoints(10, 25, 40, 5)); // 80

// Rest при деструктуризации объекта: вынимаем author, остальное складываем в rest
const { author, ...taskWithoutAuthor } = tasksDb[0];
console.log("Без автора:", Object.keys(taskWithoutAuthor));

// ============================================================
// 6. ВЫЧИСЛЯЕМЫЕ КЛЮЧИ
// ============================================================

/**
 * Группировка массива объектов по значению поля.
 * Имя ключа неизвестно заранее, поэтому используется синтаксис [computedKey].
 */
function groupBy(items, key) {
    const result = {};

    for (const item of items) {
        const groupKey = item[key];

        if (!result[groupKey]) {
            result[groupKey] = [];
        }

        result[groupKey].push(item);
    }

    return result;
}

const byTrack = groupBy(tasksDb, "track");

console.log("\n=== Группировка ===");
// Object.entries даёт массив пар [ключ, значение] — его удобно деструктурировать
for (const [track, items] of Object.entries(byTrack)) {
    console.log(`${track}: ${items.length} шт. (${items.map((t) => t.title).join(", ")})`);
}

// Вычисляемый ключ при создании объекта
const statKey = "totalPoints";
const stats = {
    [statKey]: taskService.getTotalPoints(),
    [`solved_${new Date().getFullYear()}`]: taskService.getSolvedCount(),
};
console.log("Статистика:", stats);

// ============================================================
// 7. ИМИТАЦИЯ ЗАПРОСОВ
// ============================================================

/**
 * Промис имитирует сетевой запрос: через delay мс он либо резолвится
 * копией задания, либо реджектится ошибкой.
 */
function fetchTaskWithDelay(id, delay = 500) {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            const task = tasksDb.find((t) => t.id === id);

            if (task) {
                resolve({ ...task }); // отдаём копию, чтобы вызывающий код не менял базу
            } else {
                // В reject всегда передаём объект Error: у него есть stack и message
                reject(new Error(`Task ${id} not found`));
            }
        }, delay);
    });
}

const fetchTask = (id) => fetchTaskWithDelay(id, 500);

/** Имитация сохранения изменений */
function saveTask(task) {
    return new Promise((resolve) => {
        setTimeout(() => resolve({ ...task, savedAt: new Date().toISOString() }), 300);
    });
}

// ============================================================
// 8. THEN / CATCH / FINALLY
// ============================================================

console.log("\n=== Одиночный запрос ===");

fetchTask(3)
    .then((task) => describeTask(task))   // результат then передаётся дальше по цепочке
    .then((text) => console.log("Успех:", text))
    .catch((error) => console.error("Ошибка:", error.message))
    .finally(() => console.log("Запрос завершён"));

// Тот же вызов с несуществующим id — сработает catch, но finally выполнится всё равно
fetchTask(999)
    .then((task) => console.log("Успех:", describeTask(task)))
    .catch((error) => console.error("Ошибка:", error.message))
    .finally(() => console.log("Запрос завершён"));

// ============================================================
// 9. ЦЕПОЧКА ПРОМИСОВ
// ============================================================

console.log("\n=== Цепочка: получить -> обновить -> сохранить ===");

fetchTask(2)
    .then((task) => {
        console.log("1. Получено:", task.title);
        return updateTask(task, { isSolved: true }); // обычное значение тоже уйдёт дальше
    })
    .then((updatedTask) => {
        console.log("2. Помечено решённым");
        return saveTask(updatedTask); // возвращаем промис — следующий then ждёт его
    })
    .then((savedTask) => {
        console.log("3. Сохранено в", savedTask.savedAt);
    })
    .catch((error) => console.error("Цепочка прервана:", error.message));

// ============================================================
// 10. ПАРАЛЛЕЛЬНЫЕ ЗАПРОСЫ
// ============================================================

// --- Promise.all: все запросы стартуют одновременно ---
console.log("\n=== Promise.all ===");
const startTime = Date.now();

Promise.all([fetchTask(1), fetchTask(4), fetchTask(6)])
    .then((tasks) => {
        const titles = tasks.map((task) => task.title);
        console.log("Загружено:", titles.join(" | "));
        // Три запроса по 500 мс заняли ~500 мс, а не 1500: они шли параллельно
        console.log(`Время: ~${Date.now() - startTime} мс`);
    })
    .catch((error) => console.error("Ошибка пакета:", error.message));

// --- Promise.all с одним отклонённым: падает весь набор ---
Promise.all([fetchTask(1), fetchTask(777), fetchTask(6)])
    .then((tasks) => console.log("Сюда не попадём:", tasks.length))
    .catch((error) => console.error("Promise.all упал целиком:", error.message));

// --- Promise.allSettled: ждёт все и сообщает статус каждого ---
Promise.allSettled([fetchTask(1), fetchTask(777), fetchTask(6)]).then((results) => {
    console.log("\n=== Promise.allSettled ===");

    results.forEach((result, index) => {
        if (result.status === "fulfilled") {
            console.log(`[${index}] ok: ${result.value.title}`);
        } else {
            console.log(`[${index}] fail: ${result.reason.message}`);
        }
    });
});

// --- Promise.race: побеждает тот, кто ответил первым ---
Promise.race([
    fetchTaskWithDelay(1, 900),
    fetchTaskWithDelay(5, 200),
]).then((task) => {
    console.log("\nPromise.race победитель:", task.title); // Трекер заданий (200 мс)
});

// ============================================================
// 11. СОБСТВЕННЫЙ ПРОМИС
// ============================================================

function checkEven(number) {
    return new Promise((resolve, reject) => {
        // Number.isNaN нужен отдельно: typeof NaN === "number"
        if (typeof number !== "number" || Number.isNaN(number)) {
            reject(new Error("Аргумент должен быть числом"));
            return;
        }

        setTimeout(() => {
            resolve(number % 2 === 0 ? `${number} — чётное` : `${number} — нечётное`);
        }, 200);
    });
}

checkEven(10).then((msg) => console.log("\ncheckEven:", msg));
checkEven("десять").catch((error) => console.error("checkEven:", error.message));
```

## Разбор решения

1. **`Object.freeze` для конфигурации.** Настройки, которые не должны меняться в рантайме, безопаснее заморозить. В строгом режиме попытка записи бросает `TypeError` — ошибка проявляется сразу, а не через час отладки. Заморозка поверхностная: вложенные объекты остаются изменяемыми.
2. **Копия вместо ссылки.** `getAll()` возвращает `[...this.db]`. Если отдать саму ссылку, любой внешний `sort` или `push` незаметно изменит базу — классический источник трудноуловимых багов. По той же причине `getSortedByPoints` сортирует копию: `sort` мутирует массив на месте.
3. **`this` в методах объекта.** Внутри метода, вызванного как `taskService.getAll()`, `this` указывает на сам объект. Именно поэтому здесь используется обычный синтаксис метода, а не стрелочная функция: у стрелочной нет собственного `this`, и она взяла бы его из внешней области.
4. **Деструктуризация в параметрах.** `describeTask({ title, points, author: { name } })` избавляет от многократного `task.author.name` в теле. Переименование (`title: taskTitle`) и значения по умолчанию (`comment = "..."`) страхуют от `undefined`.
5. **Spread как инструмент иммутабельности.** `{ ...task, ...changes }` собирает новый объект: свойства из второго спреда перекрывают одноимённые из первого. Исходные данные остаются нетронутыми — тот же принцип лежит в основе обновления состояния в React.
6. **Rest в двух ролях.** В параметрах функции (`...values`) он собирает аргументы в массив; при деструктуризации (`{ author, ...rest }`) — собирает оставшиеся свойства. Синтаксис один, направление разное.
7. **Вычисляемые ключи.** `{ [groupKey]: [] }` позволяет строить объект, имена полей которого известны только в рантайме. Без этого синтаксиса пришлось бы создавать пустой объект и дописывать ключи в отдельной строке.
8. **`reject(new Error(...))`, а не строка.** Объект `Error` несёт `message` и стек вызовов; строка не даст ни того, ни другого при отладке.
9. **`finally` выполняется всегда.** И после `then`, и после `catch` — это место для снятия индикатора загрузки или закрытия соединения.
10. **`all` против `allSettled` против `race`.** `Promise.all` падает целиком при первом же отказе — подходит, когда без всех данных работать нельзя. `allSettled` дожидается всех и сообщает статус каждого — подходит, когда часть данных лучше, чем ничего. `race` возвращает первый завершившийся промис — типичное применение: таймаут запроса.
11. **Параллельно, а не последовательно.** Три запроса по 500 мс в `Promise.all` укладываются примерно в 500 мс, потому что стартуют одновременно. Последовательная цепочка `then` дала бы 1500 мс — разница, хорошо заметная на замере `Date.now()`.

# Практика: магазин музыки и Promise

## Часть 1. Объект музыкального магазина

### Условие
Создать объект магазина музыки с полями: окна, дверь, массив музыки. Добавить функцию получения всей музыки и конкретной музыки по названию, а также функцию добавления нового трека.

```js
const musicStore = {
    windows: 3,
    door: "деревянная",
    music: [
        { title: "Bohemian Rhapsody", artist: "Queen" },
        { title: "Shape of You", artist: "Ed Sheeran" },
        { title: "Billie Jean", artist: "Michael Jackson" },
    ],

    // Метод: получить всю музыку целиком
    getAllMusic() {
        return this.music;
    },

    // Метод: получить конкретный трек по названию
    getMusicByTitle(title) {
        // find возвращает первый подходящий объект или undefined, если не найден
        const track = this.music.find(item => item.title === title);

        if (!track) {
            console.log(`Трек "${title}" не найден в магазине`);
            return null;
        }

        return track;
    },

    // Метод: добавить новый трек в магазин
    addTrack(title, artist) {
        this.music.push({ title, artist });
        console.log(`Добавлен новый трек: "${title}" — ${artist}`);
    },
};

// ==== Проверка работы ====

console.log(musicStore.getAllMusic());
// [{title: "Bohemian Rhapsody", ...}, {title: "Shape of You", ...}, {title: "Billie Jean", ...}]

console.log(musicStore.getMusicByTitle("Shape of You"));
// { title: "Shape of You", artist: "Ed Sheeran" }

console.log(musicStore.getMusicByTitle("Несуществующий трек"));
// выведет предупреждение в консоль и вернёт null

musicStore.addTrack("Imagine", "John Lennon");
console.log(musicStore.getAllMusic());
// теперь в массиве уже 4 трека, включая новый "Imagine"
```

### Разбор решения
- `windows` и `door` — обычные поля объекта, описывающие "магазин" как физическое пространство (по условию задачи).
- `music` — массив объектов-треков, каждый со своими `title` и `artist`.
- `getAllMusic()` — простой метод-геттер, возвращающий весь массив `this.music` целиком.
- `getMusicByTitle(title)` — используется метод массива `find`, который проходит по `this.music` и возвращает первый элемент, чей `title` совпал с переданным — а если совпадений нет, возвращает `undefined`, что обрабатывается отдельной проверкой.
- `addTrack(title, artist)` — принимает данные нового трека, оборачивает их в объект и добавляет в конец массива через `push`, изменяя сам объект `musicStore` (метод обращается к `this.music`, то есть к массиву того же объекта, у которого он был вызван).

---

## Часть 2. Promise на проверку чётности числа

### Условие
Создать функцию, принимающую число и возвращающую Promise с `resolve(число % 2 === 0)` в случае успеха, `reject` в ином случае, вывести в консоль результат.

```js
function checkEvenNumber(number) {
    return new Promise(function (resolve, reject) {
        if (typeof number !== "number" || Number.isNaN(number)) {
            reject("Ошибка: передано не число");
            return;
        }

        if (number % 2 === 0) {
            resolve(true); // число чётное — успех
        } else {
            reject(false); // число нечётное — считаем это причиной для reject
        }
    });
}

// ==== Проверка работы ====

checkEvenNumber(10)
    .then(function (result) {
        console.log("Результат:", result); // "Результат: true"
    })
    .catch(function (error) {
        console.log("Отклонено:", error);
    });

checkEvenNumber(7)
    .then(function (result) {
        console.log("Результат:", result);
    })
    .catch(function (error) {
        console.log("Отклонено:", error); // "Отклонено: false"
    });
```

### Вариант через async/await
```js
async function runCheck(number) {
    try {
        const result = await checkEvenNumber(number);
        console.log(`Число ${number} — чётное, результат: ${result}`);
    } catch (error) {
        console.log(`Число ${number} — нечётное, причина отклонения: ${error}`);
    }
}

runCheck(4);  // "Число 4 — чётное, результат: true"
runCheck(9);  // "Число 9 — нечётное, причина отклонения: false"
```

### Разбор решения
- Внутри исполнителя `Promise` сначала идёт защитная проверка типа входного значения — если передано не число, промис сразу отклоняется с понятным сообщением об ошибке.
- Основная логика — `number % 2 === 0`: если остаток от деления на 2 равен нулю, число чётное, вызывается `resolve(true)`. В противном случае вызывается `reject(false)`, переводя промис в отклонённое состояние (по условию задачи именно такое поведение и требуется).
- В блоке `.then()` обрабатывается успешный сценарий (чётное число), в `.catch()` — отклонённый (нечётное число или ошибка типа).

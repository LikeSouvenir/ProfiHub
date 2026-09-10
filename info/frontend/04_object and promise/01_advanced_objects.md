# Объекты: продвинутое использование

## Методы объекта и ключевое слово this
Внутри метода `this` ссылается на тот объект, у которого этот метод был вызван:
```js
const store = {
    name: "Музыкальный магазин",
    tracksCount: 0,

    addTrack() {
        this.tracksCount++; // this === store
        console.log(this.name + ": треков теперь " + this.tracksCount);
    },
};

store.addTrack(); // "Музыкальный магазин: треков теперь 1"
```
*Важно: `this` определяется тем, **как** вызвана функция, а не тем, где она объявлена. Если метод "оторвать" от объекта и вызвать отдельно, `this` потеряется:*
```js
const addTrack = store.addTrack;
addTrack(); // Ошибка — this здесь уже не store
```

## Сокращённая запись свойств (shorthand)
Если имя переменной совпадает с именем нужного свойства, можно не дублировать его:
```js
const name = "Гитара";
const price = 300;

// Вместо { name: name, price: price }
const product = { name, price };
console.log(product); // { name: "Гитара", price: 300 }
```

## Деструктуризация объектов
Позволяет быстро "распаковать" нужные свойства объекта в отдельные переменные:
```js
const track = { title: "Bohemian Rhapsody", artist: "Queen", year: 1975 };

const { title, artist } = track;
console.log(title);  // "Bohemian Rhapsody"
console.log(artist); // "Queen"

// Деструктуризация с переименованием переменной
const { title: songTitle } = track;
console.log(songTitle); // "Bohemian Rhapsody"

// Деструктуризация со значением по умолчанию (если свойства нет в объекте)
const { genre = "неизвестно" } = track;
console.log(genre); // "неизвестно"
```

## Деструктуризация в параметрах функции
Удобно, когда функции нужен не весь объект, а лишь пара его полей:
```js
function printTrack({ title, artist }) {
    console.log(`${title} — ${artist}`);
}

printTrack(track); // "Bohemian Rhapsody — Queen"
```

## Spread-оператор (...) для объектов
Копирует свойства одного объекта в другой — полезно для создания копии с изменёнными полями, не трогая исходный объект:
```js
const track = { title: "Track 1", year: 2020 };

// Создаём новый объект: копия track, но с изменённым year
const updatedTrack = { ...track, year: 2024 };

console.log(track);        // { title: "Track 1", year: 2020 } — не изменился
console.log(updatedTrack); // { title: "Track 1", year: 2024 }
```

## Rest-оператор при деструктуризации
Собирает "всё остальное", что не было явно перечислено:
```js
const track = { title: "Track 1", artist: "Band", year: 2020, genre: "Rock" };

const { title, ...rest } = track;
console.log(title); // "Track 1"
console.log(rest);  // { artist: "Band", year: 2020, genre: "Rock" }
```

## Вычисляемые ключи объекта
Ключ можно задать динамически через выражение в квадратных скобках:
```js
const field = "price";

const product = {
    name: "Микрофон",
    [field]: 150, // ключ будет равен значению переменной field, то есть "price"
};

console.log(product); // { name: "Микрофон", price: 150 }
```

## Массивы объектов и методы работы с ними
```js
const tracks = [
    { title: "Track A", duration: 210 },
    { title: "Track B", duration: 180 },
    { title: "Track C", duration: 250 },
];

// find — вернёт первый подходящий объект (или undefined)
const found = tracks.find(t => t.title === "Track B");
console.log(found); // { title: "Track B", duration: 180 }

// filter — вернёт массив со всеми подходящими объектами
const longTracks = tracks.filter(t => t.duration > 200);
console.log(longTracks); // [{ title: "Track A", ... }, { title: "Track C", ... }]

// map — создаёт новый массив, преобразуя каждый объект
const titles = tracks.map(t => t.title);
console.log(titles); // ["Track A", "Track B", "Track C"]

// some / every — проверка условия по всему массиву
console.log(tracks.some(t => t.duration > 240)); // true — есть хотя бы один
console.log(tracks.every(t => t.duration > 100)); // true — все подходят
```

## Object.freeze — запрет изменений
```js
const config = Object.freeze({ apiUrl: "https://api.example.com" });

config.apiUrl = "https://hacked.com"; // изменение молча игнорируется (в строгом режиме — ошибка)
console.log(config.apiUrl); // "https://api.example.com" — не изменилось
```
Используется для объектов-констант, которые не должны меняться в процессе работы программы (например, конфигурация приложения).

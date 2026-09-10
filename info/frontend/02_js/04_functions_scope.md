# Функции и области видимости

## Что такое функция
Функция — это именованный блок кода, который можно **вызвать** сколько угодно раз, не переписывая логику заново.

## Способы объявления функций

### 1. Function Declaration (функция-объявление)
```js
function greet(name) {
    return "Привет, " + name + "!";
}

console.log(greet("Алина")); // "Привет, Алина!"
```
Особенность: такие функции **поднимаются вверх (hoisting)** — их можно вызвать даже до места объявления в коде.

### 2. Function Expression (функция-выражение)
```js
const greet = function(name) {
    return "Привет, " + name + "!";
};
```
В отличие от Declaration, такую функцию нельзя вызвать до её объявления — переменная `greet` до этой строки ещё не содержит функцию.

### 3. Стрелочная функция (Arrow Function)
Более короткий, современный синтаксис:
```js
const greet = (name) => {
    return "Привет, " + name + "!";
};

// Если тело функции — одно выражение, можно опустить {} и return
const greetShort = (name) => "Привет, " + name + "!";

// Если параметр всего один — скобки вокруг него тоже можно опустить
const square = x => x * x;

console.log(square(5)); // 25
```

## Параметры и аргументы
```js
function sum(a, b) {
    return a + b;
}
console.log(sum(2, 3)); // 5

// Параметр по умолчанию, если аргумент не передан
function sayHi(name = "Гость") {
    console.log("Привет, " + name);
}
sayHi();          // "Привет, Гость"
sayHi("Иван");    // "Привет, Иван"

// Произвольное количество аргументов (rest-параметр)
function sumAll(...numbers) {
    return numbers.reduce((total, n) => total + n, 0);
}
console.log(sumAll(1, 2, 3, 4)); // 10
```

## Возврат значения: return
```js
function isEven(number) {
    return number % 2 === 0; // сразу выходим из функции с результатом
}

console.log(isEven(4)); // true
console.log(isEven(7)); // false
```
Если `return` не указан явно, функция вернёт `undefined`.

## Области видимости (scope)

### Глобальная область видимости
Переменная, объявленная вне функций, доступна из любого места программы:
```js
let globalVar = "Я глобальная";

function show() {
    console.log(globalVar); // доступна внутри функции
}
show();
```

### Функциональная область видимости
Переменная, объявленная внутри функции через `let`/`const`, недоступна снаружи:
```js
function myFunction() {
    let localVar = "Я локальная";
    console.log(localVar); // работает
}

myFunction();
console.log(localVar); // Ошибка: localVar is not defined
```

### Блочная область видимости
`let` и `const` (в отличие от устаревшего `var`) видны только внутри блока `{ }`, в котором объявлены:
```js
if (true) {
    let blockVar = "Видна только тут";
    console.log(blockVar); // работает
}
console.log(blockVar); // Ошибка: blockVar is not defined

for (let i = 0; i < 3; i++) {
    // i видна только внутри тела цикла
}
console.log(i); // Ошибка: i is not defined
```

### var vs let: разница в области видимости
```js
if (true) {
    var oldVar = "var не уважает блоки";
}
console.log(oldVar); // "var не уважает блоки" — доступна снаружи блока!

if (true) {
    let newVar = "let уважает блоки";
}
console.log(newVar); // Ошибка — недоступна снаружи
```
Именно из-за таких неожиданных "утечек" видимости `var` считается устаревшим и в современном коде не используется — вместо него всегда `let` или `const`.

## Вложенные функции и замыкание (кратко)
Функция, объявленная внутри другой функции, "видит" переменные внешней функции даже после того, как внешняя функция завершила выполнение — это называется **замыканием (closure)**:
```js
function createCounter() {
    let count = 0;

    return function () {
        count++;
        return count;
    };
}

const counter = createCounter();
console.log(counter()); // 1
console.log(counter()); // 2
console.log(counter()); // 3
```
Каждый вызов `counter()` "помнит" своё собственное значение `count` — оно как бы "заперто" внутри функции.

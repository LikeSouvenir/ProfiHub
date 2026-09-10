# Пример работы с объектами

## Что такое объект
Объект — это структура данных, которая хранит информацию в виде пар **ключ: значение**. Удобен, когда нужно объединить связанные между собой данные в одну сущность.

## Создание объекта
```js
const user = {
    name: "Алина",
    age: 20,
    isStudent: true,
};
```

## Доступ к свойствам

### Точечная нотация
```js
console.log(user.name); // "Алина"
console.log(user.age);  // 20
```

### Квадратные скобки (полезно, когда ключ хранится в переменной)
```js
console.log(user["name"]); // "Алина"

const key = "age";
console.log(user[key]); // 20 — ключ подставляется динамически
```

## Изменение и добавление свойств
```js
user.age = 21;             // изменить существующее свойство
user.city = "Астана";      // добавить новое свойство

console.log(user);
// { name: "Алина", age: 21, isStudent: true, city: "Астана" }
```

## Удаление свойства
```js
delete user.isStudent;
console.log(user); // isStudent больше нет в объекте
```

## Методы объекта (функции внутри объекта)
```js
const person = {
    name: "Иван",
    age: 30,

    // Метод — функция, объявленная как свойство объекта
    greet: function () {
        return "Привет, меня зовут " + this.name;
    },

    // Сокращённый синтаксис метода (современный стандарт)
    sayAge() {
        return "Мне " + this.age + " лет";
    },
};

console.log(person.greet());  // "Привет, меня зовут Иван"
console.log(person.sayAge()); // "Мне 30 лет"
```
`this` внутри метода ссылается на сам объект, у которого этот метод вызван.

## Перебор свойств объекта
```js
const car = { brand: "Toyota", model: "Corolla", year: 2022 };

for (const key in car) {
    console.log(key + ": " + car[key]);
}
// brand: Toyota
// model: Corolla
// year: 2022
```

## Полезные методы Object
```js
console.log(Object.keys(car));   // ["brand", "model", "year"] — массив всех ключей
console.log(Object.values(car)); // ["Toyota", "Corolla", 2022] — массив всех значений
console.log(Object.entries(car));
// [["brand", "Toyota"], ["model", "Corolla"], ["year", 2022]] — массив пар [ключ, значение]
```

## Вложенные объекты
Значением свойства может быть другой объект — так строятся более сложные структуры данных:
```js
const employee = {
    name: "Мария",
    position: "Дизайнер",
    address: {
        city: "Алматы",
        street: "Абая, 10",
    },
};

console.log(employee.address.city); // "Алматы"
```

## Объекты и массивы вместе
```js
const students = [
    { name: "Алина", grade: 5 },
    { name: "Иван", grade: 4 },
    { name: "Мария", grade: 5 },
];

for (const student of students) {
    console.log(student.name + " — " + student.grade);
}
```

---

## Практика: разбор мини-задачи "Каталог товаров"

**Задача:** создать массив объектов-товаров, посчитать общую стоимость и вывести названия только тех товаров, которых на складе меньше 5 штук.

```js
const products = [
    { name: "Клавиатура", price: 25, quantity: 10 },
    { name: "Мышь", price: 15, quantity: 3 },
    { name: "Монитор", price: 150, quantity: 2 },
    { name: "Наушники", price: 40, quantity: 8 },
];

// 1. Считаем общую стоимость всего склада (цена * количество для каждого товара)
let totalValue = 0;

for (const product of products) {
    totalValue += product.price * product.quantity;
}

console.log("Общая стоимость склада: " + totalValue); // 25*10 + 15*3 + 150*2 + 40*8 = 915

// 2. Находим товары, которых мало на складе (< 5 штук)
const lowStock = [];

for (const product of products) {
    if (product.quantity < 5) {
        lowStock.push(product.name);
    }
}

console.log("Заканчиваются на складе:", lowStock); // ["Мышь", "Монитор"]

// 3. Выводим полную информацию о каждом товаре в удобном формате
for (const product of products) {
    console.log(
        `${product.name}: цена ${product.price}, остаток ${product.quantity} шт.`
    );
}
```

### Разбор решения
1. Цикл `for...of` перебирает массив объектов `products`, обращаясь к каждому товару через точечную нотацию (`product.price`, `product.quantity`).
2. Условие `product.quantity < 5` внутри цикла — обычное ветвление, которое отбирает нужные товары в отдельный массив `lowStock`.
3. Шаблонная строка (`` `${...}` ``) удобно подставляет значения свойств объекта прямо в текст вывода, без склейки через `+`.

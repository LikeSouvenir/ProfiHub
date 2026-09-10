# Разбор практического задания: Объявление переменных, структур, маппингов и циклов

Ниже представлено полное решение практического задания с подробными комментариями к каждой строке кода. Этот контракт является отличной шпаргалкой по базовому синтаксису Solidity.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract PracticeTask {
    
    // ==========================================
    // 1. ОБЪЯВЛЕНИЕ ПЕРЕМЕННЫХ И ИНИЦИАЛИЗАЦИЯ
    // ==========================================
    
    // Число, строка, булевое значение, адрес
    uint256 public myNumber = 42;
    string public myString = "Hello ProfiHub";
    bool public myBool = true;
    address public myAddress = msg.sender; // msg.sender - адрес того, кто вызывает или деплоит контракт
    
    // Динамический массив чисел
    uint256[] public myArray;

    // Структура с 4 значениями
    struct Person {
        string name;
        uint256 age;
        address wallet;
        bool isStudent;
    }
    
    // ==========================================
    // 2. MAPPING К СТРУКТУРЕ
    // ==========================================
    
    // Mapping, который по числу (ID) ищет данные из структуры Person
    mapping(uint256 => Person) public people;
    
    // ==========================================
    // 3. ФУНКЦИИ ДЛЯ ВОЗВРАТА ЗНАЧЕНИЙ ПЕРЕМЕННЫХ
    // ==========================================
    
    // Solidity автоматически создает геттеры для переменных с модификатором public (например, myNumber()), 
    // но по заданию мы напишем их явно:
    
    function getNumber() public view returns (uint256) {
        return myNumber;
    }
    
    // Строки требуют указания memory в returns
    function getString() public view returns (string memory) {
        return myString;
    }
    
    // ==========================================
    // 4. УПРАВЛЕНИЕ СТРУКТУРАМИ ЧЕРЕЗ MAPPING
    // ==========================================
    
    // Функция добавления значения в структуру
    // Принимает аргументы для создания структуры и сохраняет ее в mapping по указанному ID
    function addPerson(uint256 _id, string memory _name, uint256 _age, bool _isStudent) public {
        people[_id] = Person({
            name: _name,
            age: _age,
            wallet: msg.sender,
            isStudent: _isStudent
        });
    }
    
    // Функция получения значения из структуры по ID
    // Так как структура состоит из нескольких полей, возвращаем их все через запятую.
    // Структуры целиком из storage можно вернуть, указав тип (Person memory), но мы распишем поля:
    function getPerson(uint256 _id) public view returns (string memory, uint256, address, bool) {
        // Забираем нужную персону в память для чтения
        Person memory p = people[_id];
        return (p.name, p.age, p.wallet, p.isStudent);
    }
    
    // ==========================================
    // 5. ДОБАВЛЕНИЕ И УДАЛЕНИЕ ИЗ МАССИВА
    // ==========================================
    
    // Добавление в конец массива (push)
    function addToArray(uint256 _value) public {
        myArray.push(_value);
    }
    
    // Удаление последнего значения из массива (pop)
    function removeLastFromArray() public {
        // Обязательная проверка, чтобы не получить ошибку, если массив уже пуст
        require(myArray.length > 0, "Array is empty");
        myArray.pop();
    }
    
    // ==========================================
    // 6. ЗАДАНИЕ СО ЗВЕЗДОЧКОЙ (ЦИКЛЫ)
    // ==========================================
    
    // Объявляем дополнительные массивы для тестов циклов
    uint256[] public forArray;
    uint256[] public whileArray;
    uint256[] public doWhileArray;

    // Цикл FOR: Добавляем 5 элементов
    function populateWithFor() public returns (uint256[] memory) {
        for(uint256 i = 0; i < 5; i++) {
            forArray.push(i);
        }
        return forArray;
    }

    // Цикл WHILE: Добавляем 5 элементов
    function populateWithWhile() public returns (uint256[] memory) {
        uint256 i = 0;
        while (i < 5) {
            whileArray.push(i);
            i++;
        }
        return whileArray;
    }

    // Цикл DO-WHILE: Добавляем 5 элементов
    function populateWithDoWhile() public returns (uint256[] memory) {
        uint256 i = 0;
        do {
            doWhileArray.push(i);
            i++;
        } while (i < 5);
        return doWhileArray;
    }
}
```

### Как это тестировать:
Вы можете скопировать этот код в [Remix IDE](https://remix.ethereum.org/), скомпилировать и задеплоить. 
После деплоя:
1. Нажмите на `addPerson`, передав туда аргументы, например: `1, "Alice", 25, true`.
2. Затем вызовите `getPerson` с аргументом `1`, чтобы увидеть, что данные успешно сохранились и вернулись.
3. Попробуйте вызвать функции `populateWithFor`, `populateWithWhile` и посмотрите в консоли Remix, какие массивы они возвращают.

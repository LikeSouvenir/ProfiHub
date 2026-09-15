# Модуль 1: Базовый синтаксис — Практическое задание

## Проект: Студенческий реестр (StudentRegistry)

### Цель

Закрепить работу с базовыми типами данных (`uint256`, `string`, `bool`, `address`), составными типами (`struct`, `enum`, `mapping`, динамические массивы), функциями с различными модификаторами видимости (`external`, `public`, `view`, `pure`) и циклами (`for`).

### Техническое задание

Создайте смарт-контракт `StudentRegistry` со следующими требованиями: +

1. **Перечисления и структуры:**

   - Объявите перечисление `Status`: `Enrolled` (Зачислен), `Graduated` (Выпустился), `Expelled` (Отчислен).

   - Объявите структуру `Student`:

     - `uint256 id` — порядковый идентификатор;

     - `string fullName` — ФИО студента;

     - `uint256\[\] scores` — динамический массив полученных оценок;

     - `Status status` — текущий академический статус;

     - `bool exists` — флаг для проверки факта регистрации.

2. **Состояние контракта (State Variables):**

   - `mapping(address =\> Student) private students` — реестр студентов по адресу кошелька;

   - `address\[\] private studentAddresses` — динамический список адресов всех студентов (для обхода реестра);

   - `uint256 public totalStudentsCount` — публичный счетчик зарегистрированных студентов.

3. **Логика функций:**

   - `registerStudent(string memory \_fullName) external`:

     - Регистрирует вызывающий адрес (`msg.sender`).

     - Если адрес уже зарегистрирован — транзакция откатывается с сообщением `"Student already registered"`.

     - Присваивает `id = totalStudentsCount + 1`, статус по умолчанию — `Status.Enrolled`.

     - Добавляет адрес в `studentAddresses`.

   - `addScore(address \_studentAddress, uint256 \_score) external`:

     - Проверяет существование студента по адресу.

     - Проверяет валидность оценки (`\_score \<= 100`).

     - Добавляет оценку в массив `scores` студента в storage.

   - `calculateAverageScore(address \_studentAddress) external view returns (uint256)`:

     - Функция только для чтения (`view`).

     - Если оценок нет, возвращает `0`.

     - Использует цикл `for` для подсчета суммы и возвращает среднее арифметическое (целочисленное деление).

   - `convertGradeToLetter(uint256 \_score) external pure returns (string memory)`:

     - Чистая функция (`pure`), не читающая и не меняющая состояние.

     - 90–100: `"A"`;

     - 75–89: `"B"`;

     - 60–74: `"C"`;

     - Меньше 60: `"F"`.

   - `getStudent(address \_studentAddress) external view returns (...)`:

     - Геттер для получения полной информации о студенте.


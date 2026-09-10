# Модуль 1: Базовый синтаксис — Эталонное решение

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/// @title StudentRegistry - Реестр студентов и их академической успеваемости
/// @notice Демонстрирует работу со структурами, enum, mapping, массивами, циклами и модификаторами функций
contract StudentRegistry {
    // 1. Перечисление статусов студента
    enum Status {
        Enrolled,   // 0: Зачислен
        Graduated,  // 1: Выпустился
        Expelled    // 2: Отчислен
    }

    // 2. Структура студента
    struct Student {
        uint256 id;
        string fullName;
        uint256[] scores;
        Status status;
        bool exists; // Флаг для быстрой O(1) проверки существования
    }

    // 3. Переменные состояния в Storage
    mapping(address => Student) private students;
    address[] private studentAddresses;
    uint256 public totalStudentsCount;

    // События для индексации действий оффчейн
    event StudentRegistered(address indexed studentAddress, uint256 id, string fullName);
    event ScoreAdded(address indexed studentAddress, uint256 score);

    /// @notice Регистрация нового студента
    /// @param _fullName ФИО студента (передается через memory)
    function registerStudent(string memory _fullName) external {
        // Проверка: повторная регистрация запрещена
        require(!students[msg.sender].exists, "Student already registered");
        require(bytes(_fullName).length > 0, "Name cannot be empty");

        totalStudentsCount++;
        studentAddresses.push(msg.sender);

        // Инициализация структуры непосредственно в storage
        Student storage newStudent = students[msg.sender];
        newStudent.id = totalStudentsCount;
        newStudent.fullName = _fullName;
        newStudent.status = Status.Enrolled;
        newStudent.exists = true;

        emit StudentRegistered(msg.sender, totalStudentsCount, _fullName);
    }

    /// @notice Добавление оценки студенту
    /// @param _studentAddress Адрес кошелька студента
    /// @param _score Оценка от 0 до 100
    function addScore(address _studentAddress, uint256 _score) external {
        require(students[_studentAddress].exists, "Student not found");
        require(_score <= 100, "Score cannot exceed 100");

        // Обращаемся по ссылке storage для мутации динамического массива
        students[_studentAddress].scores.push(_score);

        emit ScoreAdded(_studentAddress, _score);
    }

    /// @notice Подсчет среднего балла студента
    /// @dev Использует чтение массива в memory для оптимизации затрат газа при итерации
    /// @param _studentAddress Адрес студента
    /// @return Средний балл (округленный вниз)
    function calculateAverageScore(address _studentAddress) external view returns (uint256) {
        require(students[_studentAddress].exists, "Student not found");

        // Копируем ссылку на массив в memory для более дешевого чтения в цикле
        uint256[] memory scores = students[_studentAddress].scores;
        uint256 length = scores.length;

        if (length == 0) {
            return 0;
        }

        uint256 totalSum = 0;
        // Итерация по массиву оценок
        for (uint256 i = 0; i < length; i++) {
            totalSum += scores[i];
        }

        return totalSum / length;
    }

    /// @notice Конвертация числового балла в буквенный эквивалент
    /// @dev Чистая (pure) функция: не обращается к состоянию блокчейна
    /// @param _score Оценка от 0 до 100
    /// @return Буквенная шкала (A, B, C, F)
    function convertGradeToLetter(uint256 _score) external pure returns (string memory) {
        require(_score <= 100, "Invalid score");

        if (_score >= 90) {
            return "A";
        } else if (_score >= 75) {
            return "B";
        } else if (_score >= 60) {
            return "C";
        } else {
            return "F";
        }
    }

    /// @notice Получение информации о студенте
    function getStudent(address _studentAddress) external view returns (
        uint256 id,
        string memory fullName,
        uint256[] memory scores,
        Status status
    ) {
        require(students[_studentAddress].exists, "Student not found");
        Student storage s = students[_studentAddress];
        return (s.id, s.fullName, s.scores, s.status);
    }

    /// @notice Получение всех зарегистрированных адресов
    function getAllStudentAddresses() external view returns (address[] memory) {
        return studentAddresses;
    }
}
```

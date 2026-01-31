# Ресурсы для изучения алгоритмов

## Практические платформы

### LeetCode
- **URL:** https://leetcode.com
- **Описание:** 2500+ задач, разбиты по сложности (Easy/Medium/Hard)
- **Плюсы:** Самая популярная, реальные вопросы с собеседований
- **Рекомендация:** Blind 75, NeetCode 150

### HackerRank
- **URL:** https://www.hackerrank.com
- **Описание:** Задачи по алгоритмам, структурам данных, SQL
- **Плюсы:** Хорошие туториалы, сертификаты

### CodeWars
- **URL:** https://www.codewars.com
- **Описание:** Gamified platform, поддержка PHP
- **Плюсы:** Много PHP задач, разные уровни сложности

### Exercism
- **URL:** https://exercism.org
- **Описание:** Бесплатные упражнения с менторством
- **Плюсы:** PHP треки, фидбек от менторов

## Списки задач для подготовки

### Blind 75
**Самый популярный список для FAANG**
- 75 задач покрывают основные паттерны
- https://leetcode.com/discuss/general-discussion/460599/blind-75-leetcode-questions

**Категории:**
- Array (9)
- Binary (2)
- Dynamic Programming (11)
- Graph (6)
- Interval (2)
- Linked List (6)
- Matrix (2)
- String (9)
- Tree (11)
- Heap (3)

### NeetCode 150
**Расширенная версия Blind 75**
- 150 задач с видео-разборами
- https://neetcode.io/practice

**Roadmap по неделям:**
1. Arrays & Hashing
2. Two Pointers
3. Sliding Window
4. Stack
5. Binary Search
6. Linked List
7. Trees
8. Heap / Priority Queue
9. Backtracking
10. Graphs
11. Dynamic Programming
12. Greedy

### Grind 75
**Адаптивный план подготовки**
- От 1 до 26 недель
- https://www.techinterviewhandbook.org/grind75

## Книги

### Cracking the Coding Interview (6th Edition)
**Автор:** Gayle Laakmann McDowell
- 189 вопросов
- Решения с объяснениями
- Big O анализ

### Introduction to Algorithms (CLRS)
**Авторы:** Cormen, Leiserson, Rivest, Stein
- Академический подход
- Псевдокод
- Математические доказательства

### Grokking Algorithms
**Автор:** Aditya Bhargava
- Визуальный подход
- Простые объяснения
- Примеры на Python

### Algorithms (4th Edition)
**Автор:** Robert Sedgewick
- Java примеры
- Детальный анализ
- Практическое применение

## YouTube каналы

### NeetCode
- https://www.youtube.com/@NeetCode
- Разборы LeetCode задач
- Паттерны и стратегии

### Back To Back SWE
- https://www.youtube.com/@BackToBackSWE
- Детальные объяснения
- Whiteboard подход

### Errichto
- https://www.youtube.com/@Errichto
- Competitive programming
- Продвинутые алгоритмы

### Abdul Bari
- https://www.youtube.com/@abdul_bari
- Академический стиль
- Algorithms course

## Курсы

### Algorithms Specialization (Stanford)
**Платформа:** Coursera
**Инструктор:** Tim Roughgarden
- 4 курса
- Divide & Conquer
- Graph Search
- Greedy Algorithms
- Dynamic Programming

### Data Structures and Algorithms
**Платформа:** Udemy
- Много практики
- Разные языки

### AlgoExpert
**Платформ:** https://www.algoexpert.io
- 160+ задач
- Видео-объяснения
- Coding workspace

## PHP-специфичные ресурсы

### PHP Algorithms and Data Structures
**GitHub:** https://github.com/TheAlgorithms/PHP
- Реализации на PHP
- Open source

### PHP: The Right Way - Algorithms
**URL:** https://phptherightway.com
- Best practices
- Modern PHP подход

## Инструменты визуализации

### VisuAlgo
- https://visualgo.net
- Анимации алгоритмов
- Интерактивные примеры

### Algorithm Visualizer
- https://algorithm-visualizer.org
- Визуализация кода
- Step-by-step execution

## Cheat Sheets

### Big-O Cheat Sheet
- https://www.bigocheatsheet.com
- Сложность структур данных
- Сложность алгоритмов сортировки

### LeetCode Patterns
- https://seanprashad.com/leetcode-patterns/
- Группировка по паттернам
- Progress tracking

## План подготовки

### Week 1-2: Основы
- Arrays & Hashing
- Two Pointers
- Sliding Window
- Stack & Queue

### Week 3-4: Linked Lists & Trees
- Linked List операции
- Tree traversals (DFS, BFS)
- Binary Search Trees

### Week 5-6: Advanced Trees & Graphs
- Heaps
- Tries
- Graph алгоритмы

### Week 7-8: Dynamic Programming
- 1D DP
- 2D DP
- Memoization

### Week 9-10: Advanced Topics
- Backtracking
- Greedy
- Intervals
- Bit manipulation

### Week 11-12: Mock Interviews
- Timed practice
- Whiteboard practice
- Behavioral questions

## Стратегия решения задач

### 1. Понимание (5 мин)
- Прочитать внимательно
- Примеры входа/выхода
- Edge cases
- Constraints

### 2. Подход (5-10 мин)
- Brute force сначала
- Оптимизация
- Выбор структуры данных
- Time/Space complexity

### 3. Реализация (15-20 мин)
- Писать чистый код
- Комментарии для сложных частей
- Тестирование на примерах

### 4. Оптимизация (5 мин)
- Можно ли лучше?
- Trade-offs
- Обсуждение альтернатив

## Частота тем на собеседованиях

### High Frequency (учи в первую очередь)
1. Arrays & Strings (30%)
2. Trees (15%)
3. Dynamic Programming (15%)
4. Hash Maps (10%)
5. Graphs (10%)

### Medium Frequency
6. Linked Lists (5%)
7. Stacks & Queues (5%)
8. Binary Search (5%)
9. Heap / Priority Queue (3%)
10. Sorting (2%)

### Low Frequency
- Trie
- Bit Manipulation
- Math & Geometry
- Backtracking

## Tips для PHP разработчиков

### Используй встроенные функции
```php
// ✅ Хорошо
$freq = array_count_values($arr);
sort($arr);
$unique = array_unique($arr);

// ❌ Не изобретай велосипед
// (если только не просят реализовать)
```

### SPL классы
```php
SplStack, SplQueue
SplMinHeap, SplMaxHeap
SplDoublyLinkedList
SplFixedArray
```

### Знай сложность PHP функций
```php
in_array()         // O(n)
array_search()     // O(n)
isset($arr[$key])  // O(1)
count()            // O(1) для обычных массивов
array_slice()      // O(n)
array_merge()      // O(n+m)
```

### Практикуй на PHP
- Даже если собеседование на другом языке
- Понимание важнее синтаксиса
- Можно объяснить на псевдокоде

## Мотивация

**Сколько решать задач?**
- Minimum: Blind 75 (2-3 месяца)
- Recommended: NeetCode 150 (3-4 месяца)
- Strong: 300+ задач (6+ месяцев)

**Консистентность важнее количества**
- 1-2 задачи в день = 30-60 в месяц
- Повторение важнее новых задач
- Фокус на паттернах, не на зубрёжке

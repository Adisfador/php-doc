# Big O и сложность алгоритмов

## Big O Notation

### Определение
Оценка **асимптотической сложности** алгоритма — сколько времени/памяти нужно при росте входных данных.

### Основные классы сложности (от лучшего к худшему)

```
O(1)        < O(log n) < O(n)      < O(n log n) < O(n²)     < O(2ⁿ)     < O(n!)
Constant      Log        Linear      Linearithmic  Quadratic   Exponential  Factorial
```

### O(1) - Константная

```php
// Доступ к элементу массива по индексу
function getFirst(array $arr): mixed {
    return $arr[0];  // O(1)
}

// Доступ к элементу HashMap
$map = ['key' => 'value'];
$value = $map['key'];  // O(1)

// Арифметические операции
$sum = $a + $b;  // O(1)
```

**Примеры:** доступ к массиву, hash lookup, математические операции

### O(log n) - Логарифмическая

```php
// Binary Search - бинарный поиск
function binarySearch(array $arr, int $target): int {
    $left = 0;
    $right = count($arr) - 1;
    
    while ($left <= $right) {
        $mid = (int)(($left + $right) / 2);
        
        if ($arr[$mid] === $target) {
            return $mid;
        } elseif ($arr[$mid] < $target) {
            $left = $mid + 1;
        } else {
            $right = $mid - 1;
        }
    }
    
    return -1;
}
// O(log n) - каждую итерацию диапазон делится пополам
```

**Примеры:** binary search, balanced BST операции

### O(n) - Линейная

```php
// Поиск элемента в неотсортированном массиве
function linearSearch(array $arr, int $target): int {
    foreach ($arr as $i => $value) {
        if ($value === $target) {
            return $i;
        }
    }
    return -1;
}
// O(n) - в худшем случае проверим все элементы

// Сумма элементов
function sum(array $arr): int {
    $total = 0;
    foreach ($arr as $num) {
        $total += $num;
    }
    return $total;
}
// O(n)
```

**Примеры:** linear search, array traversal, finding max/min

### O(n log n) - Линейно-логарифмическая

```php
// Merge Sort, Quick Sort (average case)
function mergeSort(array $arr): array {
    if (count($arr) <= 1) return $arr;
    
    $mid = (int)(count($arr) / 2);
    $left = mergeSort(array_slice($arr, 0, $mid));
    $right = mergeSort(array_slice($arr, $mid));
    
    return merge($left, $right);
}
// O(n log n)

// PHP встроенная сортировка
sort($arr);  // O(n log n)
usort($arr, fn($a, $b) => $a <=> $b);  // O(n log n)
```

**Примеры:** эффективные сортировки (merge, quick, heap)

### O(n²) - Квадратичная

```php
// Bubble Sort
function bubbleSort(array $arr): array {
    $n = count($arr);
    for ($i = 0; $i < $n; $i++) {
        for ($j = 0; $j < $n - $i - 1; $j++) {
            if ($arr[$j] > $arr[$j + 1]) {
                [$arr[$j], $arr[$j + 1]] = [$arr[$j + 1], $arr[$j]];
            }
        }
    }
    return $arr;
}
// O(n²) - вложенные циклы

// Поиск пар с заданной суммой (naive)
function findPairs(array $arr, int $target): array {
    $pairs = [];
    for ($i = 0; $i < count($arr); $i++) {
        for ($j = $i + 1; $j < count($arr); $j++) {
            if ($arr[$i] + $arr[$j] === $target) {
                $pairs[] = [$arr[$i], $arr[$j]];
            }
        }
    }
    return $pairs;
}
// O(n²)
```

**Примеры:** вложенные циклы, bubble/selection/insertion sort

### O(2ⁿ) - Экспоненциальная

```php
// Fibonacci (naive recursive)
function fibonacci(int $n): int {
    if ($n <= 1) return $n;
    return fibonacci($n - 1) + fibonacci($n - 2);
}
// O(2ⁿ) - каждый вызов создает 2 подзадачи

// Все подмножества массива
function subsets(array $nums): array {
    $result = [[]];
    foreach ($nums as $num) {
        $newSubsets = [];
        foreach ($result as $subset) {
            $newSubsets[] = array_merge($subset, [$num]);
        }
        $result = array_merge($result, $newSubsets);
    }
    return $result;
}
// O(2ⁿ) - 2^n подмножеств
```

**Примеры:** рекурсивные brute force, генерация всех подмножеств

### O(n!) - Факториальная

```php
// Генерация всех перестановок
function permute(array $nums): array {
    if (count($nums) <= 1) return [$nums];
    
    $result = [];
    foreach ($nums as $i => $num) {
        $rest = array_merge(
            array_slice($nums, 0, $i),
            array_slice($nums, $i + 1)
        );
        foreach (permute($rest) as $perm) {
            $result[] = array_merge([$num], $perm);
        }
    }
    return $result;
}
// O(n!) - n! перестановок
```

**Примеры:** все перестановки, Traveling Salesman (brute force)

## Space Complexity (Пространственная сложность)

### O(1) - Константная память

```php
function swap(array &$arr, int $i, int $j): void {
    $temp = $arr[$i];  // O(1) дополнительной памяти
    $arr[$i] = $arr[$j];
    $arr[$j] = $temp;
}
```

### O(n) - Линейная память

```php
// Копирование массива
function copyArray(array $arr): array {
    return $arr;  // O(n) памяти
}

// Рекурсия с глубиной n
function factorial(int $n): int {
    if ($n <= 1) return 1;
    return $n * factorial($n - 1);
}
// O(n) памяти на call stack
```

## Правила анализа Big O

### 1. Отбрасываем константы

```php
// O(2n) → O(n)
function process(array $arr): void {
    foreach ($arr as $item) {  // O(n)
        echo $item;
    }
    foreach ($arr as $item) {  // O(n)
        echo $item;
    }
}
// O(2n) упрощается до O(n)
```

### 2. Отбрасываем меньшие члены

```php
// O(n² + n) → O(n²)
function process(array $arr): void {
    foreach ($arr as $i) {           // O(n)
        echo $i;
    }
    foreach ($arr as $i) {           // O(n)
        foreach ($arr as $j) {       // O(n)
            echo "$i $j";
        }
    }
}
// O(n + n²) упрощается до O(n²)
```

### 3. Разные входные данные - разные переменные

```php
// O(a + b), НЕ O(n)
function process(array $a, array $b): void {
    foreach ($a as $item) {  // O(a)
        echo $item;
    }
    foreach ($b as $item) {  // O(b)
        echo $item;
    }
}
// O(a + b)

// O(a * b), НЕ O(n²)
function pairs(array $a, array $b): void {
    foreach ($a as $x) {      // O(a)
        foreach ($b as $y) {  // O(b)
            echo "$x $y";
        }
    }
}
// O(a * b)
```

## Сравнение сложностей

### Время выполнения для n = 1,000,000

| Сложность | Операций        | Время (грубо)  |
|-----------|-----------------|----------------|
| O(1)      | 1               | 1 наносек      |
| O(log n)  | ~20             | 20 наносек     |
| O(n)      | 1,000,000       | 1 миллисек     |
| O(n log n)| 20,000,000      | 20 миллисек    |
| O(n²)     | 1,000,000,000,000| 11+ дней      |
| O(2ⁿ)     | 2^1000000       | никогда        |

## Amortized Time (Амортизированная сложность)

### Dynamic Array (ArrayList)

```php
// PHP array с динамическим расширением
$arr = [];
for ($i = 0; $i < 1000; $i++) {
    $arr[] = $i;  // Amortized O(1)
}

// Как это работает:
// - Обычная вставка: O(1)
// - При заполнении: создать массив 2x размера, скопировать все → O(n)
// - Но это происходит редко (при 1, 2, 4, 8, 16, 32...)
// - В среднем: O(1) amortized
```

## Best, Average, Worst Case

### Quick Sort

```php
function quickSort(array $arr): array {
    if (count($arr) <= 1) return $arr;
    
    $pivot = $arr[0];
    $left = $right = [];
    
    for ($i = 1; $i < count($arr); $i++) {
        if ($arr[$i] < $pivot) {
            $left[] = $arr[$i];
        } else {
            $right[] = $arr[$i];
        }
    }
    
    return array_merge(
        quickSort($left),
        [$pivot],
        quickSort($right)
    );
}

// Best case:    O(n log n) - pivot делит массив пополам
// Average case: O(n log n) - обычно pivot близок к медиане
// Worst case:   O(n²)      - pivot всегда min/max (отсортированный массив)
```

## Практические примеры

### Оптимизация: O(n²) → O(n)

```php
// ❌ O(n²) - два прохода
function hasDuplicate(array $arr): bool {
    for ($i = 0; $i < count($arr); $i++) {
        for ($j = $i + 1; $j < count($arr); $j++) {
            if ($arr[$i] === $arr[$j]) {
                return true;
            }
        }
    }
    return false;
}

// ✅ O(n) - используем HashMap
function hasDuplicate(array $arr): bool {
    $seen = [];
    foreach ($arr as $num) {
        if (isset($seen[$num])) {
            return true;
        }
        $seen[$num] = true;
    }
    return false;
}
```

### Trade-off: Time vs Space

```php
// Fibonacci

// ❌ O(2ⁿ) time, O(n) space (recursion depth)
function fib(int $n): int {
    if ($n <= 1) return $n;
    return fib($n - 1) + fib($n - 2);
}

// ✅ O(n) time, O(n) space - memoization
function fib(int $n, array &$memo = []): int {
    if ($n <= 1) return $n;
    if (isset($memo[$n])) return $memo[$n];
    
    $memo[$n] = fib($n - 1, $memo) + fib($n - 2, $memo);
    return $memo[$n];
}

// ✅ O(n) time, O(1) space - iterative
function fib(int $n): int {
    if ($n <= 1) return $n;
    
    $prev = 0;
    $curr = 1;
    
    for ($i = 2; $i <= $n; $i++) {
        $next = $prev + $curr;
        $prev = $curr;
        $curr = $next;
    }
    
    return $curr;
}
```

## Частые вопросы на собесах

**Q: Что такое Big O?**
A: Математическая нотация для описания асимптотической сложности алгоритма — как растет время/память при увеличении входных данных.

**Q: Почему O(2n) = O(n)?**
A: Константы отбрасываются, важен только порядок роста. 2n и n растут линейно.

**Q: Чем O(n) отличается от O(log n)?**
A: O(n) — линейный рост (каждый элемент), O(log n) — логарифмический (делим пополам на каждом шаге).

**Q: Почему HashMap lookup O(1)?**
A: Hash function вычисляет индекс за константное время. В худшем случае (коллизии) — O(n), но amortized O(1).

**Q: Space complexity учитывает входные данные?**
A: Нет, только дополнительную память. Входной массив не считается.

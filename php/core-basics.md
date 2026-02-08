# PHP Core Basics - Основы языка

Фундаментальные концепции PHP для собеседований.

---

## Типы данных

### Скалярные типы

```php
<?php

// Integer (целые числа)
$int = 42;
$hex = 0x1A; // Шестнадцатеричное: 26
$oct = 0144; // Восьмеричное: 100
$bin = 0b1010; // Двоичное: 10
$largeInt = 1_000_000; // PHP 7.4+ - разделители для читаемости

// Float (числа с плавающей точкой)
$float = 3.14;
$scientific = 1.2e3; // 1200
$inf = INF;
$nan = NAN;

// String (строки)
$single = 'Hello';
$double = "World $single"; // Интерполяция переменных
$heredoc = <<<EOT
Многострочный
текст с интерполяцией: $single
EOT;

$nowdoc = <<<'EOT'
Многострочный без интерполяции
EOT;

// Boolean (логические)
$true = true;
$false = false;
```

### Составные типы

```php
// Array (массивы)
$indexed = [1, 2, 3]; // Индексированный
$assoc = ['name' => 'John', 'age' => 30]; // Ассоциативный
$mixed = [1, 'two', 3 => 'three']; // Смешанный

// Object (объекты)
$obj = new stdClass();
$obj->property = 'value';

class User {
    public string $name;
    public function __construct(string $name) {
        $this->name = $name;
    }
}
$user = new User('John');

// Callable (вызываемые)
$func = function() { return 'Hello'; };
$callable = [$obj, 'method'];
$staticCallable = ['ClassName', 'staticMethod'];

// Iterable (итерируемые) - PHP 7.1+
function process(iterable $data) {
    foreach ($data as $item) {
        // Array или Traversable
    }
}
```

### Специальные типы

```php
// NULL
$null = null;

// Resource (ресурсы)
$file = fopen('file.txt', 'r');
$curl = curl_init();

// Mixed (любой тип) - PHP 8.0+
function getValue(): mixed {
    return random_int(0, 1) ? 'string' : 42;
}

// Never (никогда не возвращает) - PHP 8.1+
function terminate(): never {
    exit();
}

// Void (ничего не возвращает) - PHP 7.1+
function log(string $message): void {
    echo $message;
}
```

### Проверка типов

```php
$var = 'hello';

// Функции is_*
is_int($var);      // false
is_float($var);    // false
is_string($var);   // true
is_bool($var);     // false
is_array($var);    // false
is_object($var);   // false
is_null($var);     // false
is_numeric($var);  // false (не число)
is_callable($var); // false
is_iterable($var); // false
is_resource($var); // false

// gettype()
gettype($var); // 'string'

// get_class()
get_class($user); // 'User'

// instanceof
$user instanceof User; // true
```

---

## Переменные

### Объявление и присваивание

```php
// Обычные переменные
$name = 'John';
$age = 30;

// Переменные переменных
$varName = 'age';
$$varName = 40; // Эквивалент $age = 40;
echo $age; // 40

// Константы
define('PI', 3.14159);
const MAX_SIZE = 100; // Только в глобальной области или классе

echo PI; // 3.14159
echo MAX_SIZE; // 100

// Магические константы
echo __LINE__;      // Номер строки
echo __FILE__;      // Полный путь к файлу
echo __DIR__;       // Директория файла
echo __FUNCTION__;  // Имя функции
echo __CLASS__;     // Имя класса
echo __METHOD__;    // Имя метода класса
echo __NAMESPACE__; // Имя namespace
```

### Области видимости

```php
$global = 'Global variable';

function test() {
    // Локальная переменная
    $local = 'Local variable';
    
    // Доступ к глобальной
    global $global;
    echo $global; // 'Global variable'
    
    // Или через $GLOBALS
    echo $GLOBALS['global']; // 'Global variable'
    
    // Статические переменные
    static $counter = 0;
    $counter++;
    echo $counter; // Сохраняет значение между вызовами
}

test(); // 1
test(); // 2
test(); // 3
```

### Передача по значению vs по ссылке

```php
// По значению (копируется)
function byValue($x) {
    $x = 10;
}

$a = 5;
byValue($a);
echo $a; // 5 (не изменилось)

// По ссылке (изменяет оригинал)
function byReference(&$x) {
    $x = 10;
}

$b = 5;
byReference($b);
echo $b; // 10 (изменилось!)

// Возврат по ссылке
$arr = [1, 2, 3];

function &getElement(&$array, $key) {
    return $array[$key];
}

$element = &getElement($arr, 1);
$element = 99;
print_r($arr); // [1, 99, 3]
```

---

## Операторы

### Арифметические

```php
$a = 10;
$b = 3;

$a + $b;  // 13 - сложение
$a - $b;  // 7  - вычитание
$a * $b;  // 30 - умножение
$a / $b;  // 3.333... - деление
$a % $b;  // 1  - остаток от деления
$a ** $b; // 1000 - возведение в степень (PHP 5.6+)

// Унарные
+$a;  // Унарный плюс
-$a;  // Унарный минус
++$a; // Пре-инкремент (сначала +1, потом использует)
$a++; // Пост-инкремент (сначала использует, потом +1)
--$a; // Пре-декремент
$a--; // Пост-декремент

$x = 5;
echo ++$x; // 6 (сначала увеличил)
echo $x++; // 6 (потом увеличил, сейчас показывает старое)
echo $x;   // 7
```

### Операторы присваивания

```php
$a = 10;
$a += 5;  // $a = $a + 5;  // 15
$a -= 3;  // $a = $a - 3;  // 12
$a *= 2;  // $a = $a * 2;  // 24
$a /= 4;  // $a = $a / 4;  // 6
$a %= 4;  // $a = $a % 4;  // 2
$a **= 3; // $a = $a ** 3; // 8

$str = 'Hello';
$str .= ' World'; // 'Hello World' - конкатенация

// Null coalescing (PHP 7.0+)
$name = $_GET['name'] ?? 'Guest'; // Если не задано или null - 'Guest'

// Null coalescing assignment (PHP 7.4+)
$config ??= getDefaultConfig(); // Если $config null - присвоить результат
```

### Операторы сравнения

```php
$a = 5;
$b = '5';

// Сравнение значений (с приведением типов)
$a == $b;   // true  - равно
$a != $b;   // false - не равно
$a <> $b;   // false - не равно (альтернатива)
$a < $b;    // false - меньше
$a > $b;    // false - больше
$a <= $b;   // true  - меньше или равно
$a >= $b;   // true  - больше или равно
$a <=> $b;  // 0     - spaceship operator (PHP 7.0+)
                     // -1 если $a < $b, 0 если равны, 1 если $a > $b

// Строгое сравнение (БЕЗ приведения типов)
$a === $b;  // false - строго равно (значение И тип)
$a !== $b;  // true  - строго не равно

// Примеры
5 == '5';   // true
5 === '5';  // false
0 == false; // true
0 === false; // false
null == false; // true
null === false; // false
```

### Логические операторы

```php
$a = true;
$b = false;

// AND (и)
$a && $b;  // false
$a and $b; // false (меньший приоритет)

// OR (или)
$a || $b;  // true
$a or $b;  // true (меньший приоритет)

// XOR (исключающее или)
$a xor $b; // true (только один true)

// NOT (отрицание)
!$a;  // false

// Short-circuit evaluation
$user && $user->isActive(); // Если $user false/null, второе не выполнится
$data || $data = loadData(); // Загрузит данные только если $data пустое
```

### Строковые операторы

```php
$a = 'Hello';
$b = 'World';

// Конкатенация
$a . ' ' . $b; // 'Hello World'
$a .= ' ' . $b; // $a = 'Hello World'
```

### Операторы массивов

```php
$a = ['a' => 1, 'b' => 2];
$b = ['c' => 3, 'd' => 4];

// Объединение массивов
$a + $b; // ['a' => 1, 'b' => 2, 'c' => 3, 'd' => 4]

// Сравнение массивов
$a == $b;  // false - равны (значения)
$a === $b; // false - идентичны (значения и типы)
$a != $b;  // true  - не равны
$a !== $b; // true  - не идентичны
```

### Тернарный оператор

```php
// Условие ? если_true : если_false
$status = ($age >= 18) ? 'adult' : 'minor';

// Короткий тернарный (PHP 5.3+)
$name = $userName ?: 'Guest'; // Если $userName truthy - использовать его, иначе 'Guest'

// Null coalescing (PHP 7.0+)
$name = $userName ?? 'Guest'; // Если $userName null или не существует - 'Guest'

// Вложенные тернарные (избегать!)
$type = $age < 13 ? 'child' : ($age < 20 ? 'teen' : 'adult');
```

### Оператор распаковки (Spread) - PHP 7.4+

```php
// В массивах
$arr1 = [1, 2, 3];
$arr2 = [...$arr1, 4, 5]; // [1, 2, 3, 4, 5]

$arr3 = ['a' => 1, 'b' => 2];
$arr4 = [...$arr3, 'c' => 3]; // ['a' => 1, 'b' => 2, 'c' => 3]

// В функциях (argument unpacking - PHP 5.6+)
function sum(...$numbers) {
    return array_sum($numbers);
}

sum(1, 2, 3, 4); // 10

$nums = [1, 2, 3];
sum(...$nums); // Распаковка массива в аргументы
```

---

## Управляющие конструкции

### If / Else

```php
$age = 20;

if ($age < 13) {
    echo 'Child';
} elseif ($age < 20) {
    echo 'Teen';
} else {
    echo 'Adult';
}

// Альтернативный синтаксис (для шаблонов)
<?php if ($user): ?>
    <p>Welcome, <?= $user->name ?></p>
<?php else: ?>
    <p>Please log in</p>
<?php endif; ?>
```

### Switch

```php
$role = 'admin';

switch ($role) {
    case 'admin':
        echo 'Full access';
        break;
    case 'editor':
        echo 'Can edit';
        break;
    case 'viewer':
        echo 'Read only';
        break;
    default:
        echo 'No access';
}

// Match expression (PHP 8.0+) - ЛУЧШЕ чем switch!
$message = match ($role) {
    'admin' => 'Full access',
    'editor' => 'Can edit',
    'viewer' => 'Read only',
    default => 'No access',
};

// Match - строгое сравнение (===)
$result = match ($value) {
    0 => 'zero',
    '0' => 'string zero', // Различает 0 и '0'
};

// Switch vs Match
switch (1) {
    case '1': // TRUE - нестрогое сравнение
        break;
}

match (1) {
    '1' => 'match', // НЕ сработает - строгое сравнение
    1 => 'match',   // Сработает
};
```

### Циклы

```php
// While
$i = 0;
while ($i < 5) {
    echo $i;
    $i++;
}

// Do-While (выполнится хотя бы раз)
$i = 0;
do {
    echo $i;
    $i++;
} while ($i < 5);

// For
for ($i = 0; $i < 5; $i++) {
    echo $i;
}

// Foreach
$arr = ['a' => 1, 'b' => 2, 'c' => 3];

// Только значения
foreach ($arr as $value) {
    echo $value;
}

// Ключ и значение
foreach ($arr as $key => $value) {
    echo "$key: $value";
}

// По ссылке (изменяет оригинал)
foreach ($arr as &$value) {
    $value *= 2;
}
unset($value); // ВАЖНО: удалить ссылку после цикла!

// Break и Continue
for ($i = 0; $i < 10; $i++) {
    if ($i == 5) continue; // Пропустить итерацию
    if ($i == 8) break;    // Выйти из цикла
    echo $i;
}

// Break/Continue с уровнем вложенности
for ($i = 0; $i < 3; $i++) {
    for ($j = 0; $j < 3; $j++) {
        if ($j == 1) break 2; // Выйти из обоих циклов
    }
}
```

### Goto (не рекомендуется!)

```php
goto label;
echo 'Skipped';

label:
echo 'Jumped here';
```

---

## Функции

### Объявление и вызов

```php
// Базовая функция
function sayHello() {
    echo 'Hello';
}

sayHello(); // Hello

// С параметрами
function greet($name) {
    echo "Hello, $name";
}

greet('John'); // Hello, John

// С возвратом значения
function add($a, $b) {
    return $a + $b;
}

$result = add(2, 3); // 5

// Значения по умолчанию
function power($base, $exponent = 2) {
    return $base ** $exponent;
}

power(3);    // 9 (3^2)
power(3, 3); // 27 (3^3)

// Типизированные параметры (PHP 7.0+)
function divide(int $a, int $b): float {
    return $a / $b;
}

// Nullable типы (PHP 7.1+)
function find(?int $id): ?User {
    return $id ? User::find($id) : null;
}

// Union типы (PHP 8.0+)
function process(int|string $value): void {
    // $value может быть int ИЛИ string
}

// Mixed тип (PHP 8.0+)
function getValue(): mixed {
    return rand(0, 1) ? 'string' : 42;
}
```

### Именованные аргументы (PHP 8.0+)

```php
function createUser($name, $email, $role = 'user') {
    // ...
}

// Обычный вызов
createUser('John', 'john@example.com', 'admin');

// Именованные аргументы
createUser(
    name: 'John',
    email: 'john@example.com',
    role: 'admin'
);

// Можно пропускать опциональные
createUser(
    name: 'John',
    email: 'john@example.com'
);

// Порядок не важен
createUser(
    email: 'john@example.com',
    name: 'John'
);
```

### Вариативные функции

```php
// Старый способ
function oldSum() {
    $args = func_get_args();
    return array_sum($args);
}

oldSum(1, 2, 3, 4); // 10

// Современный способ (PHP 5.6+)
function sum(...$numbers) {
    return array_sum($numbers);
}

sum(1, 2, 3, 4); // 10

// Типизированные variadic
function sumInts(int ...$numbers): int {
    return array_sum($numbers);
}
```

### Анонимные функции (Closures)

```php
// Базовое использование
$greet = function($name) {
    return "Hello, $name";
};

echo $greet('John'); // Hello, John

// С использованием внешних переменных (use)
$message = 'Hello';

$greet = function($name) use ($message) {
    return "$message, $name";
};

echo $greet('John'); // Hello, John

// По ссылке
$counter = 0;

$increment = function() use (&$counter) {
    $counter++;
};

$increment();
$increment();
echo $counter; // 2

// Arrow functions (PHP 7.4+) - короткий синтаксис
$multiply = fn($x, $y) => $x * $y;
echo $multiply(3, 4); // 12

// Автоматически захватывает переменные из родительской области
$factor = 10;
$multiply = fn($x) => $x * $factor; // use не нужен!
echo $multiply(5); // 50

// В array_map
$numbers = [1, 2, 3, 4];
$squared = array_map(fn($n) => $n ** 2, $numbers);
// [1, 4, 9, 16]
```

### Генераторы (Generators)

```php
// Обычная функция - загружает все в память
function getRange($max) {
    $array = [];
    for ($i = 0; $i < $max; $i++) {
        $array[] = $i;
    }
    return $array;
}

// Генератор - по одному значению
function rangeGenerator($max) {
    for ($i = 0; $i < $max; $i++) {
        yield $i; // Возвращает значение и приостанавливает выполнение
    }
}

// Использование
foreach (rangeGenerator(1000000) as $number) {
    // Использует минимум памяти!
    echo $number;
}

// Генератор с ключами
function getKeyValue() {
    yield 'key1' => 'value1';
    yield 'key2' => 'value2';
}

foreach (getKeyValue() as $key => $value) {
    echo "$key: $value\n";
}

// Делегирование (yield from)
function gen1() {
    yield 1;
    yield 2;
}

function gen2() {
    yield from gen1(); // Делегирование
    yield 3;
}

foreach (gen2() as $value) {
    echo $value; // 1, 2, 3
}
```

---

## Работа со строками

```php
// Конкатенация
$first = 'Hello';
$last = 'World';
$full = $first . ' ' . $last; // 'Hello World'

// Интерполяция
$name = 'John';
echo "Hello, $name"; // Hello, John
echo "Hello, {$name}"; // Фигурные скобки для ясности
echo "User: {$user->name}"; // Для свойств объектов

// Heredoc (с интерполяцией)
$text = <<<EOT
Hello, $name!
This is a multiline string.
EOT;

// Nowdoc (без интерполяции)
$text = <<<'EOT'
Hello, $name!
Variable will not be interpolated.
EOT;

// Полезные функции
strlen('hello');           // 5 - длина
strtolower('HELLO');       // 'hello'
strtoupper('hello');       // 'HELLO'
ucfirst('hello');          // 'Hello'
ucwords('hello world');    // 'Hello World'
trim('  hello  ');         // 'hello'
ltrim('  hello');          // 'hello'
rtrim('hello  ');          // 'hello'

// Поиск и замена
strpos('hello world', 'world');  // 6 - позиция
str_contains('hello', 'ell');    // true (PHP 8.0+)
str_starts_with('hello', 'he');  // true (PHP 8.0+)
str_ends_with('hello', 'lo');    // true (PHP 8.0+)
str_replace('world', 'PHP', 'hello world'); // 'hello PHP'

// Разделение и объединение
explode(' ', 'hello world');     // ['hello', 'world']
implode(', ', ['a', 'b', 'c']);  // 'a, b, c'
str_split('hello', 2);           // ['he', 'll', 'o']

// Подстроки
substr('hello world', 6);        // 'world'
substr('hello world', 0, 5);     // 'hello'
substr('hello world', -5);       // 'world'

// Форматирование
sprintf('Hello, %s! You are %d years old.', 'John', 30);
// 'Hello, John! You are 30 years old.'

number_format(1234567.89, 2, '.', ','); // '1,234,567.89'
```

---

## Работа с массивами

```php
// Создание
$arr = [1, 2, 3];
$assoc = ['name' => 'John', 'age' => 30];

// Добавление элементов
$arr[] = 4;              // [1, 2, 3, 4]
array_push($arr, 5, 6);  // [1, 2, 3, 4, 5, 6]
array_unshift($arr, 0);  // [0, 1, 2, 3, 4, 5, 6]

// Удаление элементов
$last = array_pop($arr);       // Удалить последний
$first = array_shift($arr);    // Удалить первый
unset($arr[2]);                // Удалить по индексу

// Информация о массиве
count($arr);              // Количество элементов
in_array(2, $arr);        // true - есть ли значение
array_key_exists('name', $assoc); // true - есть ли ключ
isset($assoc['name']);    // true - установлен ли ключ

// Слияние и объединение
$arr1 = [1, 2];
$arr2 = [3, 4];
array_merge($arr1, $arr2);  // [1, 2, 3, 4]
$arr1 + $arr2;              // [1, 2] - не дубл. ключи

// Извлечение части
array_slice($arr, 1, 2);    // С позиции 1, 2 элемента
array_splice($arr, 1, 2);   // Удалить и вернуть

// Фильтрация и мапинг
array_filter($arr, fn($x) => $x > 2);  // Фильтр
array_map(fn($x) => $x * 2, $arr);     // Трансформация
array_reduce($arr, fn($carry, $item) => $carry + $item, 0); // Свертка

// Сортировка
sort($arr);        // По значениям (индексы сбросятся)
rsort($arr);       // Обратная
asort($arr);       // По значениям (сохранить ключи)
arsort($arr);      // Обратная с сохранением
ksort($arr);       // По ключам
krsort($arr);      // Обратная по ключам
usort($arr, fn($a, $b) => $a <=> $b); // Пользовательская

// Ключи и значения
array_keys($assoc);    // ['name', 'age']
array_values($assoc);  // ['John', 30]
array_flip($arr);      // Поменять ключи и значения
```

---

## Include / Require

```php
// Include - предупреждение если не найден
include 'file.php';
include_once 'file.php'; // Только один раз

// Require - фатальная ошибка если не найден
require 'file.php';
require_once 'file.php'; // Только один раз (для классов/функций)

// Возврат значения из файла
// config.php
return [
    'db' => 'mysql',
    'host' => 'localhost',
];

// index.php
$config = require 'config.php';
echo $config['db']; // 'mysql'
```

**Правило:** Используй `require_once` для классов/библиотек, `include` для шаблонов.

---

## Исключения и ошибки

### Иерархия Throwable (PHP 7.0+)

```php
// Throwable - корневой интерфейс (нельзя реализовать самостоятельно)
interface Throwable {
    public function getMessage(): string;
    public function getCode(): int;
    public function getFile(): string;
    public function getLine(): int;
    public function getTrace(): array;
    public function getTraceAsString(): string;
    public function getPrevious(): ?Throwable;
}

// Иерархия:
Throwable
├── Exception           // Пользовательские исключения
│   ├── ErrorException  // Обёртка для PHP warnings/notices
│   └── [SPL Exceptions] // см. spl.md
└── Error              // Ошибки движка PHP
    ├── TypeError
    ├── ParseError
    ├── ArithmeticError
    │   └── DivisionByZeroError
    ├── CompileError
    ├── ArgumentCountError
    ├── ValueError
    ├── UnhandledMatchError
    └── AssertionError
```

### Error - ошибки движка

```php
// TypeError - неверный тип аргумента/возврата
function add(int $a, int $b): int {
    return $a + $b;
}

add(5, "10");  // TypeError: Argument #2 must be of type int, string given

// ValueError - корректный тип, но неверное значение (PHP 8.0+)
function setAge(int $age): void {
    if ($age < 0 || $age > 150) {
        throw new ValueError('Age must be between 0 and 150');
    }
}

setAge(-5);  // ValueError

// ParseError - синтаксическая ошибка в eval/require
eval('invalid php code');  // ParseError

// ArithmeticError - математическая ошибка
intdiv(10, 0);  // DivisionByZeroError (наследует ArithmeticError)

// ArgumentCountError - неверное количество аргументов
function test($a, $b) {}
test(1);  // ArgumentCountError: Too few arguments

// UnhandledMatchError - нет совпадений в match без default (PHP 8.0+)
$value = 5;
match ($value) {
    1 => 'one',
    2 => 'two',
};  // UnhandledMatchError

// CompileError - ошибка компиляции (редко в runtime)
```

### Try / Catch / Finally

```php
// Базовое использование
try {
    $result = divide(10, 0);
} catch (DivisionByZeroError $e) {
    echo "Cannot divide by zero: " . $e->getMessage();
}

// Множественные catch
try {
    riskyOperation();
} catch (TypeError $e) {
    echo "Type error: " . $e->getMessage();
} catch (ValueError $e) {
    echo "Value error: " . $e->getMessage();
} catch (Exception $e) {
    echo "General error: " . $e->getMessage();
}

// Catch нескольких типов (PHP 7.1+)
try {
    operation();
} catch (TypeError | ValueError $e) {
    echo "Type or value error: " . $e->getMessage();
}

// Finally - выполняется всегда
try {
    $file = fopen('data.txt', 'r');
    processFile($file);
} catch (Exception $e) {
    echo "Error: " . $e->getMessage();
} finally {
    // Выполнится в любом случае
    if (isset($file)) {
        fclose($file);
    }
}

// Поймать всё (PHP 7.0+)
try {
    anything();
} catch (Throwable $t) {
    // Поймает и Exception, и Error
    echo $t->getMessage();
}
```

### Создание и выброс исключений

```php
// Бросить исключение
throw new Exception('Something went wrong');

// С кодом ошибки
throw new Exception('Database error', 500);

// С предыдущим исключением (chain)
try {
    connectToDatabase();
} catch (Exception $e) {
    throw new Exception('Failed to connect', 0, $e);
}

// Получить цепочку
catch (Exception $e) {
    echo $e->getMessage();           // Текущее сообщение
    echo $e->getPrevious()->getMessage(); // Предыдущее
}
```

### Пользовательские исключения

```php
// Наследуем от Exception
class ValidationException extends Exception {
    private array $errors;
    
    public function __construct(array $errors, $code = 0, Throwable $previous = null) {
        $this->errors = $errors;
        parent::__construct('Validation failed', $code, $previous);
    }
    
    public function getErrors(): array {
        return $this->errors;
    }
}

// Использование
try {
    if (empty($email)) {
        throw new ValidationException(['email' => 'Required']);
    }
} catch (ValidationException $e) {
    print_r($e->getErrors());
}

// Иерархия для разных типов ошибок
class AppException extends Exception {}
class DatabaseException extends AppException {}
class NetworkException extends AppException {}

// Ловим группу
try {
    operation();
} catch (AppException $e) {
    // Поймает все типы AppException
}
```

### SPL Exceptions

PHP предоставляет специализированные SPL-исключения для типичных ситуаций. **Подробно описаны в [spl.md](spl.md)**.

Краткий список:

**LogicException** - ошибки логики программы (должны быть исправлены в коде):
- `InvalidArgumentException` - неверный аргумент (самое частое!)
- `DomainException` - значение вне допустимой области
- `LengthException` - неверная длина
- `OutOfRangeException` - индекс вне диапазона
- `BadFunctionCallException` / `BadMethodCallException`

**RuntimeException** - ошибки времени выполнения:
- `UnexpectedValueException` - неожиданное значение
- `OutOfBoundsException` - индекс вне границ
- `OverflowException` / `UnderflowException`
- `RangeException` - арифметическое значение вне диапазона

```php
// Примеры использования
function divide(int $a, int $b): float {
    if ($b === 0) {
        throw new InvalidArgumentException('Division by zero');
    }
    return $a / $b;
}

class Email {
    public function __construct(private string $email) {
        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
            throw new DomainException('Invalid email format');
        }
    }
}
```

### Best Practices

```php
// ✅ Используй специфичные исключения
throw new InvalidArgumentException('ID required');

// ❌ Не используй общий Exception
throw new Exception('ID required');

// ✅ Ловить конкретные типы
try {
    operation();
} catch (InvalidArgumentException $e) {
    // Конкретная обработка
} catch (Exception $e) {
    // Общая обработка
}

// ❌ Не ловить всё подряд
try {
    operation();
} catch (Exception $e) {
    // Потеряна информация о типе ошибки
}

// ✅ Использовать finally для cleanup
try {
    $lock = acquireLock();
} finally {
    releaseLock($lock);  // Выполнится всегда
}

// ✅ Не глушить исключения
try {
    operation();
} catch (Exception $e) {
    // Хотя бы залогировать!
    error_log($e->getMessage());
}

// ❌ Пустой catch - потеря информации
try {
    operation();
} catch (Exception $e) {
    // Молчит - плохая практика
}
```

---

### Error Handlers (Обработчики ошибок)

PHP позволяет перехватывать и обрабатывать ошибки, исключения и завершение скрипта.

#### set_error_handler() - обработчик ошибок

**Для чего:** Перехват ошибок (warnings, notices, deprecated) и их кастомная обработка.

```php
// Установка обработчика
set_error_handler(function($errno, $errstr, $errfile, $errline) {
    $errorTypes = [
        E_ERROR => 'ERROR',
        E_WARNING => 'WARNING',
        E_NOTICE => 'NOTICE',
        E_DEPRECATED => 'DEPRECATED',
        E_STRICT => 'STRICT',
        E_USER_ERROR => 'USER_ERROR',
        E_USER_WARNING => 'USER_WARNING',
        E_USER_NOTICE => 'USER_NOTICE',
    ];
    
    $type = $errorTypes[$errno] ?? 'UNKNOWN';
    
    error_log("[{$type}] {$errstr} in {$errfile} on line {$errline}");
    
    // true = ошибка обработана, false = передать в стандартный обработчик
    return true;
});

// Теперь все ошибки логируются
echo $undefinedVariable;  // Вызовет наш обработчик вместо стандартного warning

// Восстановить стандартный обработчик
restore_error_handler();
```

**Уровни ошибок:**

```php
// E_ERROR, E_PARSE, E_CORE_ERROR - НЕ перехватываются set_error_handler
// Используй set_exception_handler или register_shutdown_function

// E_WARNING - предупреждение (скрипт продолжает работу)
fopen('nonexistent.txt', 'r');  // E_WARNING

// E_NOTICE - уведомление (потенциальная проблема)
echo $undefined;  // E_NOTICE

// E_DEPRECATED - устаревшая фича
// E_STRICT - рекомендации по совместимости (PHP 5.x)

// E_USER_* - пользовательские ошибки
trigger_error('Custom warning', E_USER_WARNING);
trigger_error('Custom notice', E_USER_NOTICE);
trigger_error('Custom error', E_USER_ERROR);  // Останавливает скрипт
```

**Конвертация ошибок в исключения:**

```php
set_error_handler(function($errno, $errstr, $errfile, $errline) {
    // Пропустить, если ошибка подавлена через @
    if (!(error_reporting() & $errno)) {
        return false;
    }
    
    // Бросить исключение вместо warning/notice
    throw new ErrorException($errstr, 0, $errno, $errfile, $errline);
});

try {
    // Теперь warning станет исключением
    $file = fopen('nonexistent.txt', 'r');
} catch (ErrorException $e) {
    echo "Caught: " . $e->getMessage();
}
```

**Фильтрация по типу ошибки:**

```php
set_error_handler(function($errno, $errstr, $errfile, $errline) {
    // Логировать только warnings
    if ($errno === E_WARNING) {
        error_log("WARNING: $errstr");
        return true;
    }
    
    // Остальные ошибки - стандартному обработчику
    return false;
});

// Или через второй параметр (битовая маска)
set_error_handler($handler, E_WARNING | E_NOTICE);  // Только warnings и notices
```

#### set_exception_handler() - обработчик исключений

**Для чего:** Перехват **неперехваченных** исключений (последняя линия защиты).

```php
set_exception_handler(function(Throwable $exception) {
    error_log("Uncaught exception: " . $exception->getMessage());
    error_log($exception->getTraceAsString());
    
    // В production - показать красивую страницу ошибки
    if (!defined('DEBUG') || !DEBUG) {
        http_response_code(500);
        echo "Something went wrong. Please try again later.";
    } else {
        // В development - подробная информация
        echo "<h1>Exception</h1>";
        echo "<pre>" . $exception . "</pre>";
    }
    
    exit(1);  // Завершить скрипт
});

// Бросаем исключение без try/catch
throw new Exception('This will be caught by exception handler');
```

**Логирование в файл/Sentry:**

```php
set_exception_handler(function(Throwable $e) {
    // Логирование в файл
    $log = sprintf(
        "[%s] %s: %s in %s:%d\nStack trace:\n%s\n\n",
        date('Y-m-d H:i:s'),
        get_class($e),
        $e->getMessage(),
        $e->getFile(),
        $e->getLine(),
        $e->getTraceAsString()
    );
    
    file_put_contents('/var/log/php_errors.log', $log, FILE_APPEND);
    
    // Или отправить в Sentry/Bugsnag
    // Sentry\captureException($e);
    
    // Показать пользователю
    http_response_code(500);
    include 'error-500.html';
    exit;
});
```

#### register_shutdown_function() - cleanup после завершения

**Для чего:** Код, который выполнится **ВСЕГДА** в конце скрипта (даже при fatal error).

```php
register_shutdown_function(function() {
    echo "This runs even on fatal error\n";
    
    // Проверить, была ли fatal error
    $error = error_get_last();
    
    if ($error !== null && in_array($error['type'], [E_ERROR, E_PARSE, E_CORE_ERROR, E_COMPILE_ERROR])) {
        // Fatal error произошла!
        error_log("Fatal error: {$error['message']} in {$error['file']}:{$error['line']}");
        
        // Показать страницу ошибки
        http_response_code(500);
        echo "Fatal error occurred. Please contact support.";
    }
});

// Cleanup ресурсов
register_shutdown_function(function() {
    // Закрыть соединения
    if (isset($db)) {
        $db->close();
    }
    
    // Удалить временные файлы
    @unlink('/tmp/lockfile');
    
    // Логирование времени выполнения
    $executionTime = microtime(true) - $_SERVER['REQUEST_TIME_FLOAT'];
    error_log("Execution time: {$executionTime}s");
});
```

**Перехват fatal errors:**

```php
// Fatal errors (E_ERROR) НЕ перехватываются set_error_handler
// Но можно поймать через shutdown function

register_shutdown_function(function() {
    $error = error_get_last();
    
    if ($error !== null && $error['type'] === E_ERROR) {
        // Fatal error!
        $message = $error['message'];
        $file = $error['file'];
        $line = $error['line'];
        
        // Логировать
        error_log("FATAL: $message in $file:$line");
        
        // Очистить output buffer (может содержать частичный вывод)
        while (ob_get_level()) {
            ob_end_clean();
        }
        
        // Показать страницу ошибки
        http_response_code(500);
        include 'error-fatal.html';
    }
});

// Этот код вызовет fatal error
undefinedFunction();  // Будет перехвачен в shutdown function
```

#### Комплексная система обработки ошибок

```php
class ErrorHandler
{
    public static function register(): void
    {
        // 1. Конвертация ошибок в исключения
        set_error_handler([self::class, 'handleError']);
        
        // 2. Обработка неперехваченных исключений
        set_exception_handler([self::class, 'handleException']);
        
        // 3. Fatal errors + cleanup
        register_shutdown_function([self::class, 'handleShutdown']);
    }
    
    public static function handleError(
        int $level,
        string $message,
        string $file = '',
        int $line = 0
    ): bool {
        // Пропустить подавленные ошибки (@)
        if (!(error_reporting() & $level)) {
            return false;
        }
        
        // Конвертировать в исключение
        throw new ErrorException($message, 0, $level, $file, $line);
    }
    
    public static function handleException(Throwable $e): void
    {
        self::logException($e);
        self::renderException($e);
        exit(1);
    }
    
    public static function handleShutdown(): void
    {
        $error = error_get_last();
        
        if ($error !== null && in_array($error['type'], [E_ERROR, E_PARSE, E_CORE_ERROR])) {
            // Создаем исключение из fatal error
            $exception = new ErrorException(
                $error['message'],
                0,
                $error['type'],
                $error['file'],
                $error['line']
            );
            
            self::logException($exception);
            self::renderException($exception);
        }
    }
    
    private static function logException(Throwable $e): void
    {
        $log = sprintf(
            "[%s] %s: %s in %s:%d\n%s\n",
            date('Y-m-d H:i:s'),
            get_class($e),
            $e->getMessage(),
            $e->getFile(),
            $e->getLine(),
            $e->getTraceAsString()
        );
        
        error_log($log);
    }
    
    private static function renderException(Throwable $e): void
    {
        // Очистить output buffer
        while (ob_get_level()) {
            ob_end_clean();
        }
        
        http_response_code(500);
        
        if (getenv('APP_ENV') === 'production') {
            // Production: общее сообщение
            include __DIR__ . '/views/error-500.php';
        } else {
            // Development: подробности
            echo "<h1>" . get_class($e) . "</h1>";
            echo "<p><strong>Message:</strong> {$e->getMessage()}</p>";
            echo "<p><strong>File:</strong> {$e->getFile()}:{$e->getLine()}</p>";
            echo "<pre>{$e->getTraceAsString()}</pre>";
        }
    }
}

// Регистрация в начале приложения
ErrorHandler::register();
```

#### Laravel Error Handling

Laravel уже имеет мощную систему обработки ошибок:

```php
// app/Exceptions/Handler.php

class Handler extends ExceptionHandler
{
    // Исключения, которые НЕ логируются
    protected $dontReport = [
        \Illuminate\Auth\AuthenticationException::class,
        \Illuminate\Validation\ValidationException::class,
    ];
    
    // Кастомная логика логирования
    public function report(Throwable $exception)
    {
        if ($exception instanceof CustomException) {
            // Отправить в Sentry
            if (app()->bound('sentry')) {
                app('sentry')->captureException($exception);
            }
        }
        
        parent::report($exception);
    }
    
    // Кастомный рендеринг
    public function render($request, Throwable $exception)
    {
        // API запросы → JSON
        if ($request->expectsJson()) {
            return response()->json([
                'error' => $exception->getMessage(),
                'code' => $exception->getCode(),
            ], 500);
        }
        
        // Кастомная страница для конкретного исключения
        if ($exception instanceof NotFoundException) {
            return response()->view('errors.not-found', [], 404);
        }
        
        return parent::render($request, $exception);
    }
}
```

#### Best Practices

```php
// ✅ Регистрируй обработчики в начале приложения
// index.php или bootstrap файл
ErrorHandler::register();

// ✅ Логируй ВСЕ ошибки в production
set_error_handler(function($errno, $errstr, $errfile, $errline) {
    error_log("[ERROR] $errstr in $errfile:$errline");
    return true;
});

// ✅ Показывай детали только в development
if (getenv('APP_ENV') !== 'production') {
    ini_set('display_errors', '1');
    error_reporting(E_ALL);
} else {
    ini_set('display_errors', '0');
    error_reporting(E_ALL & ~E_DEPRECATED & ~E_STRICT);
}

// ✅ Используй shutdown function для cleanup
register_shutdown_function(function() {
    // Закрыть соединения, удалить lock файлы
});

// ✅ Конвертируй ошибки в исключения для единообразия
set_error_handler(function($errno, $errstr, $errfile, $errline) {
    throw new ErrorException($errstr, 0, $errno, $errfile, $errline);
});

// ❌ НЕ подавляй ошибки через @
@fopen('file.txt', 'r');  // Плохая практика

// ✅ Обрабатывай явно
try {
    $file = fopen('file.txt', 'r');
    if ($file === false) {
        throw new RuntimeException('Cannot open file');
    }
} catch (Throwable $e) {
    // Обработка
}

// ✅ Всегда очищай output buffer при ошибке
register_shutdown_function(function() {
    if (error_get_last()) {
        while (ob_get_level()) {
            ob_end_clean();
        }
    }
});
```

---

## 🎓 Для собеседования: ключевые точки

1. **Scalar types (PHP 7.0+)** - int, float, string, bool, mixed (PHP 8.0+)
2. **Type juggling vs strict_types** - declare(strict_types=1) для запрета coercion
3. **Array** - индексированный/ассоциативный, array_* функции
4. **Spread operator (...)** - array unpacking, функции с variadic параметрами
5. **Spaceship operator (<=>)** - трехстороннее сравнение (-1, 0, 1)
6. **Null coalescing (??, ??=)** - $var ?? 'default' вместо isset($var) ? $var : 'default'
7. **String heredoc/nowdoc** - многострочные строки, nowdoc без интерполяции
8. **Static variables** - сохраняются между вызовами функции
9. **Variable scope** - global, local, static, superglobals ($_GET, $_POST)
10. **Closures (anonymous functions)** - fn() => (arrow functions PHP 7.4+), use keyword
11. **Match expression (PHP 8.0+)** - строгое сравнение (===), возвращает значение
12. **Error handling** - try/catch/finally, специфичные exception types, Throwable (Exception + Error)
13. **Error Handlers** - set_error_handler (преобразование ошибок в исключения), set_exception_handler (глобальный catch), register_shutdown_function (fatal errors + cleanup)

**Главное:** Используй strict_types, типизируй всё, знай новые фичи PHP 8+ (особенно match, named arguments, attributes), всегда регистрируй error handlers для production.

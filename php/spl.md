# SPL - Standard PHP Library

Детальный разбор SPL: итераторы, структуры данных, исключения, файлы, функции для массивов.

---

## 🎯 Что такое SPL?

**SPL (Standard PHP Library)** - встроенная библиотека PHP с:
- **Итераторы** - обход коллекций
- **Структуры данных** - стеки, очереди, кучи
- **Исключения** - специализированные exceptions
- **Файлы** - работа с файловой системой
- **Функции** - для массивов и autoloading

**Преимущества SPL:**
- ✅ Встроена в PHP (не нужно устанавливать)
- ✅ Оптимизирована на уровне C
- ✅ Объектно-ориентированный подход

---

## 🔄 Итераторы (Iterators)

### Iterator Interface

**Базовый интерфейс для обхода:**
```php
interface Iterator extends Traversable {
    public function current(): mixed;  // текущий элемент
    public function key(): mixed;      // текущий ключ
    public function next(): void;      // перейти к следующему
    public function rewind(): void;    // вернуться к началу
    public function valid(): bool;     // есть ли текущий элемент
}

// Пример реализации:
class MyIterator implements Iterator {
    private array $items;
    private int $position = 0;
    
    public function __construct(array $items) {
        $this->items = $items;
    }
    
    public function current(): mixed {
        return $this->items[$this->position];
    }
    
    public function key(): mixed {
        return $this->position;
    }
    
    public function next(): void {
        $this->position++;
    }
    
    public function rewind(): void {
        $this->position = 0;
    }
    
    public function valid(): bool {
        return isset($this->items[$this->position]);
    }
}

// Использование:
$iterator = new MyIterator([1, 2, 3, 4, 5]);

foreach ($iterator as $key => $value) {
    echo "$key => $value\n";
}

// Вывод:
// 0 => 1
// 1 => 2
// 2 => 3
// 3 => 4
// 4 => 5
```

### ArrayIterator - итератор для массивов

```php
$array = ['a' => 1, 'b' => 2, 'c' => 3];
$iterator = new ArrayIterator($array);

foreach ($iterator as $key => $value) {
    echo "$key => $value\n";
}

// Методы ArrayIterator:
$iterator->append(4);        // добавить элемент
$iterator->count();          // количество элементов (implements Countable)
$iterator->seek(2);          // перейти к позиции 2
$iterator->current();        // текущий элемент
$iterator->asort();          // сортировка по значениям
$iterator->ksort();          // сортировка по ключам
$iterator->getArrayCopy();   // получить массив

// Флаги поведения:
$iterator = new ArrayIterator($array, ArrayIterator::ARRAY_AS_PROPS);
echo $iterator->a;  // 1 (доступ как к свойствам)
```

### IteratorAggregate - делегирование итерации

```php
// Упрощенная реализация (делегирует итерацию)
class MyCollection implements IteratorAggregate {
    private array $items = [];
    
    public function add($item): void {
        $this->items[] = $item;
    }
    
    // Возвращаем итератор вместо реализации всех методов Iterator
    public function getIterator(): Traversable {
        return new ArrayIterator($this->items);
    }
}

$collection = new MyCollection();
$collection->add('item1');
$collection->add('item2');

foreach ($collection as $item) {
    echo $item . "\n";
}

// Laravel Collection использует IteratorAggregate!
```

### FilterIterator - фильтрация элементов

```php
class EvenFilterIterator extends FilterIterator {
    public function accept(): bool {
        return $this->current() % 2 === 0;  // только четные
    }
}

$numbers = new ArrayIterator([1, 2, 3, 4, 5, 6, 7, 8, 9, 10]);
$even = new EvenFilterIterator($numbers);

foreach ($even as $num) {
    echo $num . " ";  // 2 4 6 8 10
}

// Или через CallbackFilterIterator (PHP 5.4+):
$even = new CallbackFilterIterator($numbers, function($current) {
    return $current % 2 === 0;
});
```

### LimitIterator - ограничение элементов

```php
$numbers = new ArrayIterator([1, 2, 3, 4, 5, 6, 7, 8, 9, 10]);

// Пропустить 2 элемента, взять 3
$limited = new LimitIterator($numbers, 2, 3);  // offset=2, count=3

foreach ($limited as $num) {
    echo $num . " ";  // 3 4 5
}

// Полезно для пагинации:
$page = 2;
$perPage = 10;
$offset = ($page - 1) * $perPage;
$paginated = new LimitIterator($iterator, $offset, $perPage);
```

### RecursiveIteratorIterator - обход вложенных структур

```php
$array = [
    'a' => 1,
    'b' => [
        'c' => 2,
        'd' => [
            'e' => 3
        ]
    ],
    'f' => 4
];

$iterator = new RecursiveIteratorIterator(
    new RecursiveArrayIterator($array),
    RecursiveIteratorIterator::SELF_FIRST
);

foreach ($iterator as $key => $value) {
    $depth = $iterator->getDepth();
    $indent = str_repeat('  ', $depth);
    echo "{$indent}{$key} => {$value}\n";
}

// Вывод:
// a => 1
//   b => Array
//     c => 2
//       d => Array
//         e => 3
// f => 4

// Режимы:
// SELF_FIRST - сначала родитель, потом дети
// CHILD_FIRST - сначала дети, потом родитель
// LEAVES_ONLY - только листья (без контейнеров)
```

### RecursiveDirectoryIterator - обход файловой системы

```php
// Рекурсивный обход всех файлов в директории
$directory = new RecursiveDirectoryIterator('/path/to/dir');
$iterator = new RecursiveIteratorIterator($directory);

foreach ($iterator as $file) {
    if ($file->isFile()) {
        echo $file->getPathname() . "\n";
    }
}

// Найти все PHP файлы:
$directory = new RecursiveDirectoryIterator(__DIR__);
$iterator = new RecursiveIteratorIterator($directory);
$regex = new RegexIterator($iterator, '/^.+\.php$/i', RecursiveRegexIterator::GET_MATCH);

foreach ($regex as $file) {
    echo $file[0] . "\n";
}
```

### AppendIterator - объединение итераторов

```php
$iterator1 = new ArrayIterator([1, 2, 3]);
$iterator2 = new ArrayIterator([4, 5, 6]);
$iterator3 = new ArrayIterator([7, 8, 9]);

$append = new AppendIterator();
$append->append($iterator1);
$append->append($iterator2);
$append->append($iterator3);

foreach ($append as $value) {
    echo $value . " ";  // 1 2 3 4 5 6 7 8 9
}
```

### CachingIterator - кэширование и lookahead

```php
$array = new ArrayIterator(['a', 'b', 'c']);
$cache = new CachingIterator($array, CachingIterator::FULL_CACHE);

foreach ($cache as $item) {
    echo $item . " ";
}

// Получить все элементы из кэша:
$cached = $cache->getCache();  // ['a', 'b', 'c']

// Lookahead (проверить есть ли следующий элемент):
$array = new ArrayIterator([1, 2, 3]);
$cache = new CachingIterator($array);

foreach ($cache as $current) {
    echo $current;
    if ($cache->hasNext()) {
        echo ", ";  // запятая только если есть следующий
    }
}
// Вывод: 1, 2, 3
```

---

## 📦 Структуры данных (Data Structures)

### SplStack - стек (LIFO)

```php
$stack = new SplStack();

// Push элементы
$stack->push('first');
$stack->push('second');
$stack->push('third');

// Pop элементы (LIFO - Last In First Out)
echo $stack->pop() . "\n";  // 'third'
echo $stack->pop() . "\n";  // 'second'
echo $stack->pop() . "\n";  // 'first'

// Методы:
$stack->top();     // верхний элемент (без удаления)
$stack->isEmpty(); // пустой ли стек
$stack->count();   // количество элементов

// Пример: undo операции
class UndoManager {
    private SplStack $undoStack;
    
    public function __construct() {
        $this->undoStack = new SplStack();
    }
    
    public function execute(callable $action): void {
        $action();
        $this->undoStack->push($action);
    }
    
    public function undo(): void {
        if (!$this->undoStack->isEmpty()) {
            $action = $this->undoStack->pop();
            // отменить действие
        }
    }
}
```

### SplQueue - очередь (FIFO)

```php
$queue = new SplQueue();

// Enqueue элементы
$queue->enqueue('first');
$queue->enqueue('second');
$queue->enqueue('third');

// Dequeue элементы (FIFO - First In First Out)
echo $queue->dequeue() . "\n";  // 'first'
echo $queue->dequeue() . "\n";  // 'second'
echo $queue->dequeue() . "\n";  // 'third'

// Пример: обработка задач
class TaskQueue {
    private SplQueue $queue;
    
    public function __construct() {
        $this->queue = new SplQueue();
    }
    
    public function addTask(callable $task): void {
        $this->queue->enqueue($task);
    }
    
    public function processNext(): void {
        if (!$this->queue->isEmpty()) {
            $task = $this->queue->dequeue();
            $task();
        }
    }
    
    public function processAll(): void {
        while (!$this->queue->isEmpty()) {
            $this->processNext();
        }
    }
}
```

### SplPriorityQueue - очередь с приоритетом

```php
$pq = new SplPriorityQueue();

// Вставка с приоритетом (чем больше приоритет, тем раньше извлекается)
$pq->insert('low priority', 1);
$pq->insert('high priority', 10);
$pq->insert('medium priority', 5);

// Извлечение в порядке приоритета
echo $pq->extract() . "\n";  // 'high priority' (10)
echo $pq->extract() . "\n";  // 'medium priority' (5)
echo $pq->extract() . "\n";  // 'low priority' (1)

// Режим сравнения приоритетов:
$pq->setExtractFlags(SplPriorityQueue::EXTR_BOTH);  // [data, priority]
$pq->setExtractFlags(SplPriorityQueue::EXTR_DATA);  // только data (по умолчанию)
$pq->setExtractFlags(SplPriorityQueue::EXTR_PRIORITY);  // только priority

// Пример: задачи с приоритетом
class PriorityTaskQueue {
    private SplPriorityQueue $queue;
    
    public function __construct() {
        $this->queue = new SplPriorityQueue();
    }
    
    public function addTask(callable $task, int $priority): void {
        $this->queue->insert($task, $priority);
    }
    
    public function processNext(): void {
        if (!$this->queue->isEmpty()) {
            $task = $this->queue->extract();
            $task();
        }
    }
}

$queue = new PriorityTaskQueue();
$queue->addTask(fn() => echo "Low\n", 1);
$queue->addTask(fn() => echo "Critical\n", 10);
$queue->addTask(fn() => echo "Medium\n", 5);

$queue->processNext();  // "Critical"
$queue->processNext();  // "Medium"
$queue->processNext();  // "Low"
```

### SplHeap - куча (бинарная куча)

```php
// Min Heap (минимальный элемент вверху)
class MinHeap extends SplMinHeap {
    public function compare($a, $b): int {
        return $b <=> $a;  // обратное сравнение для min heap
    }
}

$heap = new SplMinHeap();
$heap->insert(10);
$heap->insert(5);
$heap->insert(20);
$heap->insert(3);

echo $heap->extract() . "\n";  // 3 (минимум)
echo $heap->extract() . "\n";  // 5
echo $heap->extract() . "\n";  // 10
echo $heap->extract() . "\n";  // 20

// Max Heap (максимальный элемент вверху)
$heap = new SplMaxHeap();
$heap->insert(10);
$heap->insert(5);
$heap->insert(20);
$heap->insert(3);

echo $heap->extract() . "\n";  // 20 (максимум)
echo $heap->extract() . "\n";  // 10
echo $heap->extract() . "\n";  // 5
echo $heap->extract() . "\n";  // 3

// Custom Heap
class TaskHeap extends SplHeap {
    protected function compare($a, $b): int {
        // Сравнение по приоритету
        return $a['priority'] <=> $b['priority'];
    }
}

$heap = new TaskHeap();
$heap->insert(['name' => 'low', 'priority' => 1]);
$heap->insert(['name' => 'high', 'priority' => 10]);
$heap->insert(['name' => 'medium', 'priority' => 5]);

$task = $heap->extract();  // ['name' => 'high', 'priority' => 10]
```

### SplFixedArray - массив фиксированного размера

```php
// Обычный array растет динамически (медленнее, больше памяти)
$array = [];
$array[1000000] = 'value';  // память для 1M элементов

// SplFixedArray - фиксированный размер (быстрее, меньше памяти)
$fixed = new SplFixedArray(5);
$fixed[0] = 'a';
$fixed[1] = 'b';
$fixed[2] = 'c';

// Нельзя добавить больше элементов:
$fixed[5] = 'd';  // RuntimeException: Index invalid or out of range

// Изменить размер:
$fixed->setSize(10);  // теперь можно добавить еще 5 элементов

// Конвертация:
$array = [1, 2, 3, 4, 5];
$fixed = SplFixedArray::fromArray($array);
$array = $fixed->toArray();

// Производительность:
// SplFixedArray на ~10-20% быстрее и использует меньше памяти
// НО только для больших массивов (> 100k элементов)
```

### SplDoublyLinkedList - двусвязный список

```php
$list = new SplDoublyLinkedList();

// Добавление
$list->push('a');       // добавить в конец
$list->push('b');
$list->unshift('z');    // добавить в начало

// z, a, b

// Удаление
$list->pop();           // удалить с конца ('b')
$list->shift();         // удалить с начала ('z')

// Доступ
$list->bottom();        // первый элемент
$list->top();           // последний элемент

// Режимы итерации:
$list->setIteratorMode(
    SplDoublyLinkedList::IT_MODE_LIFO |  // обход в обратном порядке (стек)
    SplDoublyLinkedList::IT_MODE_DELETE  // удалять элементы при обходе
);

// SplStack и SplQueue наследуют SplDoublyLinkedList
```

### SplObjectStorage - хранилище объектов (set)

```php
$storage = new SplObjectStorage();

$obj1 = new stdClass();
$obj2 = new stdClass();
$obj3 = new stdClass();

// Добавить объекты
$storage->attach($obj1);
$storage->attach($obj2, 'metadata for obj2');  // с метаданными

// Проверить наличие
$storage->contains($obj1);  // true

// Получить метаданные
$storage->getInfo();  // 'metadata for obj2'

// Удалить
$storage->detach($obj1);

// Количество
$storage->count();  // 1

// Обход
foreach ($storage as $object) {
    $info = $storage->getInfo();  // метаданные
    echo get_class($object) . ": {$info}\n";
}

// Пример: Event Listeners
class EventDispatcher {
    private SplObjectStorage $listeners;
    
    public function __construct() {
        $this->listeners = new SplObjectStorage();
    }
    
    public function addListener(object $listener, int $priority = 0): void {
        $this->listeners->attach($listener, $priority);
    }
    
    public function removeListener(object $listener): void {
        $this->listeners->detach($listener);
    }
    
    public function dispatch(string $event): void {
        foreach ($this->listeners as $listener) {
            $listener->handle($event);
        }
    }
}
```

---

## ⚠️ Исключения (Exceptions)

### SPL Exception Hierarchy

```php
Exception
├── LogicException (ошибки логики программы, должны быть исправлены)
│   ├── BadFunctionCallException
│   │   └── BadMethodCallException
│   ├── DomainException
│   ├── InvalidArgumentException
│   ├── LengthException
│   └── OutOfRangeException
└── RuntimeException (ошибки времени выполнения)
    ├── OutOfBoundsException
    ├── OverflowException
    ├── RangeException
    ├── UnderflowException
    └── UnexpectedValueException
```

### LogicException - ошибки логики

**BadFunctionCallException** - функция вызвана неправильно:
```php
class Calculator {
    private bool $initialized = false;
    
    public function calculate(): int {
        if (!$this->initialized) {
            throw new BadFunctionCallException('Calculator not initialized');
        }
        return 42;
    }
}
```

**BadMethodCallException** - метод вызван неправильно:
```php
class User {
    public function __call(string $name, array $args) {
        throw new BadMethodCallException("Method {$name} does not exist");
    }
}
```

**DomainException** - значение не в допустимой области:
```php
class Age {
    public function __construct(private int $age) {
        if ($age < 0 || $age > 150) {
            throw new DomainException('Age must be between 0 and 150');
        }
    }
}
```

**InvalidArgumentException** - неверный аргумент:
```php
function divide(int $a, int $b): float {
    if ($b === 0) {
        throw new InvalidArgumentException('Division by zero');
    }
    return $a / $b;
}

// Самое часто используемое SPL исключение!
```

**LengthException** - неверная длина:
```php
class Password {
    public function __construct(private string $password) {
        if (strlen($password) < 8) {
            throw new LengthException('Password must be at least 8 characters');
        }
    }
}
```

**OutOfRangeException** - индекс вне диапазона (compile-time):
```php
class FixedArray {
    private array $data = [1, 2, 3];
    
    public function get(int $index): mixed {
        if ($index < 0 || $index >= count($this->data)) {
            throw new OutOfRangeException("Index {$index} out of range");
        }
        return $this->data[$index];
    }
}
```

### RuntimeException - ошибки времени выполнения

**OutOfBoundsException** - индекс вне границ (runtime):
```php
$array = new SplFixedArray(3);
$array[5] = 'value';  // OutOfBoundsException
```

**OverflowException** - переполнение контейнера:
```php
class LimitedQueue {
    private SplQueue $queue;
    private int $maxSize;
    
    public function __construct(int $maxSize) {
        $this->queue = new SplQueue();
        $this->maxSize = $maxSize;
    }
    
    public function enqueue($item): void {
        if ($this->queue->count() >= $this->maxSize) {
            throw new OverflowException('Queue is full');
        }
        $this->queue->enqueue($item);
    }
}
```

**UnderflowException** - попытка взять элемент из пустого контейнера:
```php
$stack = new SplStack();
$stack->pop();  // UnderflowException: Can't pop from an empty datastructure
```

**RangeException** - арифметическое значение вне диапазона:
```php
function sqrt(float $x): float {
    if ($x < 0) {
        throw new RangeException('Cannot calculate sqrt of negative number');
    }
    return sqrt($x);
}
```

**UnexpectedValueException** - неожиданное значение:
```php
function parseJson(string $json): array {
    $data = json_decode($json, true);
    
    if (!is_array($data)) {
        throw new UnexpectedValueException('Expected JSON array');
    }
    
    return $data;
}
```

---

## 📁 Работа с файлами

### SplFileInfo - информация о файле

```php
$file = new SplFileInfo('/path/to/file.txt');

// Информация о файле
$file->getFilename();      // 'file.txt'
$file->getBasename();      // 'file.txt'
$file->getExtension();     // 'txt'
$file->getPath();          // '/path/to'
$file->getPathname();      // '/path/to/file.txt'
$file->getRealPath();      // абсолютный путь

// Проверки
$file->isFile();           // true если файл
$file->isDir();            // true если директория
$file->isLink();           // true если symlink
$file->isReadable();       // можно читать
$file->isWritable();       // можно писать
$file->isExecutable();     // можно выполнить

// Размер и время
$file->getSize();          // размер в байтах
$file->getMTime();         // время изменения (timestamp)
$file->getATime();         // время доступа
$file->getCTime();         // время создания

// Права доступа
$file->getPerms();         // права в числовом виде
$file->getOwner();         // владелец (UID)
$file->getGroup();         // группа (GID)

// Открыть файл
$fileObject = $file->openFile('r');
```

### SplFileObject - работа с файлом

```php
$file = new SplFileObject('/path/to/file.txt', 'r');

// Чтение построчно
foreach ($file as $line) {
    echo $line;
}

// Или:
while (!$file->eof()) {
    $line = $file->fgets();  // прочитать строку
    echo $line;
}

// Запись
$file = new SplFileObject('/path/to/file.txt', 'w');
$file->fwrite("Hello, World!\n");
$file->fwrite("Second line\n");

// CSV
$file = new SplFileObject('data.csv', 'r');
$file->setFlags(SplFileObject::READ_CSV);

foreach ($file as $row) {
    var_dump($row);  // массив из CSV колонок
}

// Запись CSV
$file = new SplFileObject('output.csv', 'w');
$file->fputcsv(['Name', 'Age', 'Email']);
$file->fputcsv(['John', 30, 'john@example.com']);

// Seek (перемотка к строке)
$file->seek(10);  // перейти к 11-й строке (0-based)
$line = $file->current();

// Методы:
$file->feof();         // конец файла
$file->fflush();       // сбросить буфер
$file->flock(LOCK_EX); // блокировка файла
$file->fread(1024);    // прочитать байты
$file->fstat();        // статистика
$file->ftell();        // текущая позиция
$file->ftruncate(0);   // обрезать файл
```

### SplTempFileObject - временный файл в памяти

```php
// Файл в памяти (до 2MB, потом на диск)
$temp = new SplTempFileObject(2 * 1024 * 1024);  // 2MB limit

$temp->fwrite("Temporary data\n");
$temp->fwrite("More data\n");

// Чтение
$temp->rewind();
while (!$temp->eof()) {
    echo $temp->fgets();
}

// Полезно для тестов или обработки данных без файлов на диске
```

---

## 🔧 Функции SPL для массивов

### class_implements() - получить интерфейсы класса

```php
interface Loggable { }
interface Cacheable { }

class User implements Loggable, Cacheable { }

$interfaces = class_implements(User::class);
// ['Loggable' => 'Loggable', 'Cacheable' => 'Cacheable']

// Проверка реализации интерфейса:
if (in_array(Loggable::class, class_implements($user))) {
    // User реализует Loggable
}

// С автозагрузкой:
class_implements($className, autoload: true);
```

### class_parents() - получить родительские классы

```php
class Animal { }
class Mammal extends Animal { }
class Dog extends Mammal { }

$parents = class_parents(Dog::class);
// ['Mammal' => 'Mammal', 'Animal' => 'Animal']

// С автозагрузкой:
class_parents($className, autoload: true);
```

### class_uses() - получить traits класса

```php
trait Timestampable { }
trait SoftDeletes { }

class User {
    use Timestampable, SoftDeletes;
}

$traits = class_uses(User::class);
// ['Timestampable' => 'Timestampable', 'SoftDeletes' => 'SoftDeletes']

// Рекурсивно (включая traits родителей):
function class_uses_recursive($class) {
    $traits = [];
    
    do {
        $traits = array_merge(class_uses($class), $traits);
    } while ($class = get_parent_class($class));
    
    return array_unique($traits);
}
```

### iterator_to_array() - конвертировать итератор в массив

```php
$iterator = new ArrayIterator([1, 2, 3, 4, 5]);

$array = iterator_to_array($iterator);
// [1, 2, 3, 4, 5]

// Без сохранения ключей:
$array = iterator_to_array($iterator, use_keys: false);
// [0 => 1, 1 => 2, 2 => 3, 3 => 4, 4 => 5]

// Полезно для Generator:
function generateNumbers() {
    yield 1;
    yield 2;
    yield 3;
}

$array = iterator_to_array(generateNumbers());
// [0 => 1, 1 => 2, 2 => 3]
```

### iterator_count() - посчитать элементы в итераторе

```php
$iterator = new ArrayIterator([1, 2, 3, 4, 5]);

$count = iterator_count($iterator);  // 5

// ⚠️ Обходит весь итератор! Для Generator это "потребляет" его
```

### iterator_apply() - применить функцию к итератору

```php
$iterator = new ArrayIterator([1, 2, 3, 4, 5]);

$result = iterator_apply($iterator, function($iterator) {
    echo $iterator->current() . "\n";
    return true;  // продолжить
}, [$iterator]);

// Вызовет функцию для каждого элемента
```

---

## 🔐 Autoloading

### spl_autoload_register() - регистрация автозагрузчика

```php
// Простой автозагрузчик
spl_autoload_register(function ($class) {
    $file = __DIR__ . '/' . str_replace('\\', '/', $class) . '.php';
    
    if (file_exists($file)) {
        require $file;
    }
});

// PSR-4 автозагрузчик
spl_autoload_register(function ($class) {
    $prefix = 'App\\';
    $baseDir = __DIR__ . '/src/';
    
    $len = strlen($prefix);
    if (strncmp($prefix, $class, $len) !== 0) {
        return;  // не наш namespace
    }
    
    $relativeClass = substr($class, $len);
    $file = $baseDir . str_replace('\\', '/', $relativeClass) . '.php';
    
    if (file_exists($file)) {
        require $file;
    }
});

// Использование:
$user = new App\Models\User();  // автоматически загрузит src/Models/User.php

// Composer использует spl_autoload_register для PSR-4!
```

### spl_autoload_functions() - список автозагрузчиков

```php
$autoloaders = spl_autoload_functions();
// [callable, callable, ...]

// Проверить есть ли Composer автозагрузчик
```

### spl_autoload_unregister() - удалить автозагрузчик

```php
spl_autoload_unregister($autoloader);
```

---

## 🎯 Практические примеры

### 1. Ленивая загрузка большого файла

```php
class LazyFileReader implements Iterator {
    private SplFileObject $file;
    private int $key = 0;
    private ?string $current = null;
    
    public function __construct(string $filename) {
        $this->file = new SplFileObject($filename, 'r');
    }
    
    public function current(): string {
        return $this->current;
    }
    
    public function key(): int {
        return $this->key;
    }
    
    public function next(): void {
        $this->current = $this->file->fgets();
        $this->key++;
    }
    
    public function rewind(): void {
        $this->file->rewind();
        $this->current = $this->file->fgets();
        $this->key = 0;
    }
    
    public function valid(): bool {
        return !$this->file->eof();
    }
}

// Обработка 10GB файла без загрузки в память:
$reader = new LazyFileReader('huge_file.txt');
foreach ($reader as $line) {
    processLine($line);
}
```

### 2. Фильтрация файлов по расширению

```php
class PhpFileFilterIterator extends FilterIterator {
    public function accept(): bool {
        $file = $this->getInnerIterator()->current();
        return $file->isFile() && $file->getExtension() === 'php';
    }
}

$directory = new RecursiveDirectoryIterator(__DIR__);
$iterator = new RecursiveIteratorIterator($directory);
$phpFiles = new PhpFileFilterIterator($iterator);

foreach ($phpFiles as $file) {
    echo $file->getPathname() . "\n";
}
```

### 3. Cache с TTL через SplObjectStorage

```php
class CacheItem {
    public function __construct(
        public mixed $value,
        public int $expiresAt
    ) {}
    
    public function isExpired(): bool {
        return time() > $this->expiresAt;
    }
}

class Cache {
    private SplObjectStorage $storage;
    
    public function __construct() {
        $this->storage = new SplObjectStorage();
    }
    
    public function set(string $key, mixed $value, int $ttl): void {
        $keyObj = (object)['key' => $key];
        $item = new CacheItem($value, time() + $ttl);
        $this->storage->attach($keyObj, $item);
    }
    
    public function get(string $key): mixed {
        foreach ($this->storage as $keyObj) {
            if ($keyObj->key === $key) {
                $item = $this->storage->getInfo();
                if ($item->isExpired()) {
                    $this->storage->detach($keyObj);
                    return null;
                }
                return $item->value;
            }
        }
        return null;
    }
}
```

### 4. Обработка CSV с валидацией

```php
class CsvValidator extends FilterIterator {
    private int $expectedColumns;
    
    public function __construct(Iterator $iterator, int $expectedColumns) {
        parent::__construct($iterator);
        $this->expectedColumns = $expectedColumns;
    }
    
    public function accept(): bool {
        $row = $this->getInnerIterator()->current();
        return count($row) === $this->expectedColumns;
    }
}

$file = new SplFileObject('data.csv', 'r');
$file->setFlags(SplFileObject::READ_CSV | SplFileObject::SKIP_EMPTY);

$csv = new CsvValidator($file, 3);  // ожидаем 3 колонки

foreach ($csv as $row) {
    [$name, $age, $email] = $row;
    echo "Name: {$name}, Age: {$age}, Email: {$email}\n";
}
```

---

## 🎓 Best Practices

### 1. Используй Iterator вместо загрузки в память

```php
// ❌ Плохо (загружает весь файл в память)
$lines = file('huge_file.txt');
foreach ($lines as $line) {
    processLine($line);
}

// ✅ Хорошо (читает построчно)
$file = new SplFileObject('huge_file.txt', 'r');
foreach ($file as $line) {
    processLine($line);
}
```

### 2. Используй SPL exceptions вместо общих

```php
// ❌ Плохо
function divide(int $a, int $b): float {
    if ($b === 0) {
        throw new Exception('Division by zero');
    }
    return $a / $b;
}

// ✅ Хорошо
function divide(int $a, int $b): float {
    if ($b === 0) {
        throw new InvalidArgumentException('Division by zero');
    }
    return $a / $b;
}
```

### 3. Используй SplFileObject вместо fopen/fgets

```php
// ❌ Плохо (процедурный стиль)
$handle = fopen('file.txt', 'r');
while (!feof($handle)) {
    $line = fgets($handle);
    echo $line;
}
fclose($handle);

// ✅ Хорошо (ООП стиль)
$file = new SplFileObject('file.txt', 'r');
foreach ($file as $line) {
    echo $line;
}
// Файл закрывается автоматически
```

### 4. Используй IteratorAggregate для коллекций

```php
// ✅ Делегируй итерацию вместо реализации всех методов Iterator
class UserCollection implements IteratorAggregate, Countable {
    private array $users = [];
    
    public function add(User $user): void {
        $this->users[] = $user;
    }
    
    public function getIterator(): Traversable {
        return new ArrayIterator($this->users);
    }
    
    public function count(): int {
        return count($this->users);
    }
}
```

---

## 🎓 Для собеседования: ключевые точки

1. **Iterator interface** - 5 методов (current, key, next, rewind, valid)
2. **IteratorAggregate** - делегирование через getIterator()
3. **ArrayIterator** - обертка для массивов
4. **FilterIterator** - фильтрация элементов
5. **RecursiveIteratorIterator** - обход вложенных структур
6. **SplStack/SplQueue** - LIFO/FIFO структуры данных
7. **SplPriorityQueue** - очередь с приоритетом
8. **SplFixedArray** - фиксированный массив (быстрее для больших данных)
9. **SPL Exceptions** - InvalidArgumentException (самое частое), DomainException, RangeException
10. **SplFileObject** - ООП работа с файлами, построчное чтение
11. **spl_autoload_register** - PSR-4 autoloading
12. **class_implements/parents/uses** - рефлексия классов

**Главное:** SPL для эффективной работы с большими данными, итераторы вместо загрузки в память, специализированные исключения вместо Exception.

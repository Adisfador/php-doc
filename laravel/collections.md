# Laravel Collections - Коллекции

Полный разбор Collections API в Laravel - мощный инструмент для работы с массивами данных.

---

## 🎯 Что такое Collections?

**Collection** - обертка над массивом с удобными методами для трансформации и фильтрации данных.

**Преимущества:**
- 🔗 Chainable (цепочки методов)
- 💡 Выразительный и читаемый код
- 🚀 Lazy evaluation (отложенное выполнение)
- 🎯 Immutable операции

```php
// Вместо этого
$activeUsers = [];
foreach ($users as $user) {
    if ($user->status === 'active') {
        $activeUsers[] = strtoupper($user->name);
    }
}

// Пишем так
$activeUsers = collect($users)
    ->filter(fn($user) => $user->status === 'active')
    ->map(fn($user) => strtoupper($user->name))
    ->values();
```

---

## 📦 Создание коллекций

### Через collect() helper

```php
// Из массива
$collection = collect([1, 2, 3, 4, 5]);

// Из Eloquent результата (автоматически)
$users = User::where('active', true)->get(); // Возвращает Collection

// Из Query Builder (нужно вручную)
$users = collect(DB::table('users')->get());

// Пустая коллекция
$empty = collect();
```

### Создание вручную

```php
use Illuminate\Support\Collection;

$collection = new Collection([1, 2, 3]);
```

---

## 🔍 Основные методы

### all() - Получить массив

```php
$collection = collect([1, 2, 3]);

$array = $collection->all(); // [1, 2, 3]
```

### toArray() - Рекурсивное преобразование

```php
$collection = collect([
    'user' => User::find(1),
    'posts' => Post::where('user_id', 1)->get(),
]);

// Преобразует все модели в массивы
$array = $collection->toArray();
```

### toJson() - В JSON

```php
$collection = collect(['name' => 'John', 'age' => 30]);

$json = $collection->toJson(); // {"name":"John","age":30}
```

### count() - Количество элементов

```php
$collection = collect([1, 2, 3, 4, 5]);

$collection->count(); // 5
```

---

## 🎯 Фильтрация

### filter() - Фильтровать элементы

```php
$collection = collect([1, 2, 3, 4, 5]);

// Вернуть только четные
$filtered = $collection->filter(function ($value) {
    return $value % 2 === 0;
});
// [2, 4]

// Short closure
$filtered = $collection->filter(fn($value) => $value > 3);
// [4, 5]
```

### where() - Фильтровать по ключу

```php
$collection = collect([
    ['name' => 'John', 'age' => 30],
    ['name' => 'Jane', 'age' => 25],
    ['name' => 'Bob', 'age' => 30],
]);

$filtered = $collection->where('age', 30);
// [['name' => 'John', 'age' => 30], ['name' => 'Bob', 'age' => 30]]

// Операторы
$filtered = $collection->where('age', '>', 25);
$filtered = $collection->where('age', '>=', 30);
$filtered = $collection->where('name', '!=', 'John');
```

### whereIn() / whereNotIn()

```php
$collection = collect([
    ['product' => 'Apple', 'price' => 100],
    ['product' => 'Banana', 'price' => 50],
    ['product' => 'Orange', 'price' => 75],
]);

$filtered = $collection->whereIn('product', ['Apple', 'Orange']);
// Apple и Orange

$filtered = $collection->whereNotIn('price', [50]);
// Все кроме Banana
```

### whereBetween() / whereNotBetween()

```php
$collection = collect([
    ['product' => 'Apple', 'price' => 100],
    ['product' => 'Banana', 'price' => 50],
    ['product' => 'Orange', 'price' => 75],
]);

$filtered = $collection->whereBetween('price', [60, 90]);
// Orange (75)
```

### whereNull() / whereNotNull()

```php
$collection = collect([
    ['name' => 'John', 'email' => 'john@example.com'],
    ['name' => 'Jane', 'email' => null],
]);

$withEmail = $collection->whereNotNull('email');
// John

$withoutEmail = $collection->whereNull('email');
// Jane
```

### first() / firstWhere() / last()

```php
$collection = collect([1, 2, 3, 4, 5]);

$first = $collection->first(); // 1

$first = $collection->first(fn($value) => $value > 3); // 4

// firstWhere
$users = collect([
    ['name' => 'John', 'active' => true],
    ['name' => 'Jane', 'active' => false],
]);

$user = $users->firstWhere('active', true); // John

$last = $collection->last(); // 5
```

### reject() - Обратная filter

```php
$collection = collect([1, 2, 3, 4, 5]);

// Убрать четные
$filtered = $collection->reject(fn($value) => $value % 2 === 0);
// [1, 3, 5]
```

---

## 🔄 Трансформация

### map() - Трансформировать элементы

```php
$collection = collect([1, 2, 3, 4, 5]);

$multiplied = $collection->map(fn($value) => $value * 2);
// [2, 4, 6, 8, 10]

// С массивами
$users = collect([
    ['name' => 'John', 'age' => 30],
    ['name' => 'Jane', 'age' => 25],
]);

$names = $users->map(fn($user) => $user['name']);
// ['John', 'Jane']
```

### mapWithKeys() - Изменить ключи

```php
$collection = collect([
    ['name' => 'John', 'age' => 30],
    ['name' => 'Jane', 'age' => 25],
]);

$mapped = $collection->mapWithKeys(function ($user) {
    return [$user['name'] => $user['age']];
});
// ['John' => 30, 'Jane' => 25]
```

### transform() - Мутирующий map

```php
$collection = collect([1, 2, 3]);

$collection->transform(fn($value) => $value * 2);
// $collection теперь [2, 4, 6] (изменяет оригинал)
```

### pluck() - Извлечь значения по ключу

```php
$collection = collect([
    ['name' => 'John', 'age' => 30],
    ['name' => 'Jane', 'age' => 25],
]);

$names = $collection->pluck('name');
// ['John', 'Jane']

// С ключами
$users = $collection->pluck('age', 'name');
// ['John' => 30, 'Jane' => 25]

// Вложенные ключи (dot notation)
$posts = collect([
    ['id' => 1, 'user' => ['name' => 'John']],
    ['id' => 2, 'user' => ['name' => 'Jane']],
]);

$names = $posts->pluck('user.name');
// ['John', 'Jane']
```

### flatten() - Разгладить многомерный массив

```php
$collection = collect([
    'name' => 'John',
    'languages' => ['PHP', 'JavaScript', 'Python'],
]);

$flattened = $collection->flatten();
// ['John', 'PHP', 'JavaScript', 'Python']

// С глубиной
$collection = collect([
    ['a' => [1, 2]],
    ['b' => [3, 4]],
]);

$flattened = $collection->flatten(1);
// [1, 2, 3, 4]
```

### flatMap() - map + flatten

```php
$collection = collect([
    ['name' => 'John', 'hobbies' => ['reading', 'gaming']],
    ['name' => 'Jane', 'hobbies' => ['cooking', 'traveling']],
]);

$hobbies = $collection->flatMap(fn($person) => $person['hobbies']);
// ['reading', 'gaming', 'cooking', 'traveling']
```

---

## 📊 Агрегация

### sum() / avg() / max() / min()

```php
$collection = collect([1, 2, 3, 4, 5]);

$collection->sum(); // 15
$collection->avg(); // 3
$collection->max(); // 5
$collection->min(); // 1

// С ключом
$products = collect([
    ['name' => 'Apple', 'price' => 100],
    ['name' => 'Banana', 'price' => 50],
]);

$total = $products->sum('price'); // 150
$avg = $products->avg('price'); // 75
```

### reduce() - Свернуть в одно значение

```php
$collection = collect([1, 2, 3, 4, 5]);

$sum = $collection->reduce(function ($carry, $item) {
    return $carry + $item;
}, 0); // 15

// Конкатенация строк
$names = collect(['John', 'Jane', 'Bob']);

$string = $names->reduce(function ($carry, $name) {
    return $carry . $name . ', ';
}, 'Names: '); // "Names: John, Jane, Bob, "
```

---

## 🔢 Группировка и сортировка

### groupBy() - Группировать по ключу

```php
$collection = collect([
    ['name' => 'John', 'department' => 'IT'],
    ['name' => 'Jane', 'department' => 'HR'],
    ['name' => 'Bob', 'department' => 'IT'],
]);

$grouped = $collection->groupBy('department');
/*
[
    'IT' => [
        ['name' => 'John', 'department' => 'IT'],
        ['name' => 'Bob', 'department' => 'IT']
    ],
    'HR' => [
        ['name' => 'Jane', 'department' => 'HR']
    ]
]
*/

// С callback
$grouped = $collection->groupBy(fn($user) => strtolower($user['department']));
```

### keyBy() - Индексировать по ключу

```php
$collection = collect([
    ['id' => 1, 'name' => 'John'],
    ['id' => 2, 'name' => 'Jane'],
]);

$keyed = $collection->keyBy('id');
/*
[
    1 => ['id' => 1, 'name' => 'John'],
    2 => ['id' => 2, 'name' => 'Jane']
]
*/
```

### sort() / sortBy() / sortByDesc()

```php
$collection = collect([5, 3, 1, 2, 4]);

$sorted = $collection->sort();
// [1, 2, 3, 4, 5]

// По ключу
$users = collect([
    ['name' => 'John', 'age' => 30],
    ['name' => 'Jane', 'age' => 25],
    ['name' => 'Bob', 'age' => 35],
]);

$sorted = $users->sortBy('age');
// Jane (25), John (30), Bob (35)

$sorted = $users->sortByDesc('age');
// Bob (35), John (30), Jane (25)

// С callback
$sorted = $users->sortBy(fn($user) => $user['name']);
```

### sortKeys() / sortKeysDesc()

```php
$collection = collect([
    'b' => 2,
    'a' => 1,
    'c' => 3,
]);

$sorted = $collection->sortKeys();
// ['a' => 1, 'b' => 2, 'c' => 3]
```

### reverse()

```php
$collection = collect([1, 2, 3]);

$reversed = $collection->reverse();
// [3, 2, 1]
```

---

## ✂️ Разбиение и объединение

### chunk() - Разбить на части

```php
$collection = collect([1, 2, 3, 4, 5, 6, 7]);

$chunks = $collection->chunk(3);
/*
[
    [1, 2, 3],
    [4, 5, 6],
    [7]
]
*/
```

### split() - Разделить на N частей

```php
$collection = collect([1, 2, 3, 4, 5]);

$split = $collection->split(2);
// [[1, 2, 3], [4, 5]]
```

### sliding() - Скользящее окно

```php
$collection = collect([1, 2, 3, 4, 5]);

$sliding = $collection->sliding(2);
/*
[
    [1, 2],
    [2, 3],
    [3, 4],
    [4, 5]
]
*/

// С шагом
$sliding = $collection->sliding(2, step: 3);
// [[1, 2], [4, 5]]
```

### partition() - Разделить по условию

```php
$collection = collect([1, 2, 3, 4, 5, 6]);

[$even, $odd] = $collection->partition(fn($value) => $value % 2 === 0);
// $even: [2, 4, 6]
// $odd: [1, 3, 5]
```

### merge() - Объединить коллекции

```php
$collection1 = collect(['a', 'b']);
$collection2 = collect(['c', 'd']);

$merged = $collection1->merge($collection2);
// ['a', 'b', 'c', 'd']

// С ассоциативными массивами (перезаписывает)
$collection1 = collect(['name' => 'John', 'age' => 30]);
$collection2 = collect(['age' => 25, 'city' => 'NYC']);

$merged = $collection1->merge($collection2);
// ['name' => 'John', 'age' => 25, 'city' => 'NYC']
```

### concat() - Добавить значения

```php
$collection = collect(['John']);

$concatenated = $collection->concat(['Jane'])->concat(['Bob']);
// ['John', 'Jane', 'Bob']
```

### union() - Объединение без перезаписи

```php
$collection1 = collect(['a' => 1, 'b' => 2]);
$collection2 = collect(['a' => 3, 'c' => 4]);

$union = $collection1->union($collection2);
// ['a' => 1, 'b' => 2, 'c' => 4] (a не перезаписывается)
```

---

## 🎲 Выборка и извлечение

### take() / skip()

```php
$collection = collect([1, 2, 3, 4, 5]);

$taken = $collection->take(3);
// [1, 2, 3]

$skipped = $collection->skip(2);
// [3, 4, 5]

// Отрицательные значения (с конца)
$taken = $collection->take(-2);
// [4, 5]
```

### slice() - Извлечь часть

```php
$collection = collect([1, 2, 3, 4, 5]);

$slice = $collection->slice(2);
// [3, 4, 5]

$slice = $collection->slice(1, 3);
// [2, 3, 4]
```

### random() - Случайные элементы

```php
$collection = collect([1, 2, 3, 4, 5]);

$random = $collection->random();
// Один случайный элемент

$random = $collection->random(2);
// Два случайных элемента
```

### unique() - Уникальные значения

```php
$collection = collect([1, 2, 2, 3, 3, 3]);

$unique = $collection->unique();
// [1, 2, 3]

// По ключу
$users = collect([
    ['name' => 'John', 'role' => 'admin'],
    ['name' => 'Jane', 'role' => 'user'],
    ['name' => 'Bob', 'role' => 'admin'],
]);

$unique = $users->unique('role');
// John (admin), Jane (user)
```

### duplicates() - Найти дубликаты

```php
$collection = collect([1, 2, 2, 3, 3, 3]);

$duplicates = $collection->duplicates();
// [2 => 2, 4 => 3, 5 => 3]
```

---

## 🔗 Проверки

### contains() - Содержит элемент

```php
$collection = collect([1, 2, 3, 4, 5]);

$collection->contains(3); // true
$collection->contains(10); // false

// С callback
$collection->contains(fn($value) => $value > 5); // false

// По ключу
$users = collect([
    ['name' => 'John', 'active' => true],
]);

$users->contains('name', 'John'); // true
```

### isEmpty() / isNotEmpty()

```php
$collection = collect([]);

$collection->isEmpty(); // true
$collection->isNotEmpty(); // false
```

### every() - Все элементы соответствуют

```php
$collection = collect([2, 4, 6, 8]);

$allEven = $collection->every(fn($value) => $value % 2 === 0); // true
```

### some() - Хотя бы один соответствует

```php
$collection = collect([1, 2, 3]);

$hasEven = $collection->some(fn($value) => $value % 2 === 0); // true (есть 2)
```

---

## 🚀 Lazy Collections

**Lazy Collections** - отложенное выполнение для работы с большими данными.

### Создание

```php
use Illuminate\Support\LazyCollection;

// Из генератора
$lazyCollection = LazyCollection::make(function () {
    foreach (range(1, 1000000) as $number) {
        yield $number;
    }
});

// Из курсора Eloquent
$users = User::cursor(); // LazyCollection
```

### Преимущества

```php
// Обычная коллекция - загружает всё в память
$users = User::all()->filter(fn($user) => $user->active);

// Lazy - обрабатывает по одному
$users = User::cursor()->filter(fn($user) => $user->active);

// Чтение большого файла
$lines = LazyCollection::make(function () {
    $handle = fopen('large-file.txt', 'r');
    
    while (($line = fgets($handle)) !== false) {
        yield $line;
    }
    
    fclose($handle);
});

$lines->chunk(100)->each(function ($chunk) {
    // Обрабатывать по 100 строк
});
```

---

## 🎯 Higher Order Messages

Упрощенный синтаксис для частых операций.

```php
$users = collect([
    ['name' => 'John', 'active' => true],
    ['name' => 'Jane', 'active' => false],
]);

// Вместо
$names = $users->map(fn($user) => $user['name']);

// Можно
$names = $users->map->name;

// Вместо
$users->each(fn($user) => $user->activate());

// Можно
$users->each->activate();

// Вместо
$collection->filter(fn($item) => $item->isActive());

// Можно
$collection->filter->isActive();
```

---

## 🧪 Условные методы

### when() / unless()

```php
$collection = collect([1, 2, 3]);

$collection = $collection->when(true, function ($collection) {
    return $collection->push(4);
}); // [1, 2, 3, 4]

$collection = $collection->unless(false, function ($collection) {
    return $collection->push(5);
}); // [1, 2, 3, 4, 5]

// С параметром
$status = 'active';

$users = User::all()->when($status, function ($collection, $status) {
    return $collection->where('status', $status);
});
```

### whenEmpty() / whenNotEmpty()

```php
$collection = collect([]);

$collection->whenEmpty(function ($collection) {
    return $collection->push('default');
}); // ['default']

$collection = collect([1, 2, 3]);

$collection->whenNotEmpty(function ($collection) {
    return $collection->push(4);
}); // [1, 2, 3, 4]
```

---

## 🔧 Макросы

Расширение Collection своими методами.

```php
use Illuminate\Support\Collection;

// AppServiceProvider::boot()
Collection::macro('toUpper', function () {
    return $this->map(fn($value) => strtoupper($value));
});

// Использование
$collection = collect(['john', 'jane', 'bob']);

$collection->toUpper(); // ['JOHN', 'JANE', 'BOB']

// Макрос с параметрами
Collection::macro('prefixWith', function ($prefix) {
    return $this->map(fn($value) => $prefix . $value);
});

$collection->prefixWith('Mr. '); // ['Mr. john', 'Mr. jane', 'Mr. bob']
```

---

## 📚 Практические примеры

### Пример 1: Обработка заказов

```php
$orders = Order::with('items')->get();

$report = $orders
    ->groupBy('status')
    ->map(function ($orders, $status) {
        return [
            'status' => $status,
            'count' => $orders->count(),
            'total' => $orders->sum('total'),
            'avg' => $orders->avg('total'),
        ];
    })
    ->sortByDesc('total');
```

### Пример 2: Трансформация данных для API

```php
$users = User::with('posts')->get();

$response = $users->map(function ($user) {
    return [
        'id' => $user->id,
        'name' => $user->name,
        'posts_count' => $user->posts->count(),
        'latest_post' => $user->posts->sortByDesc('created_at')->first()?->title,
    ];
});
```

### Пример 3: Построение дерева категорий

```php
$categories = Category::all();

$tree = $categories
    ->whereNull('parent_id')
    ->map(function ($category) use ($categories) {
        return [
            'id' => $category->id,
            'name' => $category->name,
            'children' => $categories
                ->where('parent_id', $category->id)
                ->values(),
        ];
    });
```

---

## 🎓 Best Practices

### 1. Используй Lazy Collections для больших данных

```php
// ❌ ПЛОХО - загружает всё в память
User::all()->each(function ($user) {
    // process
});

// ✅ ХОРОШО - обрабатывает по одному
User::cursor()->each(function ($user) {
    // process
});
```

### 2. Не злоупотребляй цепочками

```php
// ❌ ПЛОХО - неразборчиво
$result = $collection->filter()->map()->groupBy()->flatten()->unique()->values();

// ✅ ХОРОШО - разбить на шаги
$filtered = $collection->filter(fn($item) => $item->active);
$transformed = $filtered->map(fn($item) => $item->name);
$unique = $transformed->unique();
```

### 3. Используй tap() для дебага

```php
$collection
    ->filter(fn($item) => $item->active)
    ->tap(fn($collection) => logger('After filter: ' . $collection->count()))
    ->map(fn($item) => $item->name)
    ->tap(fn($collection) => logger('After map: ' . $collection->count()));
```

### 4. Предпочитай специализированные методы

```php
// ❌ ПЛОХО
$collection->filter(fn($item) => $item['status'] === 'active');

// ✅ ХОРОШО
$collection->where('status', 'active');
```

---

## 🎓 Для собеседования: ключевые точки

1. **Collection vs Array** - Collections immutable, chainable, выразительный API
2. **Lazy Collections** - для больших данных, отложенное выполнение, cursor()
3. **map vs each** - map возвращает новую коллекцию, each для side-effects
4. **pluck** - извлечение значений, работает с dot notation
5. **groupBy** - группировка данных по ключу
6. **filter vs reject** - filter оставляет true, reject убирает true
7. **Higher Order Messages** - $collection->map->property вместо callback
8. **when/unless** - условное применение методов
9. **Макросы** - расширение Collection кастомными методами
10. **chunk** - для обработки по частям (batch processing)

**Главное:** Collections делают код проще и выразительнее, но для больших данных используй LazyCollection.

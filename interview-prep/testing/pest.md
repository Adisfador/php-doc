# Pest - современный testing framework

## Основы Pest

### Что такое Pest
- Тестовый фреймворк построенный на PHPUnit
- Минималистичный и выразительный синтаксис
- Не требует классов и методов
- Богатый набор expectations
- Архитектурное тестирование из коробки

### Установка
```bash
composer require pestphp/pest --dev --with-all-dependencies
./vendor/bin/pest --init

# Laravel
composer require pestphp/pest-plugin-laravel --dev
```

### Структура теста
```php
test('basic math works', function () {
    expect(2 + 2)->toBe(4);
});

it('calculates sum correctly', function () {
    $result = sum(2, 3);
    expect($result)->toBe(5);
});
```

## Expectations API

### Базовые
```php
expect($value)->toBe(5);                 // ===
expect($value)->toEqual(5);              // ==
expect($value)->not->toBe(5);            // !==

expect($value)->toBeTrue();
expect($value)->toBeFalse();
expect($value)->toBeNull();
expect($value)->toBeEmpty();
```

### Типы
```php
expect($value)->toBeInt();
expect($value)->toBeFloat();
expect($value)->toBeString();
expect($value)->toBeBool();
expect($value)->toBeArray();
expect($value)->toBeObject();
expect($value)->toBeInstanceOf(User::class);
```

### Числа
```php
expect($value)->toBeGreaterThan(5);
expect($value)->toBeLessThan(10);
expect($value)->toBeGreaterThanOrEqual(5);
expect($value)->toBeBetween(1, 10);
```

### Массивы
```php
expect($array)->toHaveCount(3);
expect($array)->toContain('value');
expect($array)->toHaveKey('key');
expect($array)->each->toBeString();      // все элементы
```

### Строки
```php
expect($string)->toContain('substring');
expect($string)->toStartWith('prefix');
expect($string)->toEndWith('suffix');
expect($string)->toMatch('/pattern/');
expect($string)->toHaveLength(10);
```

### Exceptions
```php
expect(fn() => throw new Exception())->toThrow(Exception::class);
expect(fn() => divide(5, 0))->toThrow(DivisionByZeroError::class, 'Division by zero');
```

### JSON
```php
expect($json)->json()->toHaveCount(3);
expect($json)->json()->toMatchArray(['key' => 'value']);
```

## Hooks (Setup/Teardown)

### beforeEach / afterEach
```php
beforeEach(function () {
    $this->user = User::factory()->create();
});

afterEach(function () {
    // cleanup
});

test('user has name', function () {
    expect($this->user->name)->not->toBeEmpty();
});
```

### beforeAll / afterAll
```php
beforeAll(function () {
    // Выполняется один раз перед всеми тестами
});

afterAll(function () {
    // Выполняется один раз после всех тестов
});
```

## Datasets (Data Providers)

### Inline dataset
```php
it('adds numbers correctly', function (int $a, int $b, int $result) {
    expect($a + $b)->toBe($result);
})->with([
    [1, 2, 3],
    [5, 5, 10],
    [10, -5, 5],
]);
```

### Named datasets
```php
it('validates emails', function (string $email, bool $valid) {
    expect(isValidEmail($email))->toBe($valid);
})->with([
    'valid email' => ['test@example.com', true],
    'invalid email' => ['invalid', false],
    'empty email' => ['', false],
]);
```

### Shared datasets
```php
// tests/Datasets/Users.php
dataset('users', [
    'admin' => [['role' => 'admin']],
    'user' => [['role' => 'user']],
]);

// tests/Feature/UserTest.php
it('creates user', function (array $data) {
    $user = User::create($data);
    expect($user)->toBeInstanceOf(User::class);
})->with('users');
```

### Lazy datasets
```php
dataset('users', function () {
    return User::factory()->count(10)->create();
});
```

### Combining datasets
```php
it('tests combinations', function ($a, $b) {
    // ...
})->with([1, 2, 3])->with([10, 20]);
// Создаёт 6 тестов: 1+10, 1+20, 2+10, 2+20, 3+10, 3+20
```

## Higher Order Testing

### Higher order expectations
```php
expect($users)
    ->each->toBeInstanceOf(User::class)
    ->each->email->toContain('@');

expect($collection)
    ->sequence(
        fn($item) => $item->id->toBe(1),
        fn($item) => $item->id->toBe(2),
        fn($item) => $item->id->toBe(3)
    );
```

### Higher order tests
```php
it('validates all items')
    ->expect(fn() => ['item1', 'item2', 'item3'])
    ->each->toBeString();
```

## Architectural Testing

### Проверка зависимостей
```php
test('models do not use facades')
    ->expect('App\Models')
    ->not->toUse('Illuminate\Support\Facades');

test('controllers extend base controller')
    ->expect('App\Http\Controllers')
    ->toExtend('App\Http\Controllers\Controller');
```

### Naming conventions
```php
test('controllers have Controller suffix')
    ->expect('App\Http\Controllers')
    ->toHaveSuffix('Controller');

test('repositories have Repository suffix')
    ->expect('App\Repositories')
    ->toHaveSuffix('Repository');
```

### Dependency rules
```php
test('domain does not depend on infrastructure')
    ->expect('App\Domain')
    ->not->toUse('App\Infrastructure');

test('services layer structure')
    ->expect('App\Services')
    ->toOnlyUse([
        'App\Models',
        'App\Repositories',
        'App\Events',
    ]);
```

### Final classes
```php
test('value objects are final')
    ->expect('App\ValueObjects')
    ->toBeFinal();
```

## Laravel Integration

### HTTP Tests
```php
it('returns successful response', function () {
    $response = $this->get('/api/users');
    
    $response->assertOk();
    expect($response->json())->toHaveCount(10);
});

it('creates user', function () {
    $this->post('/users', ['name' => 'John'])
        ->assertCreated()
        ->assertJson(['name' => 'John']);
});
```

### Database
```php
use function Pest\Laravel\{assertDatabaseHas, assertDatabaseMissing};

it('creates user in database', function () {
    User::create(['email' => 'test@example.com']);
    
    assertDatabaseHas('users', ['email' => 'test@example.com']);
});

it('soft deletes user', function () {
    $user = User::factory()->create();
    $user->delete();
    
    assertDatabaseHas('users', ['id' => $user->id]);
    expect($user->trashed())->toBeTrue();
});
```

### Authentication
```php
use function Pest\Laravel\{actingAs};

it('requires authentication', function () {
    $this->get('/dashboard')->assertRedirect('/login');
});

it('allows access for authenticated users', function () {
    $user = User::factory()->create();
    
    actingAs($user)
        ->get('/dashboard')
        ->assertOk();
});
```

## Custom Expectations

### Создание custom expectation
```php
// tests/Pest.php
expect()->extend('toBeWithinRange', function (int $min, int $max) {
    return $this->toBeGreaterThanOrEqual($min)
                ->toBeLessThanOrEqual($max);
});

// Использование
test('value is in range', function () {
    expect(5)->toBeWithinRange(1, 10);
});
```

### Macro для модели
```php
expect()->extend('toBeValidUser', function () {
    return $this->toBeInstanceOf(User::class)
                ->email->toContain('@')
                ->name->not->toBeEmpty();
});

test('user is valid', function () {
    $user = User::factory()->create();
    expect($user)->toBeValidUser();
});
```

## Plugins

### Pest Plugin Laravel
```php
uses(RefreshDatabase::class);

it('refreshes database', function () {
    User::factory()->create();
    expect(User::count())->toBe(1);
});
```

### Pest Plugin Faker
```bash
composer require pestphp/pest-plugin-faker --dev
```

```php
it('generates fake data', function () {
    expect($this->faker->email)->toContain('@');
});
```

### Parallel Testing
```bash
./vendor/bin/pest --parallel
./vendor/bin/pest --parallel --processes=4
```

## Coverage

### Запуск с покрытием
```bash
./vendor/bin/pest --coverage
./vendor/bin/pest --coverage --min=80
./vendor/bin/pest --coverage-html=coverage
```

### Игнорирование файлов
```php
// tests/Pest.php
covers('App\Services');  // Тестируем только сервисы
```

## Best Practices

### Организация тестов
```
tests/
├── Unit/
│   ├── Services/
│   ├── Models/
│   └── Helpers/
├── Feature/
│   ├── Api/
│   └── Web/
├── Pest.php
└── Datasets/
```

### Именование
```php
// Используй it() для поведения
it('sends email when user registers');
it('validates user input');

// Используй test() для функциональности
test('email validation works');
test('helper functions return correct values');
```

### Группировка
```php
describe('UserService', function () {
    it('creates user', function () { /*...*/ });
    it('updates user', function () { /*...*/ });
    it('deletes user', function () { /*...*/ });
});
```

### Uses
```php
// tests/Pest.php
uses(TestCase::class)->in('Feature');
uses(TestCase::class, RefreshDatabase::class)->in('Feature/Api');

// В тестах
uses()->group('api', 'integration');
```

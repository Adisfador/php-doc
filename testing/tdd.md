# TDD - Test-Driven Development

## Основы TDD

### Что такое TDD
- Методология разработки через тестирование
- Тесты пишутся ДО кода
- Код пишется для прохождения тестов
- Непрерывный рефакторинг

### Цикл Red-Green-Refactor
1. **Red** - напиши failing тест
2. **Green** - напиши минимум кода для прохождения
3. **Refactor** - улучши код без изменения поведения
4. Повтори

### Преимущества
- Гарантированное покрытие тестами
- Лучший дизайн кода
- Документация через тесты
- Меньше багов
- Уверенность в рефакторинге
- Фокус на требованиях

### Недостатки
- Медленнее на старте
- Требует дисциплины
- Сложно применить к legacy коду
- Может привести к over-engineering

## Практика TDD

### Пример: Calculator
```php
// 1. RED - пишем тест
test('adds two numbers', function () {
    $calculator = new Calculator();
    expect($calculator->add(2, 3))->toBe(5);
});
// Тест падает - класса Calculator не существует

// 2. GREEN - минимум кода
class Calculator
{
    public function add(int $a, int $b): int
    {
        return 5; // hardcode для прохождения теста
    }
}
// Тест проходит

// 3. RED - добавляем тест
test('adds different numbers', function () {
    $calculator = new Calculator();
    expect($calculator->add(1, 1))->toBe(2);
});
// Тест падает

// 4. GREEN - реальная реализация
class Calculator
{
    public function add(int $a, int $b): int
    {
        return $a + $b;
    }
}
// Все тесты проходят

// 5. REFACTOR - если нужно
```

### Пример: User Registration
```php
// 1. RED - требование: email должен быть уникальным
test('cannot register with existing email', function () {
    $existing = User::factory()->create(['email' => 'test@example.com']);
    
    expect(fn() => User::create(['email' => 'test@example.com']))
        ->toThrow(ValidationException::class);
});

// 2. GREEN - добавляем валидацию
class User extends Model
{
    protected static function boot()
    {
        parent::boot();
        
        static::creating(function ($user) {
            if (static::where('email', $user->email)->exists()) {
                throw ValidationException::withMessages([
                    'email' => 'Email already exists'
                ]);
            }
        });
    }
}

// 3. REFACTOR - выносим в validator
class CreateUserRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'email' => 'required|email|unique:users',
        ];
    }
}
```

## Принципы TDD

### FIRST принципы
- **F**ast - тесты должны быть быстрыми
- **I**ndependent - независимы друг от друга
- **R**epeatable - воспроизводимые в любой среде
- **S**elf-validating - boolean результат (pass/fail)
- **T**imely - пишутся вовремя (до кода)

### Arrange-Act-Assert (AAA)
```php
test('user can be created', function () {
    // Arrange - подготовка
    $data = [
        'name' => 'John Doe',
        'email' => 'john@example.com',
    ];
    
    // Act - действие
    $user = User::create($data);
    
    // Assert - проверка
    expect($user->name)->toBe('John Doe');
    expect($user->email)->toBe('john@example.com');
});
```

### Given-When-Then (BDD стиль)
```php
test('user receives welcome email after registration', function () {
    // Given - исходное состояние
    Mail::fake();
    $userData = ['email' => 'john@example.com'];
    
    // When - событие
    $service = new UserRegistrationService();
    $service->register($userData);
    
    // Then - результат
    Mail::assertSent(WelcomeEmail::class, function ($mail) {
        return $mail->hasTo('john@example.com');
    });
});
```

## TDD Patterns

### Triangulation (триангуляция)
```php
// Обобщение через множественные примеры
test('calculates area of rectangle', function () {
    expect(rectangleArea(2, 3))->toBe(6);
    expect(rectangleArea(5, 4))->toBe(20);
    expect(rectangleArea(1, 1))->toBe(1);
});

// Реализация должна работать для всех случаев
function rectangleArea(int $width, int $height): int
{
    return $width * $height;
}
```

### Fake It Till You Make It
```php
// Начинаем с hardcoded значений
public function add(int $a, int $b): int
{
    return 5; // проходит первый тест
}

// Потом обобщаем
public function add(int $a, int $b): int
{
    return $a + $b; // проходит все тесты
}
```

### Obvious Implementation
```php
// Если решение очевидно - пиши сразу
test('reverses string', function () {
    expect(reverse('hello'))->toBe('olleh');
});

function reverse(string $str): string
{
    return strrev($str); // очевидная реализация
}
```

### One to Many
```php
// Сначала для одного элемента
test('finds user by id', function () {
    $user = User::factory()->create(['id' => 1]);
    expect(findUser(1))->toBe($user);
});

// Потом обобщаем на много
test('finds multiple users', function () {
    $users = User::factory()->count(3)->create();
    expect(findUsers([1, 2, 3]))->toHaveCount(3);
});
```

## TDD Workflow

### Feature development
```php
// 1. Пишем acceptance test (высокоуровневый)
test('user can place order', function () {
    $user = User::factory()->create();
    $product = Product::factory()->create();
    
    actingAs($user)
        ->post('/orders', ['product_id' => $product->id])
        ->assertCreated();
    
    expect(Order::count())->toBe(1);
});

// 2. Тест падает, начинаем unit тесты
test('order validates product exists', function () {
    expect(fn() => Order::create(['product_id' => 999]))
        ->toThrow(ValidationException::class);
});

// 3. Реализуем Order model
class Order extends Model
{
    protected static function boot()
    {
        parent::boot();
        static::creating(function ($order) {
            if (!Product::find($order->product_id)) {
                throw ValidationException::withMessages([
                    'product_id' => 'Product not found'
                ]);
            }
        });
    }
}

// 4. Продолжаем пока acceptance test не пройдёт
test('order belongs to user', function () {
    $order = Order::factory()->create(['user_id' => 1]);
    expect($order->user)->toBeInstanceOf(User::class);
});

// И так далее...
```

### Bug fixing с TDD
```php
// 1. Воспроизводим bug в тесте
test('division by zero returns null', function () {
    $calculator = new Calculator();
    // Bug: сейчас бросает exception
    expect($calculator->divide(10, 0))->toBeNull();
});

// 2. Фиксим код
class Calculator
{
    public function divide(float $a, float $b): ?float
    {
        if ($b === 0.0) {
            return null;
        }
        return $a / $b;
    }
}

// 3. Тест проходит, bug зафиксирован навсегда
```

## Типы тестов в TDD

### Unit Tests
```php
// Тестируют отдельные методы/классы
test('user has full name', function () {
    $user = new User([
        'first_name' => 'John',
        'last_name' => 'Doe'
    ]);
    
    expect($user->fullName())->toBe('John Doe');
});
```

### Integration Tests
```php
// Тестируют взаимодействие компонентов
test('user service creates user and sends email', function () {
    Mail::fake();
    
    $service = new UserService(
        new UserRepository(),
        new EmailService()
    );
    
    $user = $service->register(['email' => 'test@example.com']);
    
    expect($user)->toBeInstanceOf(User::class);
    Mail::assertSent(WelcomeEmail::class);
});
```

### Acceptance Tests (End-to-End)
```php
// Тестируют полный пользовательский сценарий
test('user can complete checkout', function () {
    $user = User::factory()->create();
    $product = Product::factory()->create(['price' => 100]);
    
    actingAs($user)
        ->post('/cart', ['product_id' => $product->id])
        ->assertOk();
    
    actingAs($user)
        ->post('/checkout', ['payment_method' => 'card'])
        ->assertRedirect('/orders');
    
    expect(Order::where('user_id', $user->id)->first())
        ->total->toBe(100);
});
```

## Best Practices

### Что тестировать
- ✅ Бизнес-логику
- ✅ Edge cases
- ✅ Граничные условия
- ✅ Error handling
- ❌ Getters/setters без логики
- ❌ Framework код
- ❌ Private методы напрямую

### Именование тестов
```php
// Хорошо - описывает сценарий
test('user cannot delete post of another user')
test('expired subscription denies access')
test('cart calculates discount for premium users')

// Плохо - неясно что тестируется
test('test user')
test('test delete')
```

### Один assertion на тест (когда возможно)
```php
// Хорошо
test('user email is validated', function () {
    expect(fn() => User::create(['email' => 'invalid']))
        ->toThrow(ValidationException::class);
});

test('user name is required', function () {
    expect(fn() => User::create(['name' => '']))
        ->toThrow(ValidationException::class);
});

// Приемлемо для связанных проверок
test('user is created with correct data', function () {
    $user = User::create(['name' => 'John', 'email' => 'john@test.com']);
    expect($user->name)->toBe('John');
    expect($user->email)->toBe('john@test.com');
});
```

### Не тестируй внутреннюю реализацию
```php
// Плохо - зависит от реализации
test('user uses bcrypt for password', function () {
    $user = User::create(['password' => 'secret']);
    expect($user->password)->toStartWith('$2y$');
});

// Хорошо - тестирует поведение
test('user can authenticate with correct password', function () {
    $user = User::create(['password' => 'secret']);
    expect(Hash::check('secret', $user->password))->toBeTrue();
});
```

### Тесты должны быть изолированы
```php
// Плохо - тесты зависят друг от друга
test('creates user', function () {
    User::create(['id' => 1, 'name' => 'John']);
});

test('finds user', function () {
    $user = User::find(1); // зависит от предыдущего теста
    expect($user->name)->toBe('John');
});

// Хорошо - каждый тест независим
test('creates user', function () {
    $user = User::create(['name' => 'John']);
    expect($user)->toBeInstanceOf(User::class);
});

test('finds user', function () {
    $user = User::factory()->create(['name' => 'John']);
    $found = User::find($user->id);
    expect($found->name)->toBe('John');
});
```

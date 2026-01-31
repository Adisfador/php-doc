# Laravel Testing - PHPUnit, HTTP Tests, Database

Полное руководство по тестированию в Laravel.

---

## Настройка тестирования

### Структура

```
tests/
├── Feature/        # Интеграционные тесты (HTTP, БД)
│   └── UserTest.php
├── Unit/          # Юнит-тесты (изолированная логика)
│   └── UserServiceTest.php
├── CreatesApplication.php
└── TestCase.php   # Базовый класс для тестов
```

### phpunit.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         bootstrap="vendor/autoload.php"
         colors="true">
    <testsuites>
        <testsuite name="Unit">
            <directory suffix="Test.php">./tests/Unit</directory>
        </testsuite>
        <testsuite name="Feature">
            <directory suffix="Test.php">./tests/Feature</directory>
        </testsuite>
    </testsuites>
    
    <php>
        <env name="APP_ENV" value="testing"/>
        <env name="DB_CONNECTION" value="sqlite"/>
        <env name="DB_DATABASE" value=":memory:"/>
        <env name="CACHE_DRIVER" value="array"/>
        <env name="SESSION_DRIVER" value="array"/>
        <env name="QUEUE_CONNECTION" value="sync"/>
        <env name="MAIL_MAILER" value="array"/>
    </php>
</phpunit>
```

### Запуск тестов

```bash
# Все тесты
php artisan test

# Или через PHPUnit
vendor/bin/phpunit

# Конкретный файл
php artisan test tests/Feature/UserTest.php

# Конкретный метод
php artisan test --filter test_user_can_login

# С покрытием кода
php artisan test --coverage

# Минимальное покрытие
php artisan test --coverage --min=80

# Параллельное выполнение
php artisan test --parallel

# С подробным выводом
php artisan test --verbose
```

---

## Unit Tests (Юнит-тесты)

### Создание теста

```bash
php artisan make:test UserServiceTest --unit
```

### Базовая структура

```php
<?php

namespace Tests\Unit;

use PHPUnit\Framework\TestCase;
use App\Services\Calculator;

class CalculatorTest extends TestCase
{
    /** @test */
    public function it_can_add_two_numbers()
    {
        // Arrange (Подготовка)
        $calculator = new Calculator();
        
        // Act (Действие)
        $result = $calculator->add(2, 3);
        
        // Assert (Проверка)
        $this->assertEquals(5, $result);
    }

    public function test_it_can_subtract()
    {
        $calculator = new Calculator();
        
        $this->assertEquals(2, $calculator->subtract(5, 3));
    }
}
```

### Assertions (Утверждения)

```php
use PHPUnit\Framework\TestCase;

class AssertionsTest extends TestCase
{
    public function test_assertions()
    {
        // Равенство
        $this->assertEquals(5, 2 + 3);
        $this->assertNotEquals(4, 2 + 3);
        
        // Строгое равенство (===)
        $this->assertSame(5, 2 + 3);
        $this->assertNotSame('5', 5);
        
        // Boolean
        $this->assertTrue(true);
        $this->assertFalse(false);
        
        // Null
        $this->assertNull(null);
        $this->assertNotNull('value');
        
        // Empty
        $this->assertEmpty([]);
        $this->assertNotEmpty([1, 2]);
        
        // Массивы
        $this->assertArrayHasKey('name', ['name' => 'John']);
        $this->assertContains('apple', ['apple', 'banana']);
        $this->assertCount(3, [1, 2, 3]);
        
        // Строки
        $this->assertStringContainsString('world', 'hello world');
        $this->assertStringStartsWith('hello', 'hello world');
        $this->assertStringEndsWith('world', 'hello world');
        
        // Regex
        $this->assertMatchesRegularExpression('/^[0-9]+$/', '123');
        
        // Типы
        $this->assertIsArray([1, 2, 3]);
        $this->assertIsString('hello');
        $this->assertIsInt(42);
        $this->assertIsBool(true);
        $this->assertIsFloat(3.14);
        $this->assertIsObject(new stdClass());
        
        // Instance
        $this->assertInstanceOf(User::class, new User());
        
        // Greater/Less
        $this->assertGreaterThan(5, 10);
        $this->assertGreaterThanOrEqual(5, 5);
        $this->assertLessThan(10, 5);
        $this->assertLessThanOrEqual(5, 5);
        
        // Исключения
        $this->expectException(InvalidArgumentException::class);
        $this->expectExceptionMessage('Invalid email');
        throw new InvalidArgumentException('Invalid email');
    }
}
```

### Data Providers (Наборы данных)

```php
class MathTest extends TestCase
{
    /**
     * @test
     * @dataProvider additionProvider
     */
    public function it_can_add_numbers($a, $b, $expected)
    {
        $calculator = new Calculator();
        
        $this->assertEquals($expected, $calculator->add($a, $b));
    }
    
    public static function additionProvider(): array
    {
        return [
            'positive numbers' => [2, 3, 5],
            'negative numbers' => [-2, -3, -5],
            'mixed numbers' => [5, -3, 2],
            'with zero' => [0, 5, 5],
        ];
    }
}
```

---

## Feature Tests (Интеграционные тесты)

### Создание теста

```bash
php artisan make:test UserTest
```

### HTTP тесты

```php
<?php

namespace Tests\Feature;

use Tests\TestCase;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;

class UserTest extends TestCase
{
    use RefreshDatabase; // Очищает БД после каждого теста
    
    public function test_users_list_can_be_accessed()
    {
        // Создать тестовых пользователей
        User::factory()->count(3)->create();
        
        // GET запрос
        $response = $this->get('/users');
        
        // Проверки
        $response->assertStatus(200);
        $response->assertViewIs('users.index');
        $response->assertViewHas('users');
        $response->assertSee('Users List');
    }
    
    public function test_user_can_be_created()
    {
        // POST запрос
        $response = $this->post('/users', [
            'name' => 'John Doe',
            'email' => 'john@example.com',
            'password' => 'password123',
        ]);
        
        // Проверить редирект
        $response->assertRedirect('/users');
        
        // Проверить сохранение в БД
        $this->assertDatabaseHas('users', [
            'name' => 'John Doe',
            'email' => 'john@example.com',
        ]);
        
        // Проверить количество записей
        $this->assertDatabaseCount('users', 1);
    }
    
    public function test_user_validation_errors()
    {
        $response = $this->post('/users', [
            'name' => '', // Пустое имя
            'email' => 'invalid-email',
        ]);
        
        // Проверить ошибки валидации
        $response->assertSessionHasErrors(['name', 'email']);
        
        // Проверить конкретную ошибку
        $response->assertSessionHasErrors([
            'email' => 'The email field must be a valid email address.',
        ]);
    }
    
    public function test_user_update()
    {
        $user = User::factory()->create();
        
        $response = $this->put("/users/{$user->id}", [
            'name' => 'Updated Name',
            'email' => $user->email,
        ]);
        
        $response->assertRedirect("/users/{$user->id}");
        
        // Проверить что старого значения нет
        $this->assertDatabaseMissing('users', [
            'name' => $user->name,
        ]);
        
        // Проверить новое значение
        $this->assertDatabaseHas('users', [
            'id' => $user->id,
            'name' => 'Updated Name',
        ]);
        
        // Или через модель
        $this->assertEquals('Updated Name', $user->fresh()->name);
    }
    
    public function test_user_deletion()
    {
        $user = User::factory()->create();
        
        $response = $this->delete("/users/{$user->id}");
        
        $response->assertRedirect('/users');
        
        // Soft delete
        $this->assertSoftDeleted('users', ['id' => $user->id]);
        
        // Hard delete
        // $this->assertDatabaseMissing('users', ['id' => $user->id]);
        // $this->assertModelMissing($user);
    }
}
```

### HTTP Assertions

```php
class HttpAssertionsTest extends TestCase
{
    public function test_http_assertions()
    {
        $response = $this->get('/users');
        
        // Status codes
        $response->assertOk(); // 200
        $response->assertCreated(); // 201
        $response->assertNoContent(); // 204
        $response->assertNotFound(); // 404
        $response->assertForbidden(); // 403
        $response->assertUnauthorized(); // 401
        $response->assertStatus(200);
        $response->assertSuccessful(); // 2xx
        
        // Redirects
        $response->assertRedirect('/users');
        $response->assertRedirectToRoute('users.index');
        $response->assertRedirectContains('/users');
        
        // Headers
        $response->assertHeader('Content-Type', 'application/json');
        $response->assertHeaderMissing('X-Custom-Header');
        
        // Cookies
        $response->assertCookie('name', 'value');
        $response->assertCookieExpired('name');
        $response->assertCookieNotExpired('name');
        $response->assertCookieMissing('name');
        
        // View
        $response->assertViewIs('users.index');
        $response->assertViewHas('users');
        $response->assertViewHas('count', 10);
        $response->assertViewMissing('admin');
        
        // Content
        $response->assertSee('Hello World');
        $response->assertSeeInOrder(['First', 'Second']);
        $response->assertSeeText('Plain text'); // Без HTML тегов
        $response->assertDontSee('Secret');
        
        // JSON
        $response->assertJson(['name' => 'John']);
        $response->assertJsonStructure([
            'data' => [
                '*' => ['id', 'name', 'email']
            ]
        ]);
        $response->assertJsonPath('data.0.name', 'John');
        $response->assertJsonCount(3, 'data');
        $response->assertJsonFragment(['name' => 'John']);
        $response->assertJsonMissing(['password' => 'secret']);
        
        // Session
        $response->assertSessionHas('status', 'success');
        $response->assertSessionMissing('error');
        $response->assertSessionHasErrors(['email', 'password']);
        $response->assertSessionDoesntHaveErrors(['name']);
    }
}
```

---

## Authentication Tests (Тесты аутентификации)

```php
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;

class AuthTest extends TestCase
{
    use RefreshDatabase;
    
    public function test_user_can_login()
    {
        $user = User::factory()->create([
            'email' => 'test@example.com',
            'password' => bcrypt('password'),
        ]);
        
        $response = $this->post('/login', [
            'email' => 'test@example.com',
            'password' => 'password',
        ]);
        
        $response->assertRedirect('/dashboard');
        $this->assertAuthenticatedAs($user);
    }
    
    public function test_user_cannot_login_with_wrong_password()
    {
        $user = User::factory()->create();
        
        $response = $this->post('/login', [
            'email' => $user->email,
            'password' => 'wrong-password',
        ]);
        
        $response->assertSessionHasErrors();
        $this->assertGuest();
    }
    
    public function test_authenticated_user_can_access_dashboard()
    {
        $user = User::factory()->create();
        
        // Аутентифицировать пользователя для теста
        $response = $this->actingAs($user)->get('/dashboard');
        
        $response->assertOk();
        $response->assertSee('Dashboard');
    }
    
    public function test_guest_cannot_access_dashboard()
    {
        $response = $this->get('/dashboard');
        
        $response->assertRedirect('/login');
    }
    
    public function test_user_can_logout()
    {
        $user = User::factory()->create();
        
        $response = $this->actingAs($user)->post('/logout');
        
        $response->assertRedirect('/');
        $this->assertGuest();
    }
}
```

---

## Database Testing

### RefreshDatabase vs DatabaseTransactions

```php
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Foundation\Testing\DatabaseTransactions;

// RefreshDatabase - мигрирует БД заново перед ВСЕМИ тестами
class UserTest extends TestCase
{
    use RefreshDatabase;
    
    // БД пустая в начале каждого теста
}

// DatabaseTransactions - откатывает транзакцию после КАЖДОГО теста
class OrderTest extends TestCase
{
    use DatabaseTransactions;
    
    // Быстрее, но не работает с тестами самих транзакций
}
```

### Database Assertions

```php
public function test_database_assertions()
{
    $user = User::factory()->create(['name' => 'John']);
    
    // Есть в БД
    $this->assertDatabaseHas('users', [
        'name' => 'John',
        'email' => $user->email,
    ]);
    
    // Нет в БД
    $this->assertDatabaseMissing('users', [
        'name' => 'Jane',
    ]);
    
    // Количество записей
    $this->assertDatabaseCount('users', 1);
    
    // Soft deleted
    $user->delete();
    $this->assertSoftDeleted('users', ['id' => $user->id]);
    
    // Model существует
    $this->assertModelExists($user);
    
    // Model удален
    $user->forceDelete();
    $this->assertModelMissing($user);
}
```

### Фабрики в тестах

```php
public function test_factories()
{
    // Создать модель без сохранения
    $user = User::factory()->make();
    $this->assertNull($user->id);
    
    // Создать и сохранить
    $user = User::factory()->create();
    $this->assertNotNull($user->id);
    
    // С атрибутами
    $user = User::factory()->create(['name' => 'John']);
    
    // Множественные
    $users = User::factory()->count(5)->create();
    $this->assertCount(5, $users);
    
    // Со state
    $admin = User::factory()->admin()->create();
    
    // С отношениями
    $user = User::factory()
        ->has(Post::factory()->count(3))
        ->create();
    
    $this->assertCount(3, $user->posts);
}
```

---

## Mocking & Faking

### Fake (Laravel Fakes)

```php
use Illuminate\Support\Facades\Mail;
use Illuminate\Support\Facades\Queue;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Facades\Event;
use Illuminate\Support\Facades\Notification;

class FakingTest extends TestCase
{
    public function test_mail_fake()
    {
        Mail::fake();
        
        // Отправка письма
        Mail::to('test@example.com')->send(new WelcomeMail());
        
        // Проверки
        Mail::assertSent(WelcomeMail::class);
        
        Mail::assertSent(WelcomeMail::class, function ($mail) {
            return $mail->hasTo('test@example.com');
        });
        
        Mail::assertSent(WelcomeMail::class, 1); // Ровно 1 раз
        
        Mail::assertNotSent(AdminNotification::class);
        
        Mail::assertNothingSent();
    }
    
    public function test_queue_fake()
    {
        Queue::fake();
        
        ProcessPodcast::dispatch();
        
        Queue::assertPushed(ProcessPodcast::class);
        
        Queue::assertPushed(ProcessPodcast::class, function ($job) {
            return $job->podcast->id === 1;
        });
        
        Queue::assertPushedOn('emails', ProcessPodcast::class);
        
        Queue::assertNotPushed(AnotherJob::class);
        
        Queue::assertNothingPushed();
    }
    
    public function test_storage_fake()
    {
        Storage::fake('public');
        
        // Загрузка файла
        $response = $this->post('/upload', [
            'file' => UploadedFile::fake()->image('photo.jpg'),
        ]);
        
        // Проверки
        Storage::disk('public')->assertExists('photos/photo.jpg');
        Storage::disk('public')->assertMissing('photos/missing.jpg');
    }
    
    public function test_event_fake()
    {
        Event::fake();
        
        // Только конкретные события
        Event::fake([OrderShipped::class]);
        
        // Создание заказа (вызывает событие)
        $order = Order::create([...]);
        
        Event::assertDispatched(OrderShipped::class);
        
        Event::assertDispatched(function (OrderShipped $event) use ($order) {
            return $event->order->id === $order->id;
        });
        
        Event::assertNotDispatched(OrderCancelled::class);
    }
    
    public function test_notification_fake()
    {
        Notification::fake();
        
        $user = User::factory()->create();
        $user->notify(new InvoicePaid());
        
        Notification::assertSentTo($user, InvoicePaid::class);
        
        Notification::assertSentTo(
            [$user], 
            InvoicePaid::class,
            function ($notification) {
                return $notification->amount === 100;
            }
        );
        
        Notification::assertNotSentTo($user, AnotherNotification::class);
    }
}
```

### Mock (PHPUnit Mocks)

```php
class MockingTest extends TestCase
{
    public function test_mocking_dependencies()
    {
        // Mock внешнего сервиса
        $paymentGateway = $this->createMock(PaymentGateway::class);
        
        // Настройка ожиданий
        $paymentGateway
            ->expects($this->once())
            ->method('charge')
            ->with(100, 'USD')
            ->willReturn(true);
        
        // Inject mock в контроллер/сервис
        $service = new OrderService($paymentGateway);
        
        $result = $service->processOrder(100);
        
        $this->assertTrue($result);
    }
    
    public function test_mockery()
    {
        // Mockery (более удобный синтаксис)
        $mock = Mockery::mock(PaymentGateway::class);
        
        $mock->shouldReceive('charge')
            ->once()
            ->with(100, 'USD')
            ->andReturn(true);
        
        $this->app->instance(PaymentGateway::class, $mock);
        
        $service = app(OrderService::class);
        $result = $service->processOrder(100);
        
        $this->assertTrue($result);
    }
}
```

### Partial Mocks

```php
public function test_partial_mock()
{
    // Мокаем только один метод, остальные работают реально
    $service = $this->partialMock(UserService::class, function ($mock) {
        $mock->shouldReceive('sendEmail')
            ->once()
            ->andReturn(true);
    });
    
    // sendEmail - замокирован, остальные методы работают
    $result = $service->createUser([...]);
}
```

---

## API Testing

```php
class ApiTest extends TestCase
{
    use RefreshDatabase;
    
    public function test_get_users_api()
    {
        User::factory()->count(3)->create();
        
        $response = $this->getJson('/api/users');
        
        $response->assertOk();
        $response->assertJsonCount(3, 'data');
        $response->assertJsonStructure([
            'data' => [
                '*' => ['id', 'name', 'email', 'created_at']
            ]
        ]);
    }
    
    public function test_create_user_api()
    {
        $response = $this->postJson('/api/users', [
            'name' => 'John Doe',
            'email' => 'john@example.com',
            'password' => 'password123',
        ]);
        
        $response->assertCreated(); // 201
        $response->assertJson([
            'data' => [
                'name' => 'John Doe',
                'email' => 'john@example.com',
            ]
        ]);
        
        $this->assertDatabaseHas('users', [
            'email' => 'john@example.com',
        ]);
    }
    
    public function test_authenticated_api_request()
    {
        $user = User::factory()->create();
        
        // С Bearer токеном
        $response = $this->withToken('api-token')
            ->getJson('/api/user');
        
        // Или с Sanctum
        $response = $this->actingAs($user, 'sanctum')
            ->getJson('/api/user');
        
        $response->assertOk();
    }
    
    public function test_validation_error_response()
    {
        $response = $this->postJson('/api/users', [
            'name' => '',
            'email' => 'invalid-email',
        ]);
        
        $response->assertUnprocessable(); // 422
        $response->assertJsonValidationErrors(['name', 'email']);
        $response->assertJsonFragment([
            'message' => 'The name field is required.',
        ]);
    }
}
```

---

## Testing Console Commands

```php
class CommandTest extends TestCase
{
    public function test_console_command()
    {
        $this->artisan('emails:send')
            ->expectsOutput('Emails sent!')
            ->assertExitCode(0);
    }
    
    public function test_command_with_confirmation()
    {
        $this->artisan('db:clear')
            ->expectsConfirmation('Are you sure?', 'yes')
            ->expectsOutput('Database cleared')
            ->assertExitCode(0);
    }
    
    public function test_command_with_arguments()
    {
        $this->artisan('user:create', ['name' => 'John'])
            ->assertExitCode(0);
    }
    
    public function test_command_with_options()
    {
        $this->artisan('report:generate', ['--format' => 'pdf'])
            ->assertExitCode(0);
    }
}
```

---

## Testing Jobs

```php
class JobTest extends TestCase
{
    use RefreshDatabase;
    
    public function test_job_is_dispatched()
    {
        Queue::fake();
        
        $user = User::factory()->create();
        
        SendWelcomeEmail::dispatch($user);
        
        Queue::assertPushed(SendWelcomeEmail::class, function ($job) use ($user) {
            return $job->user->id === $user->id;
        });
    }
    
    public function test_job_execution()
    {
        Mail::fake();
        
        $user = User::factory()->create();
        
        // Выполнить job напрямую
        (new SendWelcomeEmail($user))->handle();
        
        Mail::assertSent(WelcomeMail::class, function ($mail) use ($user) {
            return $mail->hasTo($user->email);
        });
    }
}
```

---

## Testing Events & Listeners

```php
class EventTest extends TestCase
{
    public function test_event_is_dispatched()
    {
        Event::fake();
        
        $order = Order::create([...]);
        
        Event::assertDispatched(OrderShipped::class);
    }
    
    public function test_listener_handles_event()
    {
        Mail::fake();
        
        $event = new OrderShipped(Order::factory()->create());
        $listener = new SendShipmentNotification();
        
        $listener->handle($event);
        
        Mail::assertSent(OrderShippedMail::class);
    }
}
```

---

## Best Practices

### 1. Arrange-Act-Assert Pattern

```php
public function test_user_creation()
{
    // Arrange - подготовка
    $data = [
        'name' => 'John',
        'email' => 'john@example.com',
    ];
    
    // Act - действие
    $user = User::create($data);
    
    // Assert - проверка
    $this->assertEquals('John', $user->name);
    $this->assertDatabaseHas('users', $data);
}
```

### 2. Тестируй поведение, а не реализацию

```php
// ❌ ПЛОХО - тестирует реализацию
public function test_repository_uses_eloquent()
{
    $repo = new UserRepository();
    $this->assertInstanceOf(EloquentBuilder::class, $repo->query());
}

// ✅ ХОРОШО - тестирует поведение
public function test_repository_returns_active_users()
{
    User::factory()->create(['status' => 'active']);
    User::factory()->create(['status' => 'inactive']);
    
    $repo = new UserRepository();
    $users = $repo->getActiveUsers();
    
    $this->assertCount(1, $users);
    $this->assertTrue($users->first()->status === 'active');
}
```

### 3. Один тест = одна проверка

```php
// ❌ ПЛОХО - несколько проверок
public function test_user_crud()
{
    $user = User::create([...]);
    $this->assertDatabaseHas('users', [...]);
    
    $user->update(['name' => 'Updated']);
    $this->assertEquals('Updated', $user->name);
    
    $user->delete();
    $this->assertSoftDeleted('users', ['id' => $user->id]);
}

// ✅ ХОРОШО - отдельные тесты
public function test_user_can_be_created() { ... }
public function test_user_can_be_updated() { ... }
public function test_user_can_be_deleted() { ... }
```

### 4. Используй фабрики вместо создания вручную

```php
// ❌ ПЛОХО
$user = User::create([
    'name' => 'John',
    'email' => 'john@example.com',
    'password' => bcrypt('password'),
    'email_verified_at' => now(),
]);

// ✅ ХОРОШО
$user = User::factory()->create();
$admin = User::factory()->admin()->create();
```

### 5. Изоляция тестов

```php
// Каждый тест независим
use RefreshDatabase; // БД сбрасывается

// Не используй статические переменные
// Не полагайся на порядок выполнения тестов
```

### 6. Тестируй граничные случаи

```php
public function test_age_validation()
{
    // Минимальное значение
    $this->postJson('/users', ['age' => 18])->assertOk();
    
    // Максимальное значение
    $this->postJson('/users', ['age' => 120])->assertOk();
    
    // Ниже минимума
    $this->postJson('/users', ['age' => 17])->assertUnprocessable();
    
    // Выше максимума
    $this->postJson('/users', ['age' => 121])->assertUnprocessable();
    
    // Пограничные случаи
    $this->postJson('/users', ['age' => null])->assertUnprocessable();
    $this->postJson('/users', ['age' => 'abc'])->assertUnprocessable();
}
```

### 7. Описательные имена тестов

```php
// ❌ ПЛОХО
public function test_1() { }
public function test_user() { }

// ✅ ХОРОШО
public function test_user_can_login_with_valid_credentials() { }
public function test_guest_cannot_access_admin_panel() { }
public function test_order_total_is_calculated_correctly() { }
```

### 8. Setup и Teardown

```php
class UserTest extends TestCase
{
    protected $user;
    
    protected function setUp(): void
    {
        parent::setUp();
        
        // Выполнится перед КАЖДЫМ тестом
        $this->user = User::factory()->create();
    }
    
    protected function tearDown(): void
    {
        // Выполнится после КАЖДОГО теста
        // Очистка, закрытие соединений и т.д.
        
        parent::tearDown();
    }
}
```

---

## Покрытие кода

```bash
# Генерация отчета о покрытии
php artisan test --coverage

# Минимальное покрытие (упадет если меньше)
php artisan test --coverage --min=80

# HTML отчет (требует Xdebug/PCOV)
vendor/bin/phpunit --coverage-html coverage
```

**Цель:** Стремиться к 80%+ покрытия критичного кода (модели, сервисы, контроллеры).

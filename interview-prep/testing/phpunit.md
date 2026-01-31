# PHPUnit - тестирование в PHP

## Основы PHPUnit

### Установка
```bash
composer require --dev phpunit/phpunit
./vendor/bin/phpunit --version
```

### Конфигурация phpunit.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit bootstrap="vendor/autoload.php"
         colors="true"
         stopOnFailure="false">
    <testsuites>
        <testsuite name="Unit">
            <directory>tests/Unit</directory>
        </testsuite>
        <testsuite name="Feature">
            <directory>tests/Feature</directory>
        </testsuite>
    </testsuites>
    <coverage>
        <include>
            <directory>src</directory>
        </include>
    </coverage>
</phpunit>
```

### Структура теста
```php
use PHPUnit\Framework\TestCase;

class CalculatorTest extends TestCase
{
    protected function setUp(): void
    {
        // Выполняется перед каждым тестом
    }
    
    protected function tearDown(): void
    {
        // Выполняется после каждого теста
    }
    
    public function testAddition(): void
    {
        $calculator = new Calculator();
        $result = $calculator->add(2, 3);
        
        $this->assertEquals(5, $result);
    }
}
```

## Assertions (проверки)

### Базовые
```php
// Равенство
$this->assertEquals($expected, $actual);
$this->assertSame($expected, $actual); // строгое ===
$this->assertNotEquals($expected, $actual);

// Типы
$this->assertIsInt($value);
$this->assertIsString($value);
$this->assertIsArray($value);
$this->assertIsBool($value);
$this->assertInstanceOf(User::class, $object);

// Boolean
$this->assertTrue($condition);
$this->assertFalse($condition);
$this->assertNull($value);
$this->assertNotNull($value);

// Arrays
$this->assertArrayHasKey('key', $array);
$this->assertContains('value', $array);
$this->assertCount(5, $array);
$this->assertEmpty($array);

// Strings
$this->assertStringContainsString('needle', $haystack);
$this->assertStringStartsWith('prefix', $string);
$this->assertMatchesRegularExpression('/pattern/', $string);

// Exceptions
$this->expectException(InvalidArgumentException::class);
$this->expectExceptionMessage('Error message');
```

### Assertions для объектов
```php
$this->assertObjectHasAttribute('property', $object);
$this->assertClassHasAttribute('property', ClassName::class);
$this->assertJsonStringEqualsJsonString($json1, $json2);
```

## Data Providers

### Простой provider
```php
/**
 * @dataProvider additionProvider
 */
public function testAdd(int $a, int $b, int $expected): void
{
    $this->assertEquals($expected, $a + $b);
}

public static function additionProvider(): array
{
    return [
        [1, 1, 2],
        [2, 3, 5],
        [5, 5, 10],
    ];
}
```

### Named datasets
```php
public static function additionProvider(): array
{
    return [
        'positive numbers' => [1, 1, 2],
        'negative numbers' => [-1, -1, -2],
        'mixed numbers' => [1, -1, 0],
    ];
}
```

### Generator provider
```php
public static function additionProvider(): Generator
{
    yield [1, 1, 2];
    yield [2, 3, 5];
    yield from self::moreTestCases();
}
```

## Mocking

### Создание mock
```php
$mock = $this->createMock(UserRepository::class);

// Настройка ожиданий
$mock->expects($this->once())
     ->method('find')
     ->with(123)
     ->willReturn(new User(['id' => 123]));

// Использование
$service = new UserService($mock);
$user = $service->getUser(123);
```

### Mock builder
```php
$mock = $this->getMockBuilder(UserRepository::class)
             ->disableOriginalConstructor()
             ->onlyMethods(['find', 'save'])
             ->getMock();
```

### Stub (заглушка)
```php
$stub = $this->createStub(UserRepository::class);
$stub->method('find')
     ->willReturn(new User());

// Всегда возвращает одно и то же, без проверок вызовов
```

### Expectations
```php
// Количество вызовов
$mock->expects($this->once())
$mock->expects($this->exactly(3))
$mock->expects($this->never())
$mock->expects($this->atLeastOnce())

// Параметры
->with($this->equalTo(123))
->with($this->isInstanceOf(User::class))
->with($this->anything())
->with($this->callback(fn($arg) => $arg > 0))

// Возвращаемые значения
->willReturn($value)
->willReturnSelf()
->willReturnArgument(0)
->willReturnCallback(fn($arg) => $arg * 2)
->willThrowException(new Exception())
```

## Test Doubles

### Dummy
```php
// Объект-заглушка, не используется в тесте
$dummy = $this->createStub(Logger::class);
```

### Stub
```php
// Предоставляет заранее определённые ответы
$stub = $this->createStub(Database::class);
$stub->method('query')->willReturn([]);
```

### Spy
```php
// Записывает информацию о вызовах
$spy = $this->createMock(EventDispatcher::class);
$spy->expects($this->once())->method('dispatch');
```

### Mock
```php
// Проверяет взаимодействие
$mock = $this->createMock(UserRepository::class);
$mock->expects($this->once())
     ->method('save')
     ->with($this->isInstanceOf(User::class));
```

### Fake
```php
// Упрощённая реализация
class FakeUserRepository implements UserRepositoryInterface
{
    private array $users = [];
    
    public function save(User $user): void
    {
        $this->users[$user->id] = $user;
    }
    
    public function find(int $id): ?User
    {
        return $this->users[$id] ?? null;
    }
}
```

## Code Coverage

### Запуск с покрытием
```bash
./vendor/bin/phpunit --coverage-html coverage
./vendor/bin/phpunit --coverage-text
./vendor/bin/phpunit --coverage-clover coverage.xml
```

### Annotations
```php
/**
 * @covers Calculator::add
 * @uses Calculator::validate
 */
public function testAdd(): void
{
    // ...
}

/**
 * @coversNothing
 */
public function testIntegration(): void
{
    // Не учитывается в coverage
}
```

## Test Organization

### setUp / tearDown
```php
class DatabaseTest extends TestCase
{
    private PDO $pdo;
    
    protected function setUp(): void
    {
        $this->pdo = new PDO('sqlite::memory:');
        $this->pdo->exec('CREATE TABLE users (id INT, name TEXT)');
    }
    
    protected function tearDown(): void
    {
        $this->pdo = null;
    }
    
    public static function setUpBeforeClass(): void
    {
        // Выполняется один раз перед всеми тестами класса
    }
    
    public static function tearDownAfterClass(): void
    {
        // Выполняется один раз после всех тестов класса
    }
}
```

### Группировка тестов
```php
/**
 * @group database
 * @group slow
 */
class UserRepositoryTest extends TestCase
{
    // ...
}
```

```bash
# Запуск группы
./vendor/bin/phpunit --group database
./vendor/bin/phpunit --exclude-group slow
```

## Database Testing

### Транзакции
```php
use Illuminate\Foundation\Testing\DatabaseTransactions;

class UserTest extends TestCase
{
    use DatabaseTransactions;
    
    public function testCreateUser(): void
    {
        $user = User::create(['name' => 'John']);
        $this->assertDatabaseHas('users', ['name' => 'John']);
    }
    // Автоматический rollback после теста
}
```

### Factories и seeders
```php
public function testUserCreation(): void
{
    $user = User::factory()->create([
        'email' => 'test@example.com'
    ]);
    
    $this->assertEquals('test@example.com', $user->email);
}
```

## Best Practices

### Naming conventions
```php
// test + MethodName + Scenario + ExpectedResult
public function testAddWithPositiveNumbersReturnsSum(): void
public function testDivideByZeroThrowsException(): void
public function testFindUserByIdReturnsNull(): void
```

### AAA Pattern (Arrange-Act-Assert)
```php
public function testUserCanBeBanned(): void
{
    // Arrange
    $user = new User(['status' => 'active']);
    
    // Act
    $user->ban();
    
    // Assert
    $this->assertEquals('banned', $user->status);
}
```

### Один assert на тест (когда возможно)
```php
// Хорошо
public function testUserNameIsSet(): void
{
    $user = new User(['name' => 'John']);
    $this->assertEquals('John', $user->name);
}

// Плохо (множественные проверки разных вещей)
public function testUser(): void
{
    $user = new User(['name' => 'John', 'email' => 'john@example.com']);
    $this->assertEquals('John', $user->name);
    $this->assertEquals('john@example.com', $user->email);
    $this->assertTrue($user->isActive());
}
```

### Test Isolation
- Тесты не должны зависеть друг от друга
- Можно запускать в любом порядке
- Каждый тест очищает за собой

### Fast tests
- Избегай медленных операций (sleep, реальные API)
- Используй in-memory databases
- Mock внешние зависимости
- Parallel execution где возможно

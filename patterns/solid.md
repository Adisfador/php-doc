# SOLID Принципы

SOLID - это пять принципов объектно-ориентированного программирования и дизайна, которые делают код более понятным, гибким и поддерживаемым.

---

## S - Single Responsibility Principle (Принцип единственной ответственности)

**Определение:** Класс должен иметь только одну причину для изменения. Один класс = одна ответственность.

###❌ Нарушение SRP

```php
class User
{
    public function __construct(
        private string $name,
        private string $email
    ) {}
    
    // Ответственность 1: Управление данными пользователя
    public function getName(): string
    {
        return $this->name;
    }
    
    // Ответственность 2: Валидация
    public function isValid(): bool
    {
        return filter_var($this->email, FILTER_VALIDATE_EMAIL);
    }
    
    // Ответственность 3: Сохранение в БД
    public function save(): void
    {
        $db = new PDO(/*...*/);
        $stmt = $db->prepare('INSERT INTO users ...');
        $stmt->execute([
            'name' => $this->name,
            'email' => $this->email
        ]);
    }
    
    // Ответственность 4: Отправка email
    public function sendWelcomeEmail(): void
    {
        mail($this->email, 'Welcome!', 'Welcome to our site!');
    }
}
```

**Проблемы:**
- Изменения в логике email затронут класс User
- Изменения в БД затронут класс User
- Сложно тестировать
- Нарушает принцип единственной ответственности

### ✅ Соблюдение SRP

```php
// Только данные пользователя
class User
{
    public function __construct(
        private string $name,
        private string $email
    ) {}
    
    public function getName(): string
    {
        return $this->name;
    }
    
    public function getEmail(): string
    {
        return $this->email;
    }
}

// Валидация
class UserValidator
{
    public function validate(User $user): bool
    {
        return filter_var($user->getEmail(), FILTER_VALIDATE_EMAIL);
    }
}

// Сохранение
class UserRepository
{
    public function __construct(
        private PDO $db
    ) {}
    
    public function save(User $user): void
    {
        $stmt = $this->db->prepare('INSERT INTO users (name, email) VALUES (?, ?)');
        $stmt->execute([$user->getName(), $user->getEmail()]);
    }
}

// Отправка email
class UserMailer
{
    public function sendWelcomeEmail(User $user): void
    {
        mail($user->getEmail(), 'Welcome!', 'Welcome to our site!');
    }
}

// Использование
$user = new User('John', 'john@example.com');

$validator = new UserValidator();
if ($validator->validate($user)) {
    $repository = new UserRepository($pdo);
    $repository->save($user);
    
    $mailer = new UserMailer();
    $mailer->sendWelcomeEmail($user);
}
```

---

## O - Open/Closed Principle (Принцип открытости/закрытости)

**Определение:** Классы должны быть открыты для расширения, но закрыты для модификации.

### ❌ Нарушение OCP

```php
class PaymentProcessor
{
    public function process(string $type, float $amount): void
    {
        if ($type === 'credit_card') {
            echo "Processing credit card payment: $amount\n";
            // Логика для кредитной карты
        } elseif ($type === 'paypal') {
            echo "Processing PayPal payment: $amount\n";
            // Логика для PayPal
        } elseif ($type === 'bitcoin') {
            echo "Processing Bitcoin payment: $amount\n";
            // Логика для Bitcoin
        }
        // При добавлении нового способа оплаты нужно МОДИФИЦИРОВАТЬ этот класс!
    }
}
```

### ✅ Соблюдение OCP

```php
interface PaymentMethod
{
    public function process(float $amount): void;
}

class CreditCardPayment implements PaymentMethod
{
    public function process(float $amount): void
    {
        echo "Processing credit card payment: $amount\n";
    }
}

class PayPalPayment implements PaymentMethod
{
    public function process(float $amount): void
    {
        echo "Processing PayPal payment: $amount\n";
    }
}

class BitcoinPayment implements PaymentMethod
{
    public function process(float $amount): void
    {
        echo "Processing Bitcoin payment: $amount\n";
    }
}

class PaymentProcessor
{
    public function process(PaymentMethod $paymentMethod, float $amount): void
    {
        $paymentMethod->process($amount);
    }
}

// Добавление нового способа оплаты БЕЗ изменения существующего кода
class ApplePayPayment implements PaymentMethod
{
    public function process(float $amount): void
    {
        echo "Processing Apple Pay payment: $amount\n";
    }
}

// Использование
$processor = new PaymentProcessor();
$processor->process(new CreditCardPayment(), 100);
$processor->process(new ApplePayPayment(), 200);
```

---

## L - Liskov Substitution Principle (Принцип подстановки Барбары Лисков)

**Определение:** Объекты подклассов должны быть взаимозаменяемы с объектами базового класса без изменения корректности программы.

### ❌ Нарушение LSP

```php
class Rectangle
{
    protected int $width;
    protected int $height;
    
    public function setWidth(int $width): void
    {
        $this->width = $width;
    }
    
    public function setHeight(int $height): void
    {
        $this->height = $height;
    }
    
    public function getArea(): int
    {
        return $this->width * $this->height;
    }
}

class Square extends Rectangle
{
    // Квадрат: ширина = высота
    public function setWidth(int $width): void
    {
        $this->width = $width;
        $this->height = $width; // Нарушение ожидаемого поведения!
    }
    
    public function setHeight(int $height): void
    {
        $this->width = $height;
        $this->height = $height; // Нарушение ожидаемого поведения!
    }
}

// Тест
function testRectangle(Rectangle $rectangle): void
{
    $rectangle->setWidth(5);
    $rectangle->setHeight(10);
    
    // Ожидаем: 5 * 10 = 50
    assert($rectangle->getArea() === 50); // FAIL для Square!
}

$rect = new Rectangle();
testRectangle($rect); // OK: 50

$square = new Square();
testRectangle($square); // FAIL: 100 (10 * 10)
```

### ✅ Соблюдение LSP

```php
interface Shape
{
    public function getArea(): int;
}

class Rectangle implements Shape
{
    public function __construct(
        private int $width,
        private int $height
    ) {}
    
    public function getArea(): int
    {
        return $this->width * $this->height;
    }
}

class Square implements Shape
{
    public function __construct(
        private int $side
    ) {}
    
    public function getArea(): int
    {
        return $this->side * $this->side;
    }
}

// Оба класса корректно реализуют Shape
function calculateArea(Shape $shape): int
{
    return $shape->getArea();
}

calculateArea(new Rectangle(5, 10)); // 50
calculateArea(new Square(10));       // 100
```

### Еще пример нарушения LSP

```php
// ❌ Плохо
class Bird
{
    public function fly(): void
    {
        echo "Flying...\n";
    }
}

class Penguin extends Bird
{
    public function fly(): void
    {
        throw new Exception("Penguins can't fly!"); // Нарушение LSP
    }
}

// ✅ Хорошо
interface Bird
{
    public function move(): void;
}

interface FlyingBird extends Bird
{
    public function fly(): void;
}

class Sparrow implements FlyingBird
{
    public function move(): void
    {
        $this->fly();
    }
    
    public function fly(): void
    {
        echo "Flying...\n";
    }
}

class Penguin implements Bird
{
    public function move(): void
    {
        echo "Swimming...\n";
    }
}
```

---

## I - Interface Segregation Principle (Принцип разделения интерфейса)

**Определение:** Клиенты не должны зависеть от интерфейсов, которые они не используют. Лучше много маленьких интерфейсов, чем один большой.

### ❌ Нарушение ISP

```php
interface Worker
{
    public function work(): void;
    public function eat(): void;
    public function sleep(): void;
}

class Human implements Worker
{
    public function work(): void
    {
        echo "Working...\n";
    }
    
    public function eat(): void
    {
        echo "Eating...\n";
    }
    
    public function sleep(): void
    {
        echo "Sleeping...\n";
    }
}

class Robot implements Worker
{
    public function work(): void
    {
        echo "Working...\n";
    }
    
    // Робот не ест и не спит!
    public function eat(): void
    {
        throw new Exception("Robots don't eat!");
    }
    
    public function sleep(): void
    {
        throw new Exception("Robots don't sleep!");
    }
}
```

### ✅ Соблюдение ISP

```php
interface Workable
{
    public function work(): void;
}

interface Eatable
{
    public function eat(): void;
}

interface Sleepable
{
    public function sleep(): void;
}

class Human implements Workable, Eatable, Sleepable
{
    public function work(): void
    {
        echo "Working...\n";
    }
    
    public function eat(): void
    {
        echo "Eating...\n";
    }
    
    public function sleep(): void
    {
        echo "Sleeping...\n";
    }
}

class Robot implements Workable
{
    public function work(): void
    {
        echo "Working...\n";
    }
}

// Функция зависит только от Workable
function manage(Workable $worker): void
{
    $worker->work();
}

manage(new Human());  // OK
manage(new Robot());  // OK
```

### Laravel пример

```php
// ❌ Плохо: жирный интерфейс
interface RepositoryInterface
{
    public function find(int $id);
    public function findAll();
    public function create(array $data);
    public function update(int $id, array $data);
    public function delete(int $id);
    public function search(string $query);
    public function paginate(int $perPage);
    public function export();
    public function import(string $file);
}

// ReadOnlyRepository вынужден реализовывать ненужные методы!

// ✅ Хорошо: разделенные интерфейсы
interface Readable
{
    public function find(int $id);
    public function findAll();
}

interface Writable
{
    public function create(array $data);
    public function update(int $id, array $data);
    public function delete(int $id);
}

interface Searchable
{
    public function search(string $query);
    public function paginate(int $perPage);
}

// Репозитории реализуют только нужные интерфейсы
class ReadOnlyUserRepository implements Readable
{
    // Только чтение
}

class UserRepository implements Readable, Writable, Searchable
{
    // Полная функциональность
}
```

---

## D - Dependency Inversion Principle (Принцип инверсии зависимостей)

**Определение:**
1. Модули верхнего уровня не должны зависеть от модулей нижнего уровня. Оба должны зависеть от абстракций.
2. Абстракции не должны зависеть от деталей. Детали должны зависеть от абстракций.

### ❌ Нарушение DIP

```php
// Низкоуровневый модуль
class MySQLDatabase
{
    public function connect(): void
    {
        echo "Connecting to MySQL...\n";
    }
    
    public function query(string $sql): array
    {
        echo "Executing: $sql\n";
        return [];
    }
}

// Высокоуровневый модуль зависит от конкретной реализации
class UserService
{
    private MySQLDatabase $database;
    
    public function __construct()
    {
        $this->database = new MySQLDatabase(); // Жесткая зависимость!
    }
    
    public function getUser(int $id): array
    {
        return $this->database->query("SELECT * FROM users WHERE id = $id");
    }
}

// Проблемы:
// - Нельзя легко заменить MySQL на PostgreSQL
// - Сложно тестировать (нужна реальная БД)
// - Тесная связь между модулями
```

### ✅ Соблюдение DIP

```php
// Абстракция
interface DatabaseInterface
{
    public function connect(): void;
    public function query(string $sql): array;
}

// Низкоуровневые модули зависят от абстракции
class MySQLDatabase implements DatabaseInterface
{
    public function connect(): void
    {
        echo "Connecting to MySQL...\n";
    }
    
    public function query(string $sql): array
    {
        echo "Executing MySQL: $sql\n";
        return [];
    }
}

class PostgreSQLDatabase implements DatabaseInterface
{
    public function connect(): void
    {
        echo "Connecting to PostgreSQL...\n";
    }
    
    public function query(string $sql): array
    {
        echo "Executing PostgreSQL: $sql\n";
        return [];
    }
}

// Высокоуровневый модуль зависит от абстракции
class UserService
{
    public function __construct(
        private DatabaseInterface $database
    ) {}
    
    public function getUser(int $id): array
    {
        return $this->database->query("SELECT * FROM users WHERE id = $id");
    }
}

// Использование
$mysqlService = new UserService(new MySQLDatabase());
$postgresService = new UserService(new PostgreSQLDatabase());

// Для тестов
class MockDatabase implements DatabaseInterface
{
    public function connect(): void {}
    
    public function query(string $sql): array
    {
        return ['id' => 1, 'name' => 'Test User'];
    }
}

$testService = new UserService(new MockDatabase());
```

### Laravel пример

```php
// ❌ Плохо
class OrderController extends Controller
{
    public function store(Request $request)
    {
        // Прямое использование Eloquent
        $order = Order::create($request->all());
        
        // Прямое использование конкретного сервиса
        $paymentService = new StripePaymentService();
        $paymentService->charge($order->total);
        
        // Прямая отправка email
        Mail::to($order->user)->send(new OrderConfirmation($order));
    }
}

// ✅ Хорошо
interface PaymentGatewayInterface
{
    public function charge(float $amount): bool;
}

class StripePaymentService implements PaymentGatewayInterface
{
    public function charge(float $amount): bool
    {
        // Stripe logic
        return true;
    }
}

class OrderController extends Controller
{
    public function __construct(
        private OrderRepositoryInterface $orders,
        private PaymentGatewayInterface $payment,
        private OrderMailer $mailer
    ) {}
    
    public function store(Request $request)
    {
        $order = $this->orders->create($request->validated());
        
        if ($this->payment->charge($order->total)) {
            $this->mailer->sendConfirmation($order);
        }
        
        return response()->json($order);
    }
}

// В Service Provider
$this->app->bind(PaymentGatewayInterface::class, StripePaymentService::class);
```

---

## SOLID в Laravel

### Пример полного соблюдения SOLID

```php
// Интерфейсы (DIP)
interface UserRepositoryInterface
{
    public function find(int $id): ?User;
    public function create(array $data): User;
}

interface UserValidatorInterface
{
    public function validate(array $data): bool;
}

interface UserNotifierInterface
{
    public function sendWelcomeEmail(User $user): void;
}

// Реализации (SRP - каждый класс делает одно дело)
class EloquentUserRepository implements UserRepositoryInterface
{
    public function find(int $id): ?User
    {
        return User::find($id);
    }
    
    public function create(array $data): User
    {
        return User::create($data);
    }
}

class UserValidator implements UserValidatorInterface
{
    public function validate(array $data): bool
    {
        return Validator::make($data, [
            'email' => 'required|email|unique:users',
            'name' => 'required|string',
        ])->passes();
    }
}

class UserMailNotifier implements UserNotifierInterface
{
    public function sendWelcomeEmail(User $user): void
    {
        Mail::to($user)->send(new WelcomeEmail($user));
    }
}

// Service (OCP - открыт для расширения через DI)
class UserRegistrationService
{
    public function __construct(
        private UserRepositoryInterface $repository,
        private UserValidatorInterface $validator,
        private UserNotifierInterface $notifier
    ) {}
    
    public function register(array $data): User
    {
        if (!$this->validator->validate($data)) {
            throw new ValidationException('Invalid data');
        }
        
        $user = $this->repository->create($data);
        $this->notifier->sendWelcomeEmail($user);
        
        return $user;
    }
}

// Controller (тонкий слой)
class UserController extends Controller
{
    public function __construct(
        private UserRegistrationService $registrationService
    ) {}
    
    public function register(Request $request)
    {
        try {
            $user = $this->registrationService->register($request->all());
            return response()->json($user, 201);
        } catch (ValidationException $e) {
            return response()->json(['error' => $e->getMessage()], 422);
        }
    }
}

// Service Provider (регистрация зависимостей)
class AppServiceProvider extends ServiceProvider
{
    public function register()
    {
        $this->app->bind(
            UserRepositoryInterface::class,
            EloquentUserRepository::class
        );
        
        $this->app->bind(
            UserValidatorInterface::class,
            UserValidator::class
        );
        
        $this->app->bind(
            UserNotifierInterface::class,
            UserMailNotifier::class
        );
    }
}
```

---

## Ключевые вопросы для интервью

- Объясните каждый принцип SOLID своими словами
- Приведите пример нарушения SRP
- Как OCP помогает в расширении функциональности?
- Что такое LSP и почему Square не должен наследоваться от Rectangle?
- Чем ISP помогает при проектировании интерфейсов?
- Объясните DIP на примере Laravel приложения
- Как SOLID принципы помогают в тестировании?
- Какие проблемы решает каждый принцип?
- Можете ли вы привести пример из вашего опыта, где применяли SOLID?
- Как SOLID связан с паттернами проектирования?

---

## 🎓 Для собеседования: ключевые точки

1. **Single Responsibility (SRP)** - одна причина для изменения. Класс делает одно дело хорошо
2. **Open/Closed (OCP)** - открыт для расширения (наследование/интерфейсы), закрыт для модификации
3. **Liskov Substitution (LSP)** - подклассы могут заменять базовый класс без изменения поведения
4. **Interface Segregation (ISP)** - много специализированных интерфейсов лучше одного универсального
5. **Dependency Inversion (DIP)** - завись от абстракций (интерфейсов), не от конкретных классов
6. **Dependency Injection ≠ DIP** - DI = техника, DIP = принцип
7. **SRP пример** - UserController (не должен хранить DB логику, email отправку)
8. **OCP пример** - Strategy pattern, добавляй новые стратегии без изменения существующих
9. **LSP пример** - Square extends Rectangle нарушает LSP (setWidth/setHeight)
10. **ISP пример** - Workable, Eatable, Sleepable вместо одного Worker

**Главное:** SOLID - основа чистого кода. Понимай каждый принцип, умей приводить примеры нарушений и решений.

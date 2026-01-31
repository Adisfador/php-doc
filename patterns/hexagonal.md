# Hexagonal Architecture (Ports & Adapters)

Полный разбор гексагональной архитектуры: изоляция бизнес-логики, порты, адаптеры, dependency inversion.

---

## 🎯 Что такое Hexagonal Architecture?

**Hexagonal Architecture** (Ports & Adapters) - паттерн изоляции бизнес-логики от внешних зависимостей.

**Автор:** Alistair Cockburn (2005)

**Цель:**
- Бизнес-логика НЕ зависит от UI, DB, frameworks
- Легко тестировать (mock внешние зависимости)
- Легко менять технологии (MySQL → MongoDB, HTTP → CLI)

```
        ┌────────────────────┐
        │   Presentation     │ ← Driving Adapters
        │  (HTTP, CLI, UI)   │
        └─────────┬──────────┘
                  │
         ┌────────▼────────┐
         │   Primary       │
         │   Ports         │
         │  (Interfaces)   │
         └────────┬────────┘
                  │
    ┌─────────────▼─────────────┐
    │     CORE DOMAIN           │
    │   (Business Logic)        │
    │  - Entities               │
    │  - Value Objects          │
    │  - Use Cases              │
    └─────────────┬─────────────┘
                  │
         ┌────────▼────────┐
         │   Secondary     │
         │   Ports         │
         │  (Interfaces)   │
         └────────┬────────┘
                  │
        ┌─────────▼──────────┐
        │   Infrastructure   │ ← Driven Adapters
        │ (DB, Email, APIs)  │
        └────────────────────┘
```

**Hexagon (шестиугольник)** - просто метафора, количество сторон не важно.

---

## 🔌 Ports (Порты)

**Port** = **Interface** определяющий контракт.

### Primary Ports (Driving)

**Входящие** - что приложение **предоставляет** внешнему миру.

```php
// Port (Interface)
interface UserRegistrationUseCase
{
    public function registerUser(string $name, string $email, string $password): User;
}
```

**Кто вызывает:** Controllers, CLI commands, GraphQL resolvers.

### Secondary Ports (Driven)

**Исходящие** - что приложение **требует** от внешнего мира.

```php
// Port (Interface)
interface UserRepository
{
    public function save(User $user): void;
    public function findByEmail(string $email): ?User;
}

interface EmailSender
{
    public function sendWelcomeEmail(User $user): void;
}
```

**Кто реализует:** Database adapters, SMTP adapters, API clients.

---

## 🔄 Adapters (Адаптеры)

**Adapter** = **Реализация** порта.

### Primary Adapters (Driving)

**Вызывают** use cases через primary ports.

```php
// HTTP Adapter (Controller)
class UserController extends Controller
{
    public function __construct(
        private UserRegistrationUseCase $registerUser
    ) {}
    
    public function register(Request $request): JsonResponse
    {
        $validated = $request->validate([
            'name' => 'required|max:255',
            'email' => 'required|email|unique:users',
            'password' => 'required|min:8',
        ]);
        
        $user = $this->registerUser->registerUser(
            $validated['name'],
            $validated['email'],
            $validated['password']
        );
        
        return response()->json($user, 201);
    }
}

// CLI Adapter (Console Command)
class RegisterUserCommand extends Command
{
    protected $signature = 'user:register {name} {email} {password}';
    
    public function __construct(
        private UserRegistrationUseCase $registerUser
    ) {
        parent::__construct();
    }
    
    public function handle(): int
    {
        $user = $this->registerUser->registerUser(
            $this->argument('name'),
            $this->argument('email'),
            $this->argument('password')
        );
        
        $this->info("User {$user->id} registered successfully!");
        
        return 0;
    }
}
```

### Secondary Adapters (Driven)

**Реализуют** secondary ports для инфраструктуры.

```php
// Database Adapter
class EloquentUserRepository implements UserRepository
{
    public function save(User $user): void
    {
        UserModel::updateOrCreate(
            ['id' => $user->getId()],
            [
                'name' => $user->getName(),
                'email' => $user->getEmail(),
                'password' => $user->getPassword(),
            ]
        );
    }
    
    public function findByEmail(string $email): ?User
    {
        $model = UserModel::where('email', $email)->first();
        
        if (!$model) {
            return null;
        }
        
        return new User(
            id: $model->id,
            name: $model->name,
            email: $model->email,
            password: $model->password
        );
    }
}

// SMTP Adapter
class SmtpEmailSender implements EmailSender
{
    public function sendWelcomeEmail(User $user): void
    {
        Mail::to($user->getEmail())->send(new WelcomeEmail($user));
    }
}

// Alternative Adapter (для тестов)
class InMemoryUserRepository implements UserRepository
{
    private array $users = [];
    
    public function save(User $user): void
    {
        $this->users[$user->getId()] = $user;
    }
    
    public function findByEmail(string $email): ?User
    {
        foreach ($this->users as $user) {
            if ($user->getEmail() === $email) {
                return $user;
            }
        }
        return null;
    }
}
```

---

## 🎯 Core Domain (Ядро)

### Entities

Бизнес-объекты с **идентичностью**.

```php
// Domain Entity
class User
{
    private int $id;
    private string $name;
    private Email $email; // Value Object
    private HashedPassword $password; // Value Object
    private \DateTimeImmutable $createdAt;
    
    public function __construct(
        int $id,
        string $name,
        string $email,
        string $password,
        ?\DateTimeImmutable $createdAt = null
    ) {
        $this->id = $id;
        $this->name = $name;
        $this->email = new Email($email);
        $this->password = new HashedPassword($password);
        $this->createdAt = $createdAt ?? new \DateTimeImmutable();
    }
    
    public function changeEmail(string $newEmail): void
    {
        $this->email = new Email($newEmail);
    }
    
    public function changePassword(string $plainPassword): void
    {
        $this->password = HashedPassword::fromPlain($plainPassword);
    }
    
    // Getters
    public function getId(): int { return $this->id; }
    public function getName(): string { return $this->name; }
    public function getEmail(): string { return $this->email->getValue(); }
    public function getPassword(): string { return $this->password->getValue(); }
}
```

### Value Objects

Объекты без идентичности, определяются **значением**.

```php
// Value Object
class Email
{
    private string $value;
    
    public function __construct(string $value)
    {
        if (!filter_var($value, FILTER_VALIDATE_EMAIL)) {
            throw new InvalidEmailException("Invalid email: {$value}");
        }
        
        $this->value = $value;
    }
    
    public function getValue(): string
    {
        return $this->value;
    }
    
    public function equals(Email $other): bool
    {
        return $this->value === $other->value;
    }
}

class HashedPassword
{
    private string $hash;
    
    private function __construct(string $hash)
    {
        $this->hash = $hash;
    }
    
    public static function fromPlain(string $plain): self
    {
        return new self(password_hash($plain, PASSWORD_ARGON2ID));
    }
    
    public static function fromHash(string $hash): self
    {
        return new self($hash);
    }
    
    public function verify(string $plain): bool
    {
        return password_verify($plain, $this->hash);
    }
    
    public function getValue(): string
    {
        return $this->hash;
    }
}
```

### Use Cases (Application Services)

Оркеструют бизнес-логику, **не содержат** ее.

```php
// Use Case Implementation
class RegisterUserService implements UserRegistrationUseCase
{
    public function __construct(
        private UserRepository $userRepository,
        private EmailSender $emailSender
    ) {}
    
    public function registerUser(string $name, string $email, string $password): User
    {
        // Проверка уникальности
        if ($this->userRepository->findByEmail($email)) {
            throw new EmailAlreadyExistsException();
        }
        
        // Создание entity
        $user = new User(
            id: $this->generateId(),
            name: $name,
            email: $email,
            password: $password
        );
        
        // Сохранение через port
        $this->userRepository->save($user);
        
        // Отправка email через port
        $this->emailSender->sendWelcomeEmail($user);
        
        return $user;
    }
    
    private function generateId(): int
    {
        return random_int(1, PHP_INT_MAX);
    }
}
```

---

## 📁 Структура проекта

```
app/
├── Domain/                     ← CORE
│   ├── Entity/
│   │   └── User.php
│   ├── ValueObject/
│   │   ├── Email.php
│   │   └── HashedPassword.php
│   ├── Exception/
│   │   ├── EmailAlreadyExistsException.php
│   │   └── InvalidEmailException.php
│   └── Port/                   ← Interfaces (Ports)
│       ├── UserRepository.php
│       └── EmailSender.php
│
├── Application/                ← Use Cases
│   ├── UseCase/
│   │   └── UserRegistrationUseCase.php
│   └── Service/
│       └── RegisterUserService.php
│
├── Infrastructure/             ← Driven Adapters
│   ├── Persistence/
│   │   ├── Eloquent/
│   │   │   ├── UserModel.php
│   │   │   └── EloquentUserRepository.php
│   │   └── InMemory/
│   │       └── InMemoryUserRepository.php
│   └── Email/
│       └── SmtpEmailSender.php
│
└── Presentation/               ← Driving Adapters
    ├── Http/
    │   └── Controllers/
    │       └── UserController.php
    └── Console/
        └── Commands/
            └── RegisterUserCommand.php
```

---

## 🔧 Dependency Injection (Laravel)

**Binding ports → adapters** в `AppServiceProvider`:

```php
// app/Providers/AppServiceProvider.php
class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // Bind Ports to Adapters
        $this->app->bind(
            UserRepository::class,
            EloquentUserRepository::class
        );
        
        $this->app->bind(
            EmailSender::class,
            SmtpEmailSender::class
        );
        
        $this->app->bind(
            UserRegistrationUseCase::class,
            RegisterUserService::class
        );
    }
}
```

**Для тестов** - другие адаптеры:

```php
// tests/Feature/UserRegistrationTest.php
class UserRegistrationTest extends TestCase
{
    public function test_user_can_register()
    {
        // Подмена адаптеров для тестов
        $this->app->bind(UserRepository::class, InMemoryUserRepository::class);
        $this->app->bind(EmailSender::class, FakeEmailSender::class);
        
        $response = $this->postJson('/api/users', [
            'name' => 'John Doe',
            'email' => 'john@example.com',
            'password' => 'secret123',
        ]);
        
        $response->assertStatus(201);
    }
}
```

---

## 🧪 Тестирование

### Unit Test (Use Case)

```php
class RegisterUserServiceTest extends TestCase
{
    public function test_registers_user_successfully()
    {
        // Arrange
        $userRepo = new InMemoryUserRepository();
        $emailSender = new FakeEmailSender();
        $service = new RegisterUserService($userRepo, $emailSender);
        
        // Act
        $user = $service->registerUser('John', 'john@example.com', 'secret123');
        
        // Assert
        $this->assertEquals('john@example.com', $user->getEmail());
        $this->assertTrue($emailSender->wasSentTo('john@example.com'));
    }
    
    public function test_throws_exception_when_email_already_exists()
    {
        $userRepo = new InMemoryUserRepository();
        $userRepo->save(new User(1, 'Jane', 'jane@example.com', 'hash'));
        
        $service = new RegisterUserService($userRepo, new FakeEmailSender());
        
        $this->expectException(EmailAlreadyExistsException::class);
        
        $service->registerUser('John', 'jane@example.com', 'secret');
    }
}
```

### Integration Test (Adapter)

```php
class EloquentUserRepositoryTest extends TestCase
{
    use RefreshDatabase;
    
    public function test_saves_user_to_database()
    {
        $repo = new EloquentUserRepository();
        
        $user = new User(1, 'John', 'john@example.com', 'hash');
        $repo->save($user);
        
        $this->assertDatabaseHas('users', [
            'email' => 'john@example.com',
        ]);
    }
    
    public function test_finds_user_by_email()
    {
        UserModel::create([
            'id' => 1,
            'name' => 'John',
            'email' => 'john@example.com',
            'password' => 'hash',
        ]);
        
        $repo = new EloquentUserRepository();
        $user = $repo->findByEmail('john@example.com');
        
        $this->assertEquals('John', $user->getName());
    }
}
```

---

## 🔄 Пример: смена адаптера

### Было: MySQL

```php
$this->app->bind(UserRepository::class, EloquentUserRepository::class);
```

### Стало: MongoDB

```php
// 1. Создать новый адаптер
class MongoDbUserRepository implements UserRepository
{
    public function __construct(
        private MongoDB\Client $client
    ) {}
    
    public function save(User $user): void
    {
        $collection = $this->client->myapp->users;
        
        $collection->updateOne(
            ['_id' => $user->getId()],
            ['$set' => [
                'name' => $user->getName(),
                'email' => $user->getEmail(),
                'password' => $user->getPassword(),
            ]],
            ['upsert' => true]
        );
    }
    
    public function findByEmail(string $email): ?User
    {
        $collection = $this->client->myapp->users;
        $doc = $collection->findOne(['email' => $email]);
        
        if (!$doc) {
            return null;
        }
        
        return new User(
            id: $doc['_id'],
            name: $doc['name'],
            email: $doc['email'],
            password: $doc['password']
        );
    }
}

// 2. Изменить binding
$this->app->bind(UserRepository::class, MongoDbUserRepository::class);
```

**Бизнес-логика НЕ изменилась!** Только адаптер.

---

## 🆚 Layered vs Hexagonal

| Aspect | Layered | Hexagonal |
|--------|---------|-----------|
| **Зависимости** | Top → Down | Inward (к ядру) |
| **Бизнес-логика** | В Service Layer | В Core Domain |
| **Testability** | Средняя | Высокая |
| **Coupling** | Зависит от infrastructure | Изолировано |
| **Сложность** | Низкая | Средняя |

**Layered:**
```
Controller → Service → Repository → DB
(зависимость направлена вниз)
```

**Hexagonal:**
```
Controller → UseCase ← Repository
               ↓
            (Port)
            (Core НЕ зависит от DB)
```

---

## ✅ Преимущества Hexagonal Architecture

1. **Testability** - легко моки для портов
2. **Flexibility** - меняем адаптеры без изменения ядра
3. **Technology Independence** - ядро не зависит от Laravel/DB
4. **Business Logic Isolation** - вся логика в одном месте
5. **Multiple Interfaces** - HTTP + CLI + GraphQL используют одни use cases

---

## ❌ Недостатки

1. **Complexity** - больше файлов и абстракций
2. **Overhead** - для простых CRUD избыточно
3. **Learning Curve** - команда должна понимать концепцию
4. **Boilerplate** - много интерфейсов и реализаций

---

## 🎯 Когда использовать Hexagonal Architecture?

### ✅ Используй когда:

- **Сложная бизнес-логика** (не просто CRUD)
- **Долгосрочный проект** (5+ лет)
- **Частая смена технологий** (MySQL → PostgreSQL, HTTP → gRPC)
- **Высокие требования к тестированию** (медицина, финансы)
- **Много интерфейсов** (Web + Mobile API + CLI + GraphQL)

### ❌ НЕ используй когда:

- **Простое CRUD приложение**
- **Маленькая команда** (<3 developers)
- **Прототип/MVP**
- **Deadline через 2 недели**
- **Стандартный Laravel MVC достаточен**

---

## 🚀 Практический пример: Order Service

### Ports

```php
// Primary Port
interface PlaceOrderUseCase
{
    public function placeOrder(int $userId, array $items): Order;
}

// Secondary Ports
interface OrderRepository
{
    public function save(Order $order): void;
    public function findById(int $id): ?Order;
}

interface PaymentGateway
{
    public function charge(Money $amount, string $token): PaymentResult;
}

interface InventoryService
{
    public function reserve(int $productId, int $quantity): void;
}

interface NotificationService
{
    public function notifyOrderPlaced(Order $order): void;
}
```

### Use Case

```php
class PlaceOrderService implements PlaceOrderUseCase
{
    public function __construct(
        private OrderRepository $orderRepository,
        private PaymentGateway $paymentGateway,
        private InventoryService $inventoryService,
        private NotificationService $notificationService
    ) {}
    
    public function placeOrder(int $userId, array $items): Order
    {
        // 1. Создать Order
        $order = new Order($userId, $items);
        
        // 2. Резервировать товар
        foreach ($order->getItems() as $item) {
            $this->inventoryService->reserve(
                $item->getProductId(),
                $item->getQuantity()
            );
        }
        
        // 3. Провести оплату
        $result = $this->paymentGateway->charge(
            $order->getTotal(),
            $order->getPaymentToken()
        );
        
        if (!$result->isSuccessful()) {
            throw new PaymentFailedException();
        }
        
        $order->markAsPaid($result->getTransactionId());
        
        // 4. Сохранить
        $this->orderRepository->save($order);
        
        // 5. Уведомить
        $this->notificationService->notifyOrderPlaced($order);
        
        return $order;
    }
}
```

### Adapters

```php
// Database Adapter
class EloquentOrderRepository implements OrderRepository
{
    public function save(Order $order): void
    {
        OrderModel::updateOrCreate(
            ['id' => $order->getId()],
            [
                'user_id' => $order->getUserId(),
                'total' => $order->getTotal()->getAmount(),
                'status' => $order->getStatus(),
                'transaction_id' => $order->getTransactionId(),
            ]
        );
    }
    
    public function findById(int $id): ?Order
    {
        $model = OrderModel::find($id);
        return $model ? $this->toDomain($model) : null;
    }
}

// Stripe Adapter
class StripePaymentGateway implements PaymentGateway
{
    public function charge(Money $amount, string $token): PaymentResult
    {
        $stripe = new \Stripe\StripeClient(config('services.stripe.secret'));
        
        try {
            $charge = $stripe->charges->create([
                'amount' => $amount->getAmount(),
                'currency' => 'usd',
                'source' => $token,
            ]);
            
            return PaymentResult::success($charge->id);
        } catch (\Exception $e) {
            return PaymentResult::failed($e->getMessage());
        }
    }
}

// HTTP Adapter (для внешнего Inventory API)
class HttpInventoryService implements InventoryService
{
    public function reserve(int $productId, int $quantity): void
    {
        $response = Http::post('https://inventory-service.com/api/reserve', [
            'product_id' => $productId,
            'quantity' => $quantity,
        ]);
        
        if (!$response->successful()) {
            throw new InventoryReservationFailedException();
        }
    }
}

// Email Adapter
class EmailNotificationService implements NotificationService
{
    public function notifyOrderPlaced(Order $order): void
    {
        Mail::to($order->getUserEmail())->send(new OrderPlacedEmail($order));
    }
}
```

### Testing

```php
class PlaceOrderServiceTest extends TestCase
{
    public function test_places_order_successfully()
    {
        // Arrange - fake adapters
        $orderRepo = new InMemoryOrderRepository();
        $paymentGateway = new FakePaymentGateway();
        $inventoryService = new FakeInventoryService();
        $notificationService = new FakeNotificationService();
        
        $service = new PlaceOrderService(
            $orderRepo,
            $paymentGateway,
            $inventoryService,
            $notificationService
        );
        
        // Act
        $order = $service->placeOrder(userId: 1, items: [
            ['product_id' => 10, 'quantity' => 2],
        ]);
        
        // Assert
        $this->assertEquals('paid', $order->getStatus());
        $this->assertTrue($inventoryService->wasReserved(10, 2));
        $this->assertTrue($notificationService->wasNotified($order->getId()));
    }
}
```

---

## 🎓 Для собеседования: ключевые точки

1. **Hexagonal = Ports & Adapters** - изоляция бизнес-логики
2. **Ports** - интерфейсы (Primary входящие, Secondary исходящие)
3. **Adapters** - реализации (HTTP, DB, Email)
4. **Core Domain** - Entities, Value Objects, Use Cases
5. **Dependency Inversion** - ядро определяет интерфейсы, инфраструктура реализует
6. **Testability** - легко моки через порты
7. **Flexibility** - меняй адаптеры (MySQL → MongoDB) без изменения ядра
8. **Trade-off** - сложность vs flexibility

**Главное:** Hexagonal изолирует бизнес-логику от технологий, делая систему гибкой и тестируемой.

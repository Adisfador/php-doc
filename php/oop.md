# ООП в PHP (Deep Dive)

## Основы ООП

### Класс и объект

```php
class User
{
    // Свойства (properties)
    private string $name;
    private int $age;
    
    // Конструктор
    public function __construct(string $name, int $age)
    {
        $this->name = $name;
        $this->age = $age;
    }
    
    // Методы
    public function getName(): string
    {
        return $this->name;
    }
}

// Создание объекта
$user = new User('John', 30);
```

### Видимость (Access Modifiers)

```php
class Example
{
    public $public;       // Доступно везде
    protected $protected; // Доступно в классе и наследниках
    private $private;     // Доступно только в этом классе
    
    public function publicMethod() {}
    protected function protectedMethod() {}
    private function privateMethod() {}
}
```

---

## Наследование (Inheritance)

```php
class Animal
{
    protected string $name;
    
    public function __construct(string $name)
    {
        $this->name = $name;
    }
    
    public function makeSound(): string
    {
        return "Some sound";
    }
}

class Dog extends Animal
{
    public function makeSound(): string
    {
        return "Woof!";
    }
    
    public function wagTail(): void
    {
        echo "{$this->name} is wagging tail";
    }
}

$dog = new Dog('Rex');
echo $dog->makeSound(); // "Woof!"
```

### Final класс и методы

```php
// Нельзя наследовать
final class FinalClass
{
    public function method() {}
}

class RegularClass
{
    // Нельзя переопределить в наследниках
    final public function finalMethod() {}
}
```

---

## Абстрактные классы

```php
abstract class Shape
{
    protected string $color;
    
    public function __construct(string $color)
    {
        $this->color = $color;
    }
    
    // Абстрактный метод (без реализации)
    abstract public function getArea(): float;
    
    // Обычный метод
    public function getColor(): string
    {
        return $this->color;
    }
}

class Circle extends Shape
{
    private float $radius;
    
    public function __construct(string $color, float $radius)
    {
        parent::__construct($color);
        $this->radius = $radius;
    }
    
    // Обязательная реализация
    public function getArea(): float
    {
        return pi() * $this->radius ** 2;
    }
}

// $shape = new Shape('red'); // ОШИБКА! Нельзя создать объект абстрактного класса
$circle = new Circle('red', 5);
```

---

## Интерфейсы (Interfaces)

```php
interface Payable
{
    public function pay(float $amount): bool;
    public function getBalance(): float;
}

interface Refundable
{
    public function refund(float $amount): bool;
}

// Класс может реализовывать несколько интерфейсов
class CreditCard implements Payable, Refundable
{
    private float $balance = 0;
    
    public function pay(float $amount): bool
    {
        if ($this->balance >= $amount) {
            $this->balance -= $amount;
            return true;
        }
        return false;
    }
    
    public function getBalance(): float
    {
        return $this->balance;
    }
    
    public function refund(float $amount): bool
    {
        $this->balance += $amount;
        return true;
    }
}
```

### Интерфейс vs Абстрактный класс

| Критерий | Интерфейс | Абстрактный класс |
|----------|-----------|-------------------|
| **Методы** | Только сигнатуры (PHP 8: можно с реализацией) | Могут быть и абстрактные, и обычные |
| **Свойства** | Только константы | Любые свойства |
| **Множественное наследование** | Да (implements A, B, C) | Нет (extends только один) |
| **Конструктор** | Нет | Может быть |
| **Назначение** | Контракт (что должно быть) | Базовая функциональность |

---

## Трейты (Traits)

### Основы

```php
trait Timestampable
{
    protected ?DateTime $createdAt = null;
    protected ?DateTime $updatedAt = null;
    
    public function setCreatedAt(): void
    {
        $this->createdAt = new DateTime();
    }
    
    public function setUpdatedAt(): void
    {
        $this->updatedAt = new DateTime();
    }
}

class Post
{
    use Timestampable;
    
    private string $title;
    
    public function __construct(string $title)
    {
        $this->title = $title;
        $this->setCreatedAt();
    }
}

$post = new Post('Hello World');
```

### Множественные трейты

```php
trait A
{
    public function methodA() {}
}

trait B
{
    public function methodB() {}
}

class MyClass
{
    use A, B;
}
```

### Конфликты методов

```php
trait Logger
{
    public function log(string $message): void
    {
        echo "Logging: $message";
    }
}

trait Debugger
{
    public function log(string $message): void
    {
        echo "Debugging: $message";
    }
}

class Service
{
    use Logger, Debugger {
        Logger::log insteadof Debugger;  // Использовать Logger::log
        Debugger::log as debugLog;        // Создать алиас для Debugger::log
    }
}

$service = new Service();
$service->log('test');      // "Logging: test"
$service->debugLog('test'); // "Debugging: test"
```

### Изменение видимости

```php
trait HasId
{
    private function getId(): int
    {
        return $this->id;
    }
}

class User
{
    use HasId {
        getId as public;  // Сделать public
    }
    
    private int $id = 1;
}

$user = new User();
echo $user->getId(); // Работает!
```

---

## Магические методы

### __construct и __destruct

```php
class Resource
{
    private $handle;
    
    public function __construct(string $filename)
    {
        $this->handle = fopen($filename, 'r');
        echo "Resource opened\n";
    }
    
    public function __destruct()
    {
        fclose($this->handle);
        echo "Resource closed\n";
    }
}

$resource = new Resource('file.txt');
// ... использование
// При уничтожении объекта вызовется __destruct
```

### __get, __set, __isset, __unset

```php
class DynamicProperties
{
    private array $data = [];
    
    public function __get(string $name)
    {
        echo "Getting '$name'\n";
        return $data[$name] ?? null;
    }
    
    public function __set(string $name, $value): void
    {
        echo "Setting '$name' to '$value'\n";
        $this->data[$name] = $value;
    }
    
    public function __isset(string $name): bool
    {
        return isset($this->data[$name]);
    }
    
    public function __unset(string $name): void
    {
        unset($this->data[$name]);
    }
}

$obj = new DynamicProperties();
$obj->name = 'John';     // __set
echo $obj->name;         // __get
isset($obj->name);       // __isset
unset($obj->name);       // __unset
```

### __call и __callStatic

```php
class DynamicMethods
{
    public function __call(string $name, array $arguments)
    {
        echo "Calling object method '$name' with args: " . implode(', ', $arguments);
    }
    
    public static function __callStatic(string $name, array $arguments)
    {
        echo "Calling static method '$name' with args: " . implode(', ', $arguments);
    }
}

$obj = new DynamicMethods();
$obj->someMethod(1, 2, 3);         // __call
DynamicMethods::someMethod(1, 2);  // __callStatic
```

### __toString

```php
class User
{
    public function __construct(
        private string $name,
        private string $email
    ) {}
    
    public function __toString(): string
    {
        return "{$this->name} ({$this->email})";
    }
}

$user = new User('John', 'john@example.com');
echo $user; // "John (john@example.com)"
```

### __invoke

```php
class Multiplier
{
    public function __construct(
        private int $factor
    ) {}
    
    public function __invoke(int $number): int
    {
        return $number * $this->factor;
    }
}

$double = new Multiplier(2);
echo $double(5); // 10 (объект вызывается как функция!)
```

### __clone

```php
class Address
{
    public function __construct(
        public string $city
    ) {}
}

class Person
{
    public function __construct(
        public string $name,
        public Address $address
    ) {}
    
    public function __clone()
    {
        // Deep clone
        $this->address = clone $this->address;
    }
}

$person1 = new Person('John', new Address('Moscow'));
$person2 = clone $person1;

$person2->address->city = 'SPB';

echo $person1->address->city; // "Moscow" (не изменилось благодаря __clone)
echo $person2->address->city; // "SPB"
```

### __serialize и __unserialize (PHP 7.4+)

```php
class User
{
    public function __construct(
        private string $name,
        private string $password
    ) {}
    
    public function __serialize(): array
    {
        return [
            'name' => $this->name,
            // НЕ сериализуем пароль!
        ];
    }
    
    public function __unserialize(array $data): void
    {
        $this->name = $data['name'];
        $this->password = ''; // Сброс пароля
    }
}
```

### __debugInfo

```php
class User
{
    public function __construct(
        private string $name,
        private string $password
    ) {}
    
    public function __debugInfo(): array
    {
        return [
            'name' => $this->name,
            'password' => '***hidden***',
        ];
    }
}

$user = new User('John', 'secret123');
var_dump($user);
// Выведет: password => "***hidden***"
```

---

## Статическое связывание (Late Static Binding)

### Проблема без LSB

```php
class A
{
    public static function who()
    {
        return 'A';
    }
    
    public static function test()
    {
        return self::who(); // self = A (всегда!)
    }
}

class B extends A
{
    public static function who()
    {
        return 'B';
    }
}

echo B::test(); // "A" (ожидали "B"!)
```

### Решение: static

```php
class A
{
    public static function who()
    {
        return 'A';
    }
    
    public static function test()
    {
        return static::who(); // static = вызывающий класс
    }
}

class B extends A
{
    public static function who()
    {
        return 'B';
    }
}

echo B::test(); // "B" (правильно!)
```

### Практическое применение

```php
abstract class Model
{
    protected static string $table;
    
    public static function find(int $id): static
    {
        $table = static::$table; // Таблица конкретной модели
        // SELECT * FROM $table WHERE id = $id
        return new static(); // Создание объекта вызывающего класса
    }
}

class User extends Model
{
    protected static string $table = 'users';
}

class Post extends Model
{
    protected static string $table = 'posts';
}

$user = User::find(1);  // SELECT * FROM users
$post = Post::find(1);  // SELECT * FROM posts
```

---

## Анонимные классы (PHP 7+)

```php
// Создание анонимного класса
$logger = new class {
    public function log(string $message): void
    {
        echo "[LOG] $message\n";
    }
};

$logger->log('Hello');

// С конструктором
$point = new class(10, 20) {
    public function __construct(
        private int $x,
        private int $y
    ) {}
    
    public function getDistance(): float
    {
        return sqrt($this->x ** 2 + $this->y ** 2);
    }
};

// Реализация интерфейса
interface Logger
{
    public function log(string $message): void;
}

function process(Logger $logger)
{
    $logger->log('Processing...');
}

process(new class implements Logger {
    public function log(string $message): void
    {
        file_put_contents('log.txt', $message . PHP_EOL, FILE_APPEND);
    }
});
```

---

## Объекты как значения (Value Objects)

### Immutable объекты

```php
final readonly class Money
{
    public function __construct(
        public int $amount,
        public string $currency
    ) {}
    
    public function add(Money $money): self
    {
        if ($this->currency !== $money->currency) {
            throw new InvalidArgumentException('Currency mismatch');
        }
        
        return new self(
            $this->amount + $money->amount,
            $this->currency
        );
    }
}

$money1 = new Money(100, 'USD');
$money2 = new Money(50, 'USD');
$total = $money1->add($money2); // Новый объект

// $money1->amount = 200; // ОШИБКА! readonly
```

---

## Сравнение объектов

```php
class Point
{
    public function __construct(
        public int $x,
        public int $y
    ) {}
}

$p1 = new Point(1, 2);
$p2 = new Point(1, 2);
$p3 = $p1;

// == сравнивает значения свойств
var_dump($p1 == $p2);  // true

// === сравнивает ссылки
var_dump($p1 === $p2); // false
var_dump($p1 === $p3); // true
```

---

## Reflection API

```php
class User
{
    private string $name;
    
    public function getName(): string
    {
        return $this->name;
    }
}

$reflector = new ReflectionClass(User::class);

// Методы
foreach ($reflector->getMethods() as $method) {
    echo $method->getName() . "\n";
}

// Свойства
foreach ($reflector->getProperties() as $property) {
    echo $property->getName() . "\n";
}

// Создание объекта без вызова конструктора
$user = $reflector->newInstanceWithoutConstructor();

// Доступ к private свойству
$property = $reflector->getProperty('name');
$property->setAccessible(true);
$property->setValue($user, 'John');

echo $user->getName(); // "John"
```

---

## Property Hooks (PHP 8.4+)

```php
class User
{
    // Get hook
    public string $fullName {
        get => $this->firstName . ' ' . $this->lastName;
    }
    
    // Set hook
    public string $email {
        set (string $value) {
            if (!filter_var($value, FILTER_VALIDATE_EMAIL)) {
                throw new ValueError('Invalid email');
            }
            $this->email = strtolower($value);
        }
    }
    
    public function __construct(
        public string $firstName,
        public string $lastName,
        string $email
    ) {
        $this->email = $email;
    }
}

$user = new User('John', 'Doe', 'JOHN@EXAMPLE.COM');
echo $user->fullName; // "John Doe" (computed)
echo $user->email;    // "john@example.com" (normalized)
```

---

## Best Practices

### 1. Композиция над наследованием

```php
// ❌ Плохо: глубокая иерархия наследования
class Animal {}
class Mammal extends Animal {}
class Dog extends Mammal {}
class Labrador extends Dog {}

// ✅ Хорошо: композиция
interface CanBark
{
    public function bark(): string;
}

interface CanRun
{
    public function run(): void;
}

class Dog implements CanBark, CanRun
{
    public function bark(): string
    {
        return 'Woof!';
    }
    
    public function run(): void
    {
        echo 'Running...';
    }
}
```

### 2. Инкапсуляция

```php
// ❌ Плохо
class User
{
    public string $email; // Можно установить невалидный email
}

// ✅ Хорошо
class User
{
    private string $email;
    
    public function __construct(string $email)
    {
        $this->setEmail($email);
    }
    
    public function setEmail(string $email): void
    {
        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
            throw new InvalidArgumentException('Invalid email');
        }
        $this->email = $email;
    }
    
    public function getEmail(): string
    {
        return $this->email;
    }
}
```

### 3. Dependency Injection

```php
// ❌ Плохо: жесткая зависимость
class UserService
{
    private Database $db;
    
    public function __construct()
    {
        $this->db = new MySQLDatabase(); // Жесткая связь!
    }
}

// ✅ Хорошо: внедрение зависимости
class UserService
{
    public function __construct(
        private DatabaseInterface $db
    ) {}
}

$service = new UserService(new MySQLDatabase());
```

---

## Enums (Перечисления) - PHP 8.1+

### Что такое Enums?

**Enum** - это пользовательский тип, представляющий фиксированный набор возможных значений.

**Преимущества:**
- ✅ Типобезопасность (нельзя передать неверное значение)
- ✅ Автокомплит в IDE
- ✅ Рефакторинг безопасен
- ✅ Самодокументируемый код

### Pure Enums (Unit Enums)

```php
enum Status
{
    case Pending;
    case Processing;
    case Completed;
    case Cancelled;
}

// Использование
$status = Status::Pending;

if ($status === Status::Pending) {
    echo "Order is pending";
}

// Нельзя создать неверное значение
$status = Status::Invalid;  // ❌ Fatal Error

// ✅ vs константы класса
class OldStatus {
    const PENDING = 'pending';
    const COMPLETED = 'completed';
}
$status = 'invalid';  // ❌ Компилятор не поймает ошибку!
```

### Backed Enums (со значениями)

```php
// String-backed enum
enum Status: string
{
    case Pending = 'pending';
    case Processing = 'processing';
    case Completed = 'completed';
    case Cancelled = 'cancelled';
}

// Int-backed enum
enum Priority: int
{
    case Low = 1;
    case Medium = 2;
    case High = 3;
    case Critical = 4;
}

// Получить значение
$status = Status::Pending;
echo $status->value;  // 'pending'

$priority = Priority::High;
echo $priority->value;  // 3

// Создать из значения
$status = Status::from('pending');  // Status::Pending

// tryFrom - возвращает null если не найдено
$status = Status::tryFrom('invalid');  // null
$status = Status::from('invalid');     // ValueError

// Полезно для БД
$order->status = Status::Processing;
$order->save();  // 'processing' в БД

$order = Order::find(1);
$status = Status::from($order->status);  // Status enum
```

### Методы в Enums

```php
enum Status: string
{
    case Pending = 'pending';
    case Processing = 'processing';
    case Completed = 'completed';
    case Cancelled = 'cancelled';
    
    // Метод экземпляра
    public function label(): string
    {
        return match($this) {
            self::Pending => 'В ожидании',
            self::Processing => 'В обработке',
            self::Completed => 'Завершен',
            self::Cancelled => 'Отменен',
        };
    }
    
    // Проверки
    public function isFinished(): bool
    {
        return $this === self::Completed || $this === self::Cancelled;
    }
    
    // Цвет для UI
    public function color(): string
    {
        return match($this) {
            self::Pending => 'yellow',
            self::Processing => 'blue',
            self::Completed => 'green',
            self::Cancelled => 'red',
        };
    }
}

// Использование
$status = Status::Processing;
echo $status->label();  // 'В обработке'
echo $status->color();  // 'blue'

if ($status->isFinished()) {
    echo "Order is finished";
}
```

### Статические методы

```php
enum Status: string
{
    case Pending = 'pending';
    case Completed = 'completed';
    case Cancelled = 'cancelled';
    
    // Фабричные методы
    public static function fromOrder(Order $order): self
    {
        if ($order->isPaid() && $order->isShipped()) {
            return self::Completed;
        }
        
        return self::Pending;
    }
    
    // Группировка
    public static function activeStatuses(): array
    {
        return [self::Pending, self::Processing];
    }
    
    public static function finishedStatuses(): array
    {
        return [self::Completed, self::Cancelled];
    }
}

// Использование
$status = Status::fromOrder($order);
$active = Status::activeStatuses();
```

### Получить все cases

```php
enum Status: string
{
    case Pending = 'pending';
    case Processing = 'processing';
    case Completed = 'completed';
}

// Все cases
$all = Status::cases();
// [Status::Pending, Status::Processing, Status::Completed]

foreach (Status::cases() as $status) {
    echo $status->name;   // 'Pending', 'Processing', ...
    echo $status->value;  // 'pending', 'processing', ...
}

// Для формы выбора
<select name="status">
    @foreach(Status::cases() as $status)
        <option value="{{ $status->value }}">
            {{ $status->label() }}
        </option>
    @endforeach
</select>
```

### Enum с интерфейсами

```php
interface HasColor
{
    public function color(): string;
}

enum Status: string implements HasColor
{
    case Pending = 'pending';
    case Processing = 'processing';
    case Completed = 'completed';
    
    public function color(): string
    {
        return match($this) {
            self::Pending => 'yellow',
            self::Processing => 'blue',
            self::Completed => 'green',
        };
    }
}

enum Priority: int implements HasColor
{
    case Low = 1;
    case High = 2;
    
    public function color(): string
    {
        return $this === self::Low ? 'gray' : 'red';
    }
}

// Одинаковый интерфейс для разных enums
function renderBadge(HasColor $item): string
{
    return "<span style='color: {$item->color()}'>{$item->name}</span>";
}
```

### Traits в Enums (PHP 8.2+)

```php
trait EnumHelpers
{
    public static function names(): array
    {
        return array_column(self::cases(), 'name');
    }
    
    public static function values(): array
    {
        return array_column(self::cases(), 'value');
    }
}

enum Status: string
{
    use EnumHelpers;
    
    case Pending = 'pending';
    case Completed = 'completed';
}

$names = Status::names();    // ['Pending', 'Completed']
$values = Status::values();  // ['pending', 'completed']
```

### Константы в Enums

```php
enum Status: string
{
    case Pending = 'pending';
    case Completed = 'completed';
    
    // Константы (НЕ cases!)
    public const DEFAULT = self::Pending;
    public const TRANSITIONS = [
        'pending' => ['completed', 'cancelled'],
        'completed' => [],
    ];
    
    public function canTransitionTo(self $newStatus): bool
    {
        $allowed = self::TRANSITIONS[$this->value] ?? [];
        return in_array($newStatus->value, $allowed);
    }
}

// Использование
$status = Status::DEFAULT;  // Status::Pending

if ($currentStatus->canTransitionTo(Status::Completed)) {
    // Можно перейти
}
```

### Сравнение с константами класса

```php
// ❌ Старый способ (константы)
class Status {
    const PENDING = 1;
    const COMPLETED = 2;
}

function updateStatus(int $status) {
    // Можно передать любое int значение!
    if ($status === 999) {  // ❌ Невалидный статус, но компилятор не поймает
        // ...
    }
}

updateStatus(Status::PENDING);  // ✅
updateStatus(999);  // ❌ Компилируется, но логически неверно!

// ✅ Новый способ (enum)
enum Status: int {
    case Pending = 1;
    case Completed = 2;
}

function updateStatus(Status $status) {
    // Можно передать только валидный Status!
}

updateStatus(Status::Pending);  // ✅
updateStatus(999);  // ❌ Fatal Error: невозможно передать int
```

### Практический пример: State Machine

```php
enum OrderStatus: string
{
    case Draft = 'draft';
    case Pending = 'pending';
    case Paid = 'paid';
    case Shipped = 'shipped';
    case Delivered = 'delivered';
    case Cancelled = 'cancelled';
    
    public function transitions(): array
    {
        return match($this) {
            self::Draft => [self::Pending, self::Cancelled],
            self::Pending => [self::Paid, self::Cancelled],
            self::Paid => [self::Shipped, self::Cancelled],
            self::Shipped => [self::Delivered],
            self::Delivered => [],
            self::Cancelled => [],
        };
    }
    
    public function canTransitionTo(self $newStatus): bool
    {
        return in_array($newStatus, $this->transitions());
    }
    
    public function transitionTo(self $newStatus): self
    {
        if (!$this->canTransitionTo($newStatus)) {
            throw new InvalidArgumentException(
                "Cannot transition from {$this->value} to {$newStatus->value}"
            );
        }
        
        return $newStatus;
    }
    
    public function isActive(): bool
    {
        return !in_array($this, [self::Cancelled, self::Delivered]);
    }
}

// Использование
$order = new Order();
$order->status = OrderStatus::Draft;

// Переход по состояниям
$order->status = $order->status->transitionTo(OrderStatus::Pending);
$order->status = $order->status->transitionTo(OrderStatus::Paid);

// ❌ Невозможный переход
$order->status->transitionTo(OrderStatus::Delivered);  // Exception!
```

### Laravel Integration

```php
// Eloquent cast
class Order extends Model
{
    protected $casts = [
        'status' => Status::class,  // Автоматическое преобразование
    ];
}

$order = Order::find(1);
$order->status;  // Status enum, не string!
$order->status = Status::Completed;
$order->save();

// Validation
$request->validate([
    'status' => ['required', new Enum(Status::class)],
    // Или
    'status' => ['required', Rule::enum(Status::class)],
]);

// Database query
Order::where('status', Status::Pending)->get();
```

### Best Practices

```php
// ✅ Используй Backed Enums для БД
enum Status: string {
    case Pending = 'pending';  // Явное значение для БД
}

// ❌ Pure Enum сложнее хранить в БД
enum Status {
    case Pending;  // Как хранить в БД?
}

// ✅ Методы для логики
enum Status: string {
    case Pending = 'pending';
    
    public function canEdit(): bool {
        return $this === self::Pending;
    }
}

// ❌ Не дублировать логику вне enum
if ($status->value === 'pending') {  // Дублирование!
    // ...
}

// ✅ Используй match для маппинга
public function label(): string {
    return match($this) {
        self::Pending => 'В ожидании',
        self::Completed => 'Завершен',
    };
}

// ❌ Не используй if/elseif
public function label(): string {
    if ($this === self::Pending) return 'В ожидании';
    if ($this === self::Completed) return 'Завершен';
}
```

---

## Ключевые вопросы для интервью

- В чем разница между абстрактным классом и интерфейсом?
- Когда использовать трейты?
- Что такое Late Static Binding и зачем он нужен?
- Объясните разницу между == и === для объектов
- Какие магические методы вы знаете?
- Что такое immutable объект и зачем он нужен?
- В чем разница между self и static?
- Композиция vs Наследование - когда что использовать?
- Как работает __clone и зачем нужен deep clone?
- Что такое Dependency Injection?

---

## 🎓 Для собеседования: ключевые точки

1. **Абстрактный класс vs Интерфейс** - абстрактный может иметь состояние + реализацию, интерфейс - только контракт
2. **Трейты** - повторное использование кода без наследования (horizontal reuse). Конфликты: insteadof/as
3. **Late Static Binding** - static:: разрешается в runtime (на вызывающий класс), self:: в compile time
4. **== vs ===** - == сравнивает свойства, === сравнивает ссылки (тот же объект?)
5. **Магические методы** - __construct, __get/__set, __call/__callStatic, __toString, __invoke, __clone
6. **Immutable объект** - readonly properties (PHP 8.1), нет setters, новый объект при изменении
7. **Композиция > Наследование** - has-a vs is-a. Композиция более гибкая
8. **Deep clone** - клонировать вложенные объекты через __clone, не только ссылки
9. **Dependency Injection** - передача зависимостей через конструктор/сеттеры (не new внутри класса)
10. **Visibility** - public (везде), protected (класс + наследники), private (только класс)
11. **Constructor Property Promotion** - PHP 8.0+ краткий синтаксис для свойств
12. **Полиморфизм** - единый интерфейс для разных реализаций (PaymentInterface)
13. **Enums (PHP 8.1+)** - типобезопасные перечисления. Backed enums (со значениями), Pure enums, from/tryFrom, cases(), методы

**Главное:** Предпочитай композицию наследованию, используй интерфейсы для контрактов, знай когда static:: vs self::, используй Enums вместо констант для фиксированных наборов значений.

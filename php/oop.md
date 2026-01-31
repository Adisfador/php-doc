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

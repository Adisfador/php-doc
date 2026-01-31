# Design Patterns (GoF) - 23 паттерна проектирования

Gang of Four паттерны - классика объектно-ориентированного проектирования. Разделены на три категории.

---

## Порождающие паттерны (Creational)

Отвечают за создание объектов.

### 1. Singleton (Одиночка)

**Цель:** Гарантировать, что у класса есть только один экземпляр, и предоставить глобальную точку доступа.

```php
<?php

class Database
{
    private static ?Database $instance = null;
    private PDO $connection;

    // Private constructor
    private function __construct()
    {
        $this->connection = new PDO(
            'mysql:host=localhost;dbname=test',
            'user',
            'password'
        );
    }

    // Запретить клонирование
    private function __clone() {}

    // Запретить unserialize
    public function __wakeup()
    {
        throw new \Exception("Cannot unserialize singleton");
    }

    public static function getInstance(): Database
    {
        if (self::$instance === null) {
            self::$instance = new self();
        }
        return self::$instance;
    }

    public function query(string $sql): array
    {
        return $this->connection->query($sql)->fetchAll();
    }
}

// Использование
$db1 = Database::getInstance();
$db2 = Database::getInstance();
var_dump($db1 === $db2); // true - один и тот же объект
```

**В Laravel:**
```php
// Service Container - это registry of singletons
app()->singleton(PaymentGateway::class, StripeGateway::class);

$gateway1 = app(PaymentGateway::class);
$gateway2 = app(PaymentGateway::class);
// $gateway1 === $gateway2
```

---

### 2. Factory Method (Фабричный метод)

**Цель:** Определить интерфейс для создания объекта, но позволить подклассам решать, какой класс инстанцировать.

```php
<?php

interface Payment
{
    public function process(float $amount): bool;
}

class CreditCardPayment implements Payment
{
    public function process(float $amount): bool
    {
        echo "Processing ${amount} via Credit Card\n";
        return true;
    }
}

class PayPalPayment implements Payment
{
    public function process(float $amount): bool
    {
        echo "Processing ${amount} via PayPal\n";
        return true;
    }
}

// Factory
abstract class PaymentFactory
{
    abstract public function createPayment(): Payment;

    public function processPayment(float $amount): bool
    {
        $payment = $this->createPayment(); // Factory Method
        return $payment->process($amount);
    }
}

class CreditCardFactory extends PaymentFactory
{
    public function createPayment(): Payment
    {
        return new CreditCardPayment();
    }
}

class PayPalFactory extends PaymentFactory
{
    public function createPayment(): Payment
    {
        return new PayPalPayment();
    }
}

// Использование
$factory = new CreditCardFactory();
$factory->processPayment(100);

$factory = new PayPalFactory();
$factory->processPayment(200);
```

---

### 3. Abstract Factory (Абстрактная фабрика)

**Цель:** Создание семейств связанных объектов без указания конкретных классов.

```php
<?php

// Семейство продуктов: Button и Checkbox
interface Button
{
    public function render(): string;
}

interface Checkbox
{
    public function render(): string;
}

// Windows версии
class WindowsButton implements Button
{
    public function render(): string
    {
        return '<button class="windows">Click</button>';
    }
}

class WindowsCheckbox implements Checkbox
{
    public function render(): string
    {
        return '<input type="checkbox" class="windows">';
    }
}

// Mac версии
class MacButton implements Button
{
    public function render(): string
    {
        return '<button class="mac">Click</button>';
    }
}

class MacCheckbox implements Checkbox
{
    public function render(): string
    {
        return '<input type="checkbox" class="mac">';
    }
}

// Абстрактная фабрика
interface GUIFactory
{
    public function createButton(): Button;
    public function createCheckbox(): Checkbox;
}

class WindowsFactory implements GUIFactory
{
    public function createButton(): Button
    {
        return new WindowsButton();
    }

    public function createCheckbox(): Checkbox
    {
        return new WindowsCheckbox();
    }
}

class MacFactory implements GUIFactory
{
    public function createButton(): Button
    {
        return new MacButton();
    }

    public function createCheckbox(): Checkbox
    {
        return new MacCheckbox();
    }
}

// Использование
function renderUI(GUIFactory $factory)
{
    $button = $factory->createButton();
    $checkbox = $factory->createCheckbox();

    echo $button->render();
    echo $checkbox->render();
}

renderUI(new WindowsFactory());
renderUI(new MacFactory());
```

---

### 4. Builder (Строитель)

**Цель:** Отделить конструирование сложного объекта от его представления.

```php
<?php

class QueryBuilder
{
    private string $table = '';
    private array $columns = ['*'];
    private array $wheres = [];
    private ?int $limit = null;

    public function table(string $table): self
    {
        $this->table = $table;
        return $this;
    }

    public function select(array $columns): self
    {
        $this->columns = $columns;
        return $this;
    }

    public function where(string $column, string $operator, $value): self
    {
        $this->wheres[] = compact('column', 'operator', 'value');
        return $this;
    }

    public function limit(int $limit): self
    {
        $this->limit = $limit;
        return $this;
    }

    public function build(): string
    {
        $sql = 'SELECT ' . implode(', ', $this->columns);
        $sql .= ' FROM ' . $this->table;

        if (!empty($this->wheres)) {
            $conditions = array_map(
                fn($w) => "{$w['column']} {$w['operator']} '{$w['value']}'",
                $this->wheres
            );
            $sql .= ' WHERE ' . implode(' AND ', $conditions);
        }

        if ($this->limit) {
            $sql .= ' LIMIT ' . $this->limit;
        }

        return $sql;
    }
}

// Использование (fluent interface)
$sql = (new QueryBuilder())
    ->table('users')
    ->select(['id', 'name', 'email'])
    ->where('status', '=', 'active')
    ->where('age', '>', '18')
    ->limit(10)
    ->build();

echo $sql;
// SELECT id, name, email FROM users WHERE status = 'active' AND age > '18' LIMIT 10
```

**В Laravel:**
```php
// Eloquent Query Builder - это Builder паттерн
User::where('status', 'active')
    ->where('age', '>', 18)
    ->limit(10)
    ->get();
```

---

### 5. Prototype (Прототип)

**Цель:** Создание новых объектов копированием существующих (прототипов).

```php
<?php

class Document
{
    public string $title;
    public string $content;
    public array $metadata;

    public function __construct(string $title, string $content, array $metadata = [])
    {
        $this->title = $title;
        $this->content = $content;
        $this->metadata = $metadata;
    }

    // Deep clone
    public function __clone()
    {
        $this->metadata = array_map(function ($item) {
            return is_object($item) ? clone $item : $item;
        }, $this->metadata);
    }
}

// Использование
$original = new Document('Original', 'Content', ['author' => 'John']);

// Клонирование
$copy = clone $original;
$copy->title = 'Copy';
$copy->metadata['author'] = 'Jane';

var_dump($original->title); // 'Original'
var_dump($copy->title);     // 'Copy'
```

---

## Структурные паттерны (Structural)

Определяют отношения между классами и объектами.

### 6. Adapter (Адаптер)

**Цель:** Преобразование интерфейса класса в другой интерфейс, ожидаемый клиентами.

```php
<?php

// Целевой интерфейс
interface PaymentGateway
{
    public function pay(float $amount): bool;
}

// Сторонняя библиотека (несовместимый интерфейс)
class StripeAPI
{
    public function createCharge(int $cents): array
    {
        echo "Stripe charging {$cents} cents\n";
        return ['status' => 'success', 'id' => 'ch_123'];
    }
}

// Adapter
class StripeAdapter implements PaymentGateway
{
    private StripeAPI $stripe;

    public function __construct(StripeAPI $stripe)
    {
        $this->stripe = $stripe;
    }

    public function pay(float $amount): bool
    {
        // Адаптируем интерфейс: $amount (float) -> cents (int)
        $cents = (int) ($amount * 100);
        $result = $this->stripe->createCharge($cents);
        return $result['status'] === 'success';
    }
}

// Использование
function processPayment(PaymentGateway $gateway, float $amount)
{
    return $gateway->pay($amount);
}

$stripe = new StripeAPI();
$adapter = new StripeAdapter($stripe);
processPayment($adapter, 99.99); // Stripe charging 9999 cents
```

---

### 7. Bridge (Мост)

**Цель:** Отделить абстракцию от реализации, чтобы они могли изменяться независимо.

```php
<?php

// Реализация (платформы)
interface MessageSender
{
    public function send(string $message): void;
}

class EmailSender implements MessageSender
{
    public function send(string $message): void
    {
        echo "Sending email: {$message}\n";
    }
}

class SmsSender implements MessageSender
{
    public function send(string $message): void
    {
        echo "Sending SMS: {$message}\n";
    }
}

// Абстракция (типы сообщений)
abstract class Notification
{
    protected MessageSender $sender;

    public function __construct(MessageSender $sender)
    {
        $this->sender = $sender;
    }

    abstract public function notify(string $message): void;
}

class UrgentNotification extends Notification
{
    public function notify(string $message): void
    {
        $this->sender->send("[URGENT] " . $message);
    }
}

class RegularNotification extends Notification
{
    public function notify(string $message): void
    {
        $this->sender->send($message);
    }
}

// Использование - можно комбинировать любую абстракцию с любой реализацией
$urgentEmail = new UrgentNotification(new EmailSender());
$urgentEmail->notify("Server down!");

$regularSms = new RegularNotification(new SmsSender());
$regularSms->notify("Update available");
```

---

### 8. Composite (Компоновщик)

**Цель:** Компоновка объектов в древовидные структуры для представления иерархий.

```php
<?php

interface FileSystemItem
{
    public function getName(): string;
    public function getSize(): int;
}

class File implements FileSystemItem
{
    public function __construct(
        private string $name,
        private int $size
    ) {}

    public function getName(): string
    {
        return $this->name;
    }

    public function getSize(): int
    {
        return $this->size;
    }
}

class Directory implements FileSystemItem
{
    private array $items = [];

    public function __construct(
        private string $name
    ) {}

    public function add(FileSystemItem $item): void
    {
        $this->items[] = $item;
    }

    public function getName(): string
    {
        return $this->name;
    }

    public function getSize(): int
    {
        // Рекурсивно считаем размер всех дочерних элементов
        return array_reduce(
            $this->items,
            fn($total, $item) => $total + $item->getSize(),
            0
        );
    }
}

// Использование
$root = new Directory('root');

$documents = new Directory('documents');
$documents->add(new File('resume.pdf', 1000));
$documents->add(new File('cover_letter.pdf', 500));

$photos = new Directory('photos');
$photos->add(new File('vacation.jpg', 2000));
$photos->add(new File('family.jpg', 1500));

$root->add($documents);
$root->add($photos);
$root->add(new File('readme.txt', 100));

echo $root->getSize(); // 5100 (сумма всех файлов)
```

---

### 9. Decorator (Декоратор)

**Цель:** Динамическое добавление объектам новых обязанностей.

```php
<?php

interface Coffee
{
    public function getCost(): float;
    public function getDescription(): string;
}

class SimpleCoffee implements Coffee
{
    public function getCost(): float
    {
        return 2.0;
    }

    public function getDescription(): string
    {
        return "Simple coffee";
    }
}

// Базовый декоратор
abstract class CoffeeDecorator implements Coffee
{
    protected Coffee $coffee;

    public function __construct(Coffee $coffee)
    {
        $this->coffee = $coffee;
    }
}

class MilkDecorator extends CoffeeDecorator
{
    public function getCost(): float
    {
        return $this->coffee->getCost() + 0.5;
    }

    public function getDescription(): string
    {
        return $this->coffee->getDescription() . ", milk";
    }
}

class SugarDecorator extends CoffeeDecorator
{
    public function getCost(): float
    {
        return $this->coffee->getCost() + 0.2;
    }

    public function getDescription(): string
    {
        return $this->coffee->getDescription() . ", sugar";
    }
}

// Использование - оборачиваем декораторами
$coffee = new SimpleCoffee();
echo $coffee->getDescription() . ": $" . $coffee->getCost() . "\n";
// Simple coffee: $2

$coffee = new MilkDecorator($coffee);
echo $coffee->getDescription() . ": $" . $coffee->getCost() . "\n";
// Simple coffee, milk: $2.5

$coffee = new SugarDecorator($coffee);
echo $coffee->getDescription() . ": $" . $coffee->getCost() . "\n";
// Simple coffee, milk, sugar: $2.7
```

**В Laravel:**
```php
// Middleware - это Decorator для HTTP запросов
class LogMiddleware
{
    public function handle($request, Closure $next)
    {
        Log::info('Request: ' . $request->url());
        $response = $next($request); // Декорируем
        Log::info('Response: ' . $response->status());
        return $response;
    }
}
```

---

### 10. Facade (Фасад)

**Цель:** Предоставить простой интерфейс к сложной подсистеме.

```php
<?php

// Сложная подсистема
class VideoFile
{
    public function __construct(public string $filename) {}
}

class Codec {}
class MPEG4Codec extends Codec {}
class OggCodec extends Codec {}

class CodecFactory
{
    public static function extract(VideoFile $file): Codec
    {
        return str_ends_with($file->filename, '.mp4')
            ? new MPEG4Codec()
            : new OggCodec();
    }
}

class BitrateReader
{
    public static function read(VideoFile $file, Codec $codec): array
    {
        return ['bitrate' => 1000];
    }
}

class AudioMixer
{
    public function fix(array $result): array
    {
        return $result + ['audio' => 'fixed'];
    }
}

// Фасад - простой интерфейс
class VideoConverter
{
    public function convert(string $filename, string $format): array
    {
        $file = new VideoFile($filename);
        $sourceCodec = CodecFactory::extract($file);
        $destinationCodec = $format === 'mp4' ? new MPEG4Codec() : new OggCodec();
        $buffer = BitrateReader::read($file, $sourceCodec);
        $result = (new AudioMixer())->fix($buffer);
        return $result;
    }
}

// Использование - простой API вместо сложной подсистемы
$converter = new VideoConverter();
$result = $converter->convert('video.ogg', 'mp4');
```

**В Laravel:**
```php
// Laravel Facades - это Service Locator + Facade паттерны
use Illuminate\Support\Facades\Cache;

Cache::put('key', 'value', 60); // Простой API

// Вместо:
app('cache')->getStore()->put('key', 'value', 60);
```

---

### 11. Flyweight (Приспособленец)

**Цель:** Эффективная поддержка множества мелких объектов через разделение общего состояния.

```php
<?php

class TreeType
{
    public function __construct(
        private string $name,
        private string $color,
        private string $texture
    ) {}

    public function render(int $x, int $y): void
    {
        echo "Drawing {$this->name} at ({$x}, {$y})\n";
    }
}

// Фабрика Flyweight
class TreeFactory
{
    private static array $types = [];

    public static function getTreeType(string $name, string $color, string $texture): TreeType
    {
        $key = md5($name . $color . $texture);

        if (!isset(self::$types[$key])) {
            self::$types[$key] = new TreeType($name, $color, $texture);
        }

        return self::$types[$key];
    }

    public static function getCount(): int
    {
        return count(self::$types);
    }
}

class Tree
{
    public function __construct(
        private int $x,
        private int $y,
        private TreeType $type // Разделяемое состояние
    ) {}

    public function render(): void
    {
        $this->type->render($this->x, $this->y);
    }
}

// Использование
$forest = [];

// Создаем 1000 деревьев, но только 2 типа TreeType
for ($i = 0; $i < 1000; $i++) {
    $type = $i % 2 === 0
        ? TreeFactory::getTreeType('Oak', 'green', 'oak_texture.png')
        : TreeFactory::getTreeType('Pine', 'dark_green', 'pine_texture.png');

    $forest[] = new Tree(rand(0, 100), rand(0, 100), $type);
}

echo "Trees: " . count($forest) . "\n";  // 1000
echo "Tree types: " . TreeFactory::getCount() . "\n"; // 2 (экономия памяти!)
```

---

### 12. Proxy (Заместитель)

**Цель:** Предоставить суррогат или заместитель другого объекта для контроля доступа.

```php
<?php

interface Image
{
    public function display(): void;
}

class RealImage implements Image
{
    private string $filename;

    public function __construct(string $filename)
    {
        $this->filename = $filename;
        $this->loadFromDisk(); // Тяжелая операция
    }

    private function loadFromDisk(): void
    {
        echo "Loading image: {$this->filename}\n";
    }

    public function display(): void
    {
        echo "Displaying image: {$this->filename}\n";
    }
}

// Proxy - ленивая загрузка
class ProxyImage implements Image
{
    private ?RealImage $realImage = null;

    public function __construct(
        private string $filename
    ) {}

    public function display(): void
    {
        // Загрузка только когда реально нужно
        if ($this->realImage === null) {
            $this->realImage = new RealImage($this->filename);
        }
        $this->realImage->display();
    }
}

// Использование
$image = new ProxyImage('photo.jpg'); // Загрузка НЕ происходит
// ... другие операции ...
$image->display(); // Загрузка происходит здесь
```

**Виды Proxy:**
- **Virtual Proxy** - ленивая инициализация (пример выше)
- **Protection Proxy** - контроль доступа
- **Remote Proxy** - представитель удаленного объекта
- **Logging Proxy** - логирование вызовов

---

## Поведенческие паттерны (Behavioral)

Определяют алгоритмы и распределение обязанностей между объектами.

### 13. Chain of Responsibility (Цепочка обязанностей)

**Цель:** Избежать привязки отправителя запроса к получателю, дав возможность обработать запрос нескольким объектам.

```php
<?php

abstract class Handler
{
    private ?Handler $next = null;

    public function setNext(Handler $handler): Handler
    {
        $this->next = $handler;
        return $handler;
    }

    public function handle(array $request): ?array
    {
        if ($this->next) {
            return $this->next->handle($request);
        }
        return null;
    }
}

class AuthHandler extends Handler
{
    public function handle(array $request): ?array
    {
        if (empty($request['auth_token'])) {
            return ['error' => 'Unauthorized'];
        }

        echo "Auth check passed\n";
        return parent::handle($request);
    }
}

class ValidationHandler extends Handler
{
    public function handle(array $request): ?array
    {
        if (empty($request['data'])) {
            return ['error' => 'Invalid data'];
        }

        echo "Validation passed\n";
        return parent::handle($request);
    }
}

class ProcessHandler extends Handler
{
    public function handle(array $request): ?array
    {
        echo "Processing request\n";
        return ['success' => true, 'data' => $request['data']];
    }
}

// Использование
$chain = new AuthHandler();
$chain->setNext(new ValidationHandler())
      ->setNext(new ProcessHandler());

$result = $chain->handle([
    'auth_token' => 'abc123',
    'data' => ['user_id' => 1],
]);
```

**В Laravel:**
```php
// Middleware - это Chain of Responsibility
Route::get('/dashboard', function () {
    //
})->middleware(['auth', 'verified', 'admin']);
```

---

### 14. Command (Команда)

**Цель:** Инкапсулировать запрос как объект, позволяя параметризовать клиентов с различными запросами, ставить запросы в очередь, логировать их.

```php
<?php

// Получатель
class Light
{
    public function turnOn(): void
    {
        echo "Light is ON\n";
    }

    public function turnOff(): void
    {
        echo "Light is OFF\n";
    }
}

// Интерфейс команды
interface Command
{
    public function execute(): void;
    public function undo(): void;
}

// Конкретные команды
class TurnOnCommand implements Command
{
    private Light $light;

    public function __construct(Light $light)
    {
        $this->light = $light;
    }

    public function execute(): void
    {
        $this->light->turnOn();
    }

    public function undo(): void
    {
        $this->light->turnOff();
    }
}

class TurnOffCommand implements Command
{
    private Light $light;

    public function __construct(Light $light)
    {
        $this->light = $light;
    }

    public function execute(): void
    {
        $this->light->turnOff();
    }

    public function undo(): void
    {
        $this->light->turnOn();
    }
}

// Invoker (вызывающий)
class RemoteControl
{
    private array $history = [];

    public function submit(Command $command): void
    {
        $command->execute();
        $this->history[] = $command;
    }

    public function undo(): void
    {
        if (empty($this->history)) {
            return;
        }

        $command = array_pop($this->history);
        $command->undo();
    }
}

// Использование
$light = new Light();
$remote = new RemoteControl();

$remote->submit(new TurnOnCommand($light));  // Light is ON
$remote->submit(new TurnOffCommand($light)); // Light is OFF
$remote->undo(); // Light is ON (отмена последней команды)
```

**В Laravel:**
```php
// Artisan Commands - это Command паттерн
php artisan make:command SendEmails

class SendEmails extends Command
{
    protected $signature = 'emails:send';

    public function handle()
    {
        // Выполнение команды
    }
}
```

---

### 15. Iterator (Итератор)

**Цель:** Предоставить способ последовательного доступа к элементам коллекции без раскрытия внутреннего представления.

```php
<?php

class Book
{
    public function __construct(
        public string $title,
        public string $author
    ) {}
}

class BookCollection implements \Iterator
{
    private array $books = [];
    private int $position = 0;

    public function addBook(Book $book): void
    {
        $this->books[] = $book;
    }

    // Iterator methods
    public function current(): Book
    {
        return $this->books[$this->position];
    }

    public function key(): int
    {
        return $this->position;
    }

    public function next(): void
    {
        $this->position++;
    }

    public function rewind(): void
    {
        $this->position = 0;
    }

    public function valid(): bool
    {
        return isset($this->books[$this->position]);
    }
}

// Использование
$collection = new BookCollection();
$collection->addBook(new Book('1984', 'George Orwell'));
$collection->addBook(new Book('Brave New World', 'Aldous Huxley'));

foreach ($collection as $book) {
    echo "{$book->title} by {$book->author}\n";
}
```

**С генераторами (modern PHP):**
```php
class BookCollection
{
    private array $books = [];

    public function addBook(Book $book): void
    {
        $this->books[] = $book;
    }

    public function getIterator(): \Generator
    {
        foreach ($this->books as $book) {
            yield $book;
        }
    }
}

// Использование
foreach ($collection->getIterator() as $book) {
    echo "{$book->title}\n";
}
```

---

### 16. Mediator (Посредник)

**Цель:** Определить объект, инкапсулирующий способ взаимодействия множества объектов.

```php
<?php

interface Mediator
{
    public function notify(object $sender, string $event): void;
}

class ChatRoom implements Mediator
{
    private array $users = [];

    public function addUser(User $user): void
    {
        $this->users[$user->name] = $user;
        $user->setMediator($this);
    }

    public function notify(object $sender, string $event): void
    {
        if ($event === 'send_message') {
            // Отправить сообщение всем кроме отправителя
            foreach ($this->users as $user) {
                if ($user !== $sender) {
                    $user->receive($sender->name, $sender->message);
                }
            }
        }
    }
}

class User
{
    private ?Mediator $mediator = null;
    public string $message = '';

    public function __construct(
        public string $name
    ) {}

    public function setMediator(Mediator $mediator): void
    {
        $this->mediator = $mediator;
    }

    public function send(string $message): void
    {
        $this->message = $message;
        echo "{$this->name} sends: {$message}\n";
        $this->mediator->notify($this, 'send_message');
    }

    public function receive(string $from, string $message): void
    {
        echo "{$this->name} receives from {$from}: {$message}\n";
    }
}

// Использование
$chatRoom = new ChatRoom();

$alice = new User('Alice');
$bob = new User('Bob');
$charlie = new User('Charlie');

$chatRoom->addUser($alice);
$chatRoom->addUser($bob);
$chatRoom->addUser($charlie);

$alice->send('Hello everyone!');
// Bob receives from Alice: Hello everyone!
// Charlie receives from Alice: Hello everyone!
```

---

### 17. Memento (Снимок)

**Цель:** Сохранить и восстановить внутреннее состояние объекта без нарушения инкапсуляции.

```php
<?php

// Memento
class EditorMemento
{
    public function __construct(
        private string $content,
        private int $cursorPosition
    ) {}

    public function getContent(): string
    {
        return $this->content;
    }

    public function getCursorPosition(): int
    {
        return $this->cursorPosition;
    }
}

// Originator
class Editor
{
    private string $content = '';
    private int $cursorPosition = 0;

    public function type(string $text): void
    {
        $this->content .= $text;
        $this->cursorPosition += strlen($text);
    }

    public function save(): EditorMemento
    {
        return new EditorMemento($this->content, $this->cursorPosition);
    }

    public function restore(EditorMemento $memento): void
    {
        $this->content = $memento->getContent();
        $this->cursorPosition = $memento->getCursorPosition();
    }

    public function getContent(): string
    {
        return $this->content;
    }
}

// Caretaker
class History
{
    private array $mementos = [];

    public function push(EditorMemento $memento): void
    {
        $this->mementos[] = $memento;
    }

    public function pop(): ?EditorMemento
    {
        return array_pop($this->mementos);
    }
}

// Использование (Undo/Redo)
$editor = new Editor();
$history = new History();

$editor->type('Hello ');
$history->push($editor->save()); // Сохранили

$editor->type('World');
$history->push($editor->save()); // Сохранили

$editor->type('!!!');
echo $editor->getContent() . "\n"; // Hello World!!!

// Undo
$editor->restore($history->pop());
echo $editor->getContent() . "\n"; // Hello World

// Undo again
$editor->restore($history->pop());
echo $editor->getContent() . "\n"; // Hello
```

---

### 18. Observer (Наблюдатель)

**Цель:** Определить зависимость "один ко многим" между объектами так, что при изменении состояния одного объекта все зависящие от него оповещаются.

```php
<?php

interface Observer
{
    public function update(string $event, array $data): void;
}

interface Subject
{
    public function attach(Observer $observer): void;
    public function detach(Observer $observer): void;
    public function notify(string $event, array $data): void;
}

class User implements Subject
{
    private array $observers = [];
    private string $email;

    public function attach(Observer $observer): void
    {
        $this->observers[] = $observer;
    }

    public function detach(Observer $observer): void
    {
        $this->observers = array_filter(
            $this->observers,
            fn($obs) => $obs !== $observer
        );
    }

    public function notify(string $event, array $data): void
    {
        foreach ($this->observers as $observer) {
            $observer->update($event, $data);
        }
    }

    public function changeEmail(string $email): void
    {
        $this->email = $email;
        $this->notify('email_changed', ['email' => $email]);
    }
}

class EmailLogger implements Observer
{
    public function update(string $event, array $data): void
    {
        if ($event === 'email_changed') {
            echo "LOG: Email changed to {$data['email']}\n";
        }
    }
}

class EmailNotifier implements Observer
{
    public function update(string $event, array $data): void
    {
        if ($event === 'email_changed') {
            echo "NOTIFICATION: Sending confirmation to {$data['email']}\n";
        }
    }
}

// Использование
$user = new User();
$user->attach(new EmailLogger());
$user->attach(new EmailNotifier());

$user->changeEmail('new@example.com');
// LOG: Email changed to new@example.com
// NOTIFICATION: Sending confirmation to new@example.com
```

**В Laravel:**
```php
// Events & Listeners - это Observer паттерн
class UserRegistered
{
    public function __construct(public User $user) {}
}

class SendWelcomeEmail
{
    public function handle(UserRegistered $event)
    {
        Mail::to($event->user)->send(new WelcomeEmail());
    }
}

// В EventServiceProvider
protected $listen = [
    UserRegistered::class => [
        SendWelcomeEmail::class,
        LogRegistration::class,
        UpdateStatistics::class,
    ],
];
```

---

### 19. State (Состояние)

**Цель:** Позволить объекту изменять поведение в зависимости от внутреннего состояния.

```php
<?php

interface OrderState
{
    public function proceedToNext(Order $order): void;
    public function toString(): string;
}

class PendingState implements OrderState
{
    public function proceedToNext(Order $order): void
    {
        echo "Processing payment...\n";
        $order->setState(new PaidState());
    }

    public function toString(): string
    {
        return 'pending';
    }
}

class PaidState implements OrderState
{
    public function proceedToNext(Order $order): void
    {
        echo "Shipping order...\n";
        $order->setState(new ShippedState());
    }

    public function toString(): string
    {
        return 'paid';
    }
}

class ShippedState implements OrderState
{
    public function proceedToNext(Order $order): void
    {
        echo "Order delivered!\n";
        $order->setState(new DeliveredState());
    }

    public function toString(): string
    {
        return 'shipped';
    }
}

class DeliveredState implements OrderState
{
    public function proceedToNext(Order $order): void
    {
        echo "Order is already delivered.\n";
    }

    public function toString(): string
    {
        return 'delivered';
    }
}

class Order
{
    private OrderState $state;

    public function __construct()
    {
        $this->state = new PendingState();
    }

    public function setState(OrderState $state): void
    {
        $this->state = $state;
        echo "State changed to: {$this->state->toString()}\n";
    }

    public function proceed(): void
    {
        $this->state->proceedToNext($this);
    }
}

// Использование
$order = new Order();
$order->proceed(); // Processing payment... State changed to: paid
$order->proceed(); // Shipping order... State changed to: shipped
$order->proceed(); // Order delivered! State changed to: delivered
$order->proceed(); // Order is already delivered.
```

---

### 20. Strategy (Стратегия)

**Цель:** Определить семейство алгоритмов, инкапсулировать каждый и сделать их взаимозаменяемыми.

```php
<?php

interface PaymentStrategy
{
    public function pay(float $amount): void;
}

class CreditCardStrategy implements PaymentStrategy
{
    private string $cardNumber;

    public function __construct(string $cardNumber)
    {
        $this->cardNumber = $cardNumber;
    }

    public function pay(float $amount): void
    {
        echo "Paid {$amount} using Credit Card {$this->cardNumber}\n";
    }
}

class PayPalStrategy implements PaymentStrategy
{
    private string $email;

    public function __construct(string $email)
    {
        $this->email = $email;
    }

    public function pay(float $amount): void
    {
        echo "Paid {$amount} using PayPal account {$this->email}\n";
    }
}

class CryptoStrategy implements PaymentStrategy
{
    private string $walletAddress;

    public function __construct(string $walletAddress)
    {
        $this->walletAddress = $walletAddress;
    }

    public function pay(float $amount): void
    {
        echo "Paid {$amount} using Crypto wallet {$this->walletAddress}\n";
    }
}

class ShoppingCart
{
    private PaymentStrategy $paymentStrategy;

    public function setPaymentStrategy(PaymentStrategy $strategy): void
    {
        $this->paymentStrategy = $strategy;
    }

    public function checkout(float $amount): void
    {
        $this->paymentStrategy->pay($amount);
    }
}

// Использование - можем менять стратегию на лету
$cart = new ShoppingCart();

$cart->setPaymentStrategy(new CreditCardStrategy('1234-5678-9012-3456'));
$cart->checkout(100); // Paid 100 using Credit Card

$cart->setPaymentStrategy(new PayPalStrategy('user@example.com'));
$cart->checkout(50); // Paid 50 using PayPal

$cart->setPaymentStrategy(new CryptoStrategy('0x123abc...'));
$cart->checkout(200); // Paid 200 using Crypto
```

**В Laravel:**
```php
// Cache drivers - это Strategy паттерн
Cache::store('redis')->put('key', 'value', 60);
Cache::store('file')->put('key', 'value', 60);
Cache::store('database')->put('key', 'value', 60);

// Можем менять стратегию в runtime
$driver = config('cache.default'); // redis, file, database
Cache::store($driver)->put('key', 'value', 60);
```

---

### 21. Template Method (Шаблонный метод)

**Цель:** Определить скелет алгоритма в методе, оставляя определение некоторых шагов подклассам.

```php
<?php

abstract class DataParser
{
    // Template Method - определяет скелет алгоритма
    public function parse(string $filename): array
    {
        $data = $this->readFile($filename);
        $parsedData = $this->parseData($data);
        $validatedData = $this->validateData($parsedData);
        return $this->transformData($validatedData);
    }

    // Общие методы
    protected function readFile(string $filename): string
    {
        return file_get_contents($filename);
    }

    protected function validateData(array $data): array
    {
        // Базовая валидация
        return array_filter($data, fn($row) => !empty($row));
    }

    // Абстрактные методы - должны быть реализованы в подклассах
    abstract protected function parseData(string $data): array;
    abstract protected function transformData(array $data): array;
}

class CsvParser extends DataParser
{
    protected function parseData(string $data): array
    {
        $lines = explode("\n", $data);
        return array_map(fn($line) => str_getcsv($line), $lines);
    }

    protected function transformData(array $data): array
    {
        // CSV специфичная трансформация
        return array_map(fn($row) => array_map('trim', $row), $data);
    }
}

class JsonParser extends DataParser
{
    protected function parseData(string $data): array
    {
        return json_decode($data, true);
    }

    protected function transformData(array $data): array
    {
        // JSON специфичная трансформация
        return array_map(fn($row) => array_change_key_case($row, CASE_LOWER), $data);
    }
}

// Использование
$csvParser = new CsvParser();
$csvData = $csvParser->parse('data.csv');

$jsonParser = new JsonParser();
$jsonData = $jsonParser->parse('data.json');
```

**В Laravel:**
```php
// Eloquent Model methods - Template Method
class User extends Model
{
    // Template method
    public function save(array $options = [])
    {
        // 1. Fire "saving" event
        // 2. Validate
        // 3. Save to DB
        // 4. Fire "saved" event
    }

    // Hooks (можем переопределить)
    protected static function boot()
    {
        parent::boot();

        static::creating(function ($user) {
            // Custom logic before creating
        });
    }
}
```

---

### 22. Visitor (Посетитель)

**Цель:** Описать операцию, выполняемую над каждым элементом структуры, не изменяя классы этих элементов.

```php
<?php

interface Shape
{
    public function accept(Visitor $visitor): void;
}

class Circle implements Shape
{
    public function __construct(
        public float $radius
    ) {}

    public function accept(Visitor $visitor): void
    {
        $visitor->visitCircle($this);
    }
}

class Rectangle implements Shape
{
    public function __construct(
        public float $width,
        public float $height
    ) {}

    public function accept(Visitor $visitor): void
    {
        $visitor->visitRectangle($this);
    }
}

interface Visitor
{
    public function visitCircle(Circle $circle): void;
    public function visitRectangle(Rectangle $rectangle): void;
}

// Посетитель для расчета площади
class AreaCalculator implements Visitor
{
    private float $totalArea = 0;

    public function visitCircle(Circle $circle): void
    {
        $area = pi() * $circle->radius ** 2;
        echo "Circle area: {$area}\n";
        $this->totalArea += $area;
    }

    public function visitRectangle(Rectangle $rectangle): void
    {
        $area = $rectangle->width * $rectangle->height;
        echo "Rectangle area: {$area}\n";
        $this->totalArea += $area;
    }

    public function getTotalArea(): float
    {
        return $this->totalArea;
    }
}

// Посетитель для экспорта в JSON
class JsonExporter implements Visitor
{
    private array $data = [];

    public function visitCircle(Circle $circle): void
    {
        $this->data[] = [
            'type' => 'circle',
            'radius' => $circle->radius,
        ];
    }

    public function visitRectangle(Rectangle $rectangle): void
    {
        $this->data[] = [
            'type' => 'rectangle',
            'width' => $rectangle->width,
            'height' => $rectangle->height,
        ];
    }

    public function export(): string
    {
        return json_encode($this->data, JSON_PRETTY_PRINT);
    }
}

// Использование
$shapes = [
    new Circle(5),
    new Rectangle(10, 20),
    new Circle(3),
];

// Разные операции с теми же объектами
$areaCalc = new AreaCalculator();
foreach ($shapes as $shape) {
    $shape->accept($areaCalc);
}
echo "Total area: " . $areaCalc->getTotalArea() . "\n";

$jsonExporter = new JsonExporter();
foreach ($shapes as $shape) {
    $shape->accept($jsonExporter);
}
echo $jsonExporter->export();
```

---

### 23. Interpreter (Интерпретатор)

**Цель:** Для заданного языка определить представление его грамматики и интерпретатор предложений.

```php
<?php

// Интерфейс выражения
interface Expression
{
    public function interpret(array $context): bool;
}

// Терминальные выражения
class VariableExpression implements Expression
{
    private string $name;

    public function __construct(string $name)
    {
        $this->name = $name;
    }

    public function interpret(array $context): bool
    {
        return $context[$this->name] ?? false;
    }
}

// Нетерминальные выражения
class AndExpression implements Expression
{
    private Expression $left;
    private Expression $right;

    public function __construct(Expression $left, Expression $right)
    {
        $this->left = $left;
        $this->right = $right;
    }

    public function interpret(array $context): bool
    {
        return $this->left->interpret($context) && $this->right->interpret($context);
    }
}

class OrExpression implements Expression
{
    private Expression $left;
    private Expression $right;

    public function __construct(Expression $left, Expression $right)
    {
        $this->left = $left;
        $this->right = $right;
    }

    public function interpret(array $context): bool
    {
        return $this->left->interpret($context) || $this->right->interpret($context);
    }
}

class NotExpression implements Expression
{
    private Expression $expression;

    public function __construct(Expression $expression)
    {
        $this->expression = $expression;
    }

    public function interpret(array $context): bool
    {
        return !$this->expression->interpret($context);
    }
}

// Использование - интерпретация булевых выражений
// Правило: (isAdmin AND isActive) OR isSuperUser

$isAdmin = new VariableExpression('isAdmin');
$isActive = new VariableExpression('isActive');
$isSuperUser = new VariableExpression('isSuperUser');

$rule = new OrExpression(
    new AndExpression($isAdmin, $isActive),
    $isSuperUser
);

// Тест 1: обычный активный админ
$context1 = [
    'isAdmin' => true,
    'isActive' => true,
    'isSuperUser' => false,
];
var_dump($rule->interpret($context1)); // true

// Тест 2: неактивный админ
$context2 = [
    'isAdmin' => true,
    'isActive' => false,
    'isSuperUser' => false,
];
var_dump($rule->interpret($context2)); // false

// Тест 3: суперпользователь (не админ, неактивен)
$context3 = [
    'isAdmin' => false,
    'isActive' => false,
    'isSuperUser' => true,
];
var_dump($rule->interpret($context3)); // true
```

**В Laravel:**
```php
// Validation rules - это Interpreter паттерн
$validator = Validator::make($request->all(), [
    'email' => 'required|email|max:255',
    'age' => 'required|integer|min:18|max:100',
]);

// Laravel интерпретирует эти правила
```

---

## Итоговая таблица паттернов

### Порождающие (Creational)
| Паттерн | Назначение |
|---------|-----------|
| Singleton | Один экземпляр класса |
| Factory Method | Делегирование создания объектов |
| Abstract Factory | Семейства связанных объектов |
| Builder | Пошаговое конструирование сложных объектов |
| Prototype | Клонирование объектов |

### Структурные (Structural)
| Паттерн | Назначение |
|---------|-----------|
| Adapter | Совместимость интерфейсов |
| Bridge | Разделение абстракции и реализации |
| Composite | Древовидные структуры |
| Decorator | Динамическое добавление обязанностей |
| Facade | Упрощенный интерфейс к подсистеме |
| Flyweight | Разделение состояния для экономии памяти |
| Proxy | Заместитель объекта |

### Поведенческие (Behavioral)
| Паттерн | Назначение |
|---------|-----------|
| Chain of Responsibility | Цепочка обработчиков |
| Command | Инкапсуляция запроса как объекта |
| Iterator | Последовательный доступ к элементам |
| Mediator | Централизация взаимодействия объектов |
| Memento | Сохранение и восстановление состояния |
| Observer | Уведомление зависимых объектов |
| State | Изменение поведения в зависимости от состояния |
| Strategy | Взаимозаменяемые алгоритмы |
| Template Method | Скелет алгоритма с переопределяемыми шагами |
| Visitor | Операции над элементами структуры |
| Interpreter | Интерпретация языка |

---

## Когда использовать?

### Признаки необходимости паттерна

**Singleton:**
- Нужен глобальный доступ к ресурсу
- Должен быть только один экземпляр (DB connection, Logger)

**Factory Method / Abstract Factory:**
- Не знаем заранее, какие объекты создавать
- Создание объектов зависит от конфигурации/условий

**Builder:**
- Объект имеет много опциональных параметров
- Нужен fluent interface

**Adapter:**
- Несовместимые интерфейсы сторонних библиотек
- Нужно использовать старый код в новой системе

**Decorator:**
- Нужно добавлять функциональность динамически
- Наследование неудобно

**Strategy:**
- Много похожих классов с разным поведением
- Нужна возможность менять алгоритм в runtime

**Observer:**
- Изменение одного объекта требует изменения других
- Количество наблюдателей неизвестно заранее

**Command:**
- Нужно параметризовать объекты действиями
- Нужна очередь операций или Undo/Redo

---

## Anti-patterns (когда НЕ использовать)

### Over-engineering
```php
// ❌ ПЛОХО: Паттерн для простой задачи
class UserNameFormatterFactory
{
    public function create(): UserNameFormatter
    {
        return new UserNameFormatter();
    }
}

// ✅ ХОРОШО: Просто функция
function formatUserName(string $name): string
{
    return ucwords($name);
}
```

### Cargo Cult Programming
Не используй паттерн только потому, что "так надо" или "это best practice". Используй когда он решает реальную проблему.

### God Object через Singleton
```php
// ❌ ПЛОХО: Singleton превращается в God Object
class App
{
    public Database $db;
    public Logger $logger;
    public Cache $cache;
    public Mailer $mailer;
    // ... еще 20 зависимостей
}

// ✅ ХОРОШО: Dependency Injection
class UserService
{
    public function __construct(
        private Database $db,
        private Logger $logger
    ) {}
}
```

---

## Best Practices

1. **Предпочитай композицию наследованию** - большинство паттернов основаны на композиции
2. **Программируй к интерфейсу, а не к реализации** - используй интерфейсы для гибкости
3. **Принцип открытости/закрытости** - открыт для расширения, закрыт для изменения
4. **Не изобретай велосипед** - используй готовые реализации (Laravel Service Container, Events, etc.)
5. **Keep It Simple** - не используй паттерн если простое решение работает

**Помни:** Паттерны - это инструменты, а не цель. Используй их когда они упрощают код, а не усложняют!

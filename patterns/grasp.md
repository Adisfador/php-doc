# GRASP Principles - General Responsibility Assignment Software Patterns

GRASP - это набор из 9 принципов для присвоения ответственности классам и объектам в ООП. Созданы Craig Larman.

---

## 1. Information Expert (Информационный эксперт)

**Принцип**: Назначай ответственность классу, который обладает необходимой информацией для выполнения этой ответственности.

### Проблема
Кто должен вычислять общую стоимость заказа?

### Решение
Класс Order, потому что у него есть вся информация о товарах.

```php
<?php

// ❌ ПЛОХО: OrderService знает слишком много
class OrderService
{
    public function calculateTotal(array $items)
    {
        $total = 0;
        foreach ($items as $item) {
            $total += $item['price'] * $item['quantity'];
        }
        return $total;
    }
}

// ✅ ХОРОШО: Order сам знает как вычислить свой total
class OrderItem
{
    public function __construct(
        private float $price,
        private int $quantity
    ) {}

    public function getSubtotal(): float
    {
        return $this->price * $this->quantity; // Information Expert!
    }
}

class Order
{
    private array $items = [];

    public function addItem(OrderItem $item): void
    {
        $this->items[] = $item;
    }

    public function getTotal(): float
    {
        // Order - эксперт, у него есть items
        return array_reduce(
            $this->items,
            fn($total, $item) => $total + $item->getSubtotal(),
            0
        );
    }
}

// Использование
$order = new Order();
$order->addItem(new OrderItem(price: 100, quantity: 2));
$order->addItem(new OrderItem(price: 50, quantity: 1));
echo $order->getTotal(); // 250
```

---

## 2. Creator (Создатель)

**Принцип**: Класс B должен создавать экземпляры класса A, если:
- B содержит или агрегирует A
- B записывает A
- B близко использует A
- B имеет данные инициализации для A

### Пример

```php
<?php

// ❌ ПЛОХО: Order создается где попало
class OrderController
{
    public function store(Request $request)
    {
        $order = new Order();
        $order->setUserId($request->user()->id);
        $order->save();
        
        foreach ($request->items as $itemData) {
            $item = new OrderItem(); // Кто должен создавать OrderItem?
            $item->setOrderId($order->id);
            $item->setProductId($itemData['product_id']);
            $item->save();
        }
    }
}

// ✅ ХОРОШО: Order создает свои OrderItem (Creator!)
class Order extends Model
{
    public function addProduct(Product $product, int $quantity): OrderItem
    {
        // Order - создатель OrderItem, потому что содержит items
        return $this->items()->create([
            'product_id' => $product->id,
            'price' => $product->price,
            'quantity' => $quantity,
        ]);
    }
}

class OrderController
{
    public function store(Request $request)
    {
        $order = $request->user()->orders()->create(); // User - создатель Order
        
        foreach ($request->items as $itemData) {
            $product = Product::find($itemData['product_id']);
            $order->addProduct($product, $itemData['quantity']); // Order - создатель OrderItem
        }
    }
}
```

---

## 3. Controller (Контроллер)

**Принцип**: Назначь ответственность за обработку системных событий классу, представляющему:
- Всю систему (facade controller)
- Сценарий использования (use case controller)
- Сессию (session controller)

### Пример

```php
<?php

// ❌ ПЛОХО: Бизнес-логика в HTTP контроллере
class UserController extends Controller
{
    public function register(Request $request)
    {
        // Валидация
        $validated = $request->validate([...]);
        
        // Создание пользователя
        $user = User::create([
            'name' => $validated['name'],
            'email' => $validated['email'],
            'password' => bcrypt($validated['password']),
        ]);
        
        // Создание профиля
        $user->profile()->create([...]);
        
        // Отправка email
        Mail::to($user)->send(new WelcomeEmail());
        
        // Логирование
        Log::info("User registered: {$user->email}");
        
        return response()->json($user);
    }
}

// ✅ ХОРОШО: Use Case контроллер (Application Service)
class RegisterUserService
{
    public function __construct(
        private UserRepository $users,
        private Mailer $mailer,
        private Logger $logger
    ) {}

    public function execute(RegisterUserDTO $data): User
    {
        DB::transaction(function () use ($data) {
            $user = $this->users->create(
                name: $data->name,
                email: $data->email,
                password: $data->password
            );
            
            $user->profile()->create($data->profileData);
            
            $this->mailer->sendWelcome($user);
            $this->logger->logRegistration($user);
            
            return $user;
        });
    }
}

// HTTP контроллер делегирует Use Case контроллеру
class UserController extends Controller
{
    public function __construct(
        private RegisterUserService $registerUser
    ) {}

    public function register(RegisterUserRequest $request)
    {
        $user = $this->registerUser->execute(
            RegisterUserDTO::fromRequest($request)
        );
        
        return response()->json($user);
    }
}
```

---

## 4. Low Coupling (Низкое зацепление)

**Принцип**: Минимизируй зависимости между классами. Класс не должен зависеть от деталей реализации других классов.

### Пример

```php
<?php

// ❌ ПЛОХО: Высокое зацепление - зависит от конкретных классов
class OrderProcessor
{
    public function process(Order $order)
    {
        // Зависит от конкретного PayPal
        $paypal = new PayPalGateway();
        $paypal->charge($order->total);
        
        // Зависит от конкретного SMTP
        $mailer = new SmtpMailer();
        $mailer->send($order->user->email, 'Order confirmed');
        
        // Зависит от конкретного MySQL
        $db = new MySQLDatabase();
        $db->save($order);
    }
}

// ✅ ХОРОШО: Низкое зацепление - зависит от интерфейсов
interface PaymentGateway
{
    public function charge(float $amount): bool;
}

interface Mailer
{
    public function send(string $to, string $subject, string $body): void;
}

interface OrderRepository
{
    public function save(Order $order): void;
}

class OrderProcessor
{
    public function __construct(
        private PaymentGateway $payment,
        private Mailer $mailer,
        private OrderRepository $orders
    ) {}

    public function process(Order $order): void
    {
        $this->payment->charge($order->total);
        $this->mailer->send($order->user->email, 'Order confirmed', '...');
        $this->orders->save($order);
    }
}

// Можем легко заменить реализации
$processor = new OrderProcessor(
    payment: new StripeGateway(), // Вместо PayPal
    mailer: new SendGridMailer(), // Вместо SMTP
    orders: new EloquentOrderRepository() // Вместо прямого MySQL
);
```

---

## 5. High Cohesion (Высокая связность)

**Принцип**: Методы класса должны быть связаны общей ответственностью. Класс не должен делать слишком много разного.

### Пример

```php
<?php

// ❌ ПЛОХО: Низкая связность - класс делает все
class User extends Model
{
    // Работа с БД
    public function save() { }
    
    // Валидация
    public function validateEmail() { }
    
    // Отправка email
    public function sendWelcomeEmail() { }
    
    // Генерация PDF
    public function generateReport() { }
    
    // Работа с API
    public function syncWithCRM() { }
    
    // Логирование
    public function logActivity() { }
}

// ✅ ХОРОШО: Высокая связность - каждый класс делает одно
class User extends Model
{
    // Только данные пользователя
    public function getFullName(): string
    {
        return "{$this->first_name} {$this->last_name}";
    }
}

class UserValidator
{
    // Только валидация
    public function validateEmail(string $email): bool { }
    public function validatePassword(string $password): bool { }
}

class UserMailer
{
    // Только отправка email
    public function sendWelcome(User $user): void { }
    public function sendPasswordReset(User $user): void { }
}

class UserReportGenerator
{
    // Только генерация отчетов
    public function generatePDF(User $user): string { }
}

class UserCRMSynchronizer
{
    // Только синхронизация с CRM
    public function sync(User $user): void { }
}
```

---

## 6. Polymorphism (Полиморфизм)

**Принцип**: Используй полиморфизм для обработки альтернатив, основанных на типе. Избегай if/switch по типам.

### Пример

```php
<?php

// ❌ ПЛОХО: Switch по типам
class PaymentProcessor
{
    public function process(Payment $payment)
    {
        switch ($payment->type) {
            case 'credit_card':
                // Логика для кредитки
                $this->chargeCreditCard($payment);
                break;
            case 'paypal':
                // Логика для PayPal
                $this->chargePayPal($payment);
                break;
            case 'bitcoin':
                // Логика для Bitcoin
                $this->chargeBitcoin($payment);
                break;
        }
    }
}

// ✅ ХОРОШО: Полиморфизм
interface PaymentMethod
{
    public function charge(float $amount): PaymentResult;
}

class CreditCardPayment implements PaymentMethod
{
    public function charge(float $amount): PaymentResult
    {
        // Логика для кредитки
        return new PaymentResult(success: true);
    }
}

class PayPalPayment implements PaymentMethod
{
    public function charge(float $amount): PaymentResult
    {
        // Логика для PayPal
        return new PaymentResult(success: true);
    }
}

class BitcoinPayment implements PaymentMethod
{
    public function charge(float $amount): PaymentResult
    {
        // Логика для Bitcoin
        return new PaymentResult(success: true);
    }
}

class PaymentProcessor
{
    public function process(PaymentMethod $method, float $amount): PaymentResult
    {
        return $method->charge($amount); // Полиморфизм!
    }
}

// Использование
$processor = new PaymentProcessor();
$processor->process(new CreditCardPayment(), 100);
$processor->process(new PayPalPayment(), 100);
```

---

## 7. Pure Fabrication (Чистая выдумка)

**Принцип**: Создай класс, который не представляет концепцию предметной области, если это помогает достичь Low Coupling и High Cohesion.

### Пример

```php
<?php

// Проблема: Сохранение Order в БД нарушает High Cohesion модели Order

// ❌ ПЛОХО: Order знает о БД
class Order extends Model
{
    public function save()
    {
        // Order знает SQL, MySQL, структуру БД...
        $sql = "INSERT INTO orders (user_id, total) VALUES (?, ?)";
        DB::execute($sql, [$this->user_id, $this->total]);
    }
}

// ✅ ХОРОШО: Pure Fabrication - OrderRepository
// Это не концепция предметной области, а технический класс

interface OrderRepository
{
    public function save(Order $order): void;
    public function find(int $id): ?Order;
    public function findByUser(User $user): Collection;
}

class EloquentOrderRepository implements OrderRepository
{
    public function save(Order $order): void
    {
        // Вся логика БД инкапсулирована здесь
        $order->save();
    }
    
    public function find(int $id): ?Order
    {
        return Order::find($id);
    }
    
    public function findByUser(User $user): Collection
    {
        return $user->orders()->get();
    }
}

// Order теперь чистая доменная модель
class Order
{
    public function __construct(
        private User $user,
        private Collection $items
    ) {}
    
    public function getTotal(): float
    {
        return $this->items->sum(fn($item) => $item->getSubtotal());
    }
    
    // Никакой логики БД!
}

// Другие примеры Pure Fabrication:
// - Logger (не доменный объект)
// - DatabaseConnection
// - EmailService
// - CacheManager
```

---

## 8. Indirection (Посредник)

**Принцип**: Используй промежуточный объект для уменьшения прямой связи между компонентами.

### Пример

```php
<?php

// ❌ ПЛОХО: Прямая зависимость
class OrderController
{
    public function store(Request $request)
    {
        // Контроллер напрямую зависит от конкретной БД
        $order = new Order();
        $order->user_id = $request->user()->id;
        $order->save(); // Прямая связь с Eloquent/БД
        
        // Напрямую зависит от конкретного email сервиса
        Mail::to($order->user)->send(new OrderConfirmation($order));
    }
}

// ✅ ХОРОШО: Indirection - используем посредников
interface OrderService
{
    public function createOrder(User $user, array $items): Order;
}

class CreateOrderService implements OrderService
{
    public function __construct(
        private OrderRepository $orders,  // Посредник для БД
        private NotificationService $notifier  // Посредник для уведомлений
    ) {}

    public function createOrder(User $user, array $items): Order
    {
        $order = new Order($user, $items);
        
        $this->orders->save($order); // Через посредника
        $this->notifier->orderCreated($order); // Через посредника
        
        return $order;
    }
}

class OrderController
{
    public function __construct(
        private OrderService $orderService // Контроллер не знает о БД и email
    ) {}

    public function store(Request $request)
    {
        $order = $this->orderService->createOrder(
            $request->user(),
            $request->items
        );
        
        return response()->json($order);
    }
}

// Другие примеры Indirection:
// - Adapter паттерн
// - Facade паттерн
// - Proxy паттерн
// - Mediator паттерн
```

---

## 9. Protected Variations (Защита от изменений)

**Принцип**: Определи точки вероятных изменений и создай стабильный интерфейс вокруг них.

### Пример

```php
<?php

// Точка изменений: способ оплаты может меняться

// ❌ ПЛОХО: Нет защиты
class OrderService
{
    public function processPayment(Order $order)
    {
        // Если захотим добавить Stripe - придется менять этот код
        $paypal = new PayPalAPI();
        $result = $paypal->createPayment(
            $order->total,
            $order->currency,
            $order->user->email
        );
        
        if ($result->status === 'completed') {
            $order->status = 'paid';
        }
    }
}

// ✅ ХОРОШО: Protected Variations
// Стабильный интерфейс защищает от изменений

interface PaymentGateway
{
    public function process(Payment $payment): PaymentResult;
}

class PayPalGateway implements PaymentGateway
{
    public function process(Payment $payment): PaymentResult
    {
        $api = new PayPalAPI();
        $result = $api->createPayment(
            $payment->amount,
            $payment->currency,
            $payment->email
        );
        
        return new PaymentResult(
            success: $result->status === 'completed',
            transactionId: $result->id
        );
    }
}

class StripeGateway implements PaymentGateway
{
    public function process(Payment $payment): PaymentResult
    {
        // Другая реализация, но тот же интерфейс
        $stripe = new StripeClient();
        $intent = $stripe->paymentIntents->create([...]);
        
        return new PaymentResult(
            success: $intent->status === 'succeeded',
            transactionId: $intent->id
        );
    }
}

class OrderService
{
    public function __construct(
        private PaymentGateway $gateway // Защищены от изменений
    ) {}

    public function processPayment(Order $order): void
    {
        // Код НЕ изменится при добавлении новых платежных систем
        $payment = Payment::fromOrder($order);
        $result = $this->gateway->process($payment);
        
        if ($result->success) {
            $order->markAsPaid($result->transactionId);
        }
    }
}

// Другие примеры Protected Variations:
// - Data Mapper (защита от изменений БД)
// - Config файлы (защита от изменений настроек)
// - Environment variables
// - Strategy паттерн
// - Dependency Injection
```

---

## GRASP в Laravel приложении

### Пример: Создание заказа

```php
<?php

// 1. Information Expert - Order знает свой total
class Order
{
    private Collection $items;
    
    public function getTotal(): float // Information Expert
    {
        return $this->items->sum(fn($i) => $i->getSubtotal());
    }
}

// 2. Creator - Order создает OrderItems
class Order
{
    public function addProduct(Product $product, int $qty): OrderItem // Creator
    {
        $item = new OrderItem($product, $qty);
        $this->items->push($item);
        return $item;
    }
}

// 3. Controller - Use Case контроллер
class CreateOrderService // Controller (GRASP)
{
    public function __construct(
        private OrderRepository $orders,
        private PaymentGateway $payment
    ) {}
    
    public function execute(CreateOrderDTO $data): Order
    {
        $order = new Order($data->user);
        
        foreach ($data->items as $item) {
            $order->addProduct($item->product, $item->quantity);
        }
        
        $this->payment->process($order);
        $this->orders->save($order);
        
        return $order;
    }
}

// 4. Low Coupling - зависит от интерфейсов
interface PaymentGateway { } // Low Coupling
interface OrderRepository { } // Low Coupling

// 5. High Cohesion - каждый класс делает одно
class Order { } // Только бизнес-логика заказа
class OrderRepository { } // Только персистентность
class PaymentGateway { } // Только оплата

// 6. Polymorphism - разные способы оплаты
interface PaymentGateway
{
    public function process(Order $order): PaymentResult;
}

class PayPalGateway implements PaymentGateway { } // Polymorphism
class StripeGateway implements PaymentGateway { } // Polymorphism

// 7. Pure Fabrication - технические классы
class OrderRepository { } // Pure Fabrication
class Logger { } // Pure Fabrication
class Cache { } // Pure Fabrication

// 8. Indirection - посредники
class CreateOrderService
{
    public function __construct(
        private OrderRepository $orders, // Indirection к БД
        private EventDispatcher $events  // Indirection к событиям
    ) {}
}

// 9. Protected Variations - защита от изменений
// PaymentGateway интерфейс защищает от изменения платежных систем
// OrderRepository интерфейс защищает от изменения БД
```

---

## Связь GRASP с SOLID

| GRASP | SOLID |
|-------|-------|
| Low Coupling | Dependency Inversion Principle |
| High Cohesion | Single Responsibility Principle |
| Polymorphism | Open/Closed Principle, Liskov Substitution |
| Protected Variations | Open/Closed Principle |
| Pure Fabrication | Single Responsibility Principle |

GRASP - это более общие принципы для присвоения ответственности.
SOLID - более конкретные принципы дизайна классов.

Используй оба набора принципов вместе для создания качественной архитектуры!

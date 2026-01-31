# Domain-Driven Design (DDD)

Полное руководство по DDD для PHP/Laravel приложений.

---

## Что такое DDD?

**Domain-Driven Design** — подход к разработке сложных систем, где бизнес-логика (домен) является центром архитектуры.

### Основные принципы

1. **Ubiquitous Language** - единый язык бизнеса и разработчиков
2. **Bounded Contexts** - разделение на изолированные контексты
3. **Domain Model** - модель предметной области
4. **Strategic Design** - архитектурные паттерны
5. **Tactical Design** - тактические паттерны (Entity, Value Object, etc.)

---

## Структура DDD проекта

```
app/
├── Domain/                    # Ядро бизнес-логики
│   ├── Order/                 # Bounded Context
│   │   ├── Models/            # Entities & Value Objects
│   │   │   ├── Order.php
│   │   │   ├── OrderItem.php
│   │   │   └── ValueObjects/
│   │   │       ├── Money.php
│   │   │       ├── OrderStatus.php
│   │   │       └── Address.php
│   │   ├── Repositories/      # Интерфейсы репозиториев
│   │   │   └── OrderRepositoryInterface.php
│   │   ├── Services/          # Domain Services
│   │   │   └── OrderCalculator.php
│   │   ├── Events/            # Domain Events
│   │   │   ├── OrderPlaced.php
│   │   │   └── OrderShipped.php
│   │   ├── Exceptions/        # Domain Exceptions
│   │   │   └── InvalidOrderException.php
│   │   └── Specifications/    # Business Rules
│   │       └── OrderCanBeShipped.php
│   │
│   └── User/                  # Другой Bounded Context
│       └── ...
│
├── Application/               # Use Cases / Application Services
│   ├── Order/
│   │   ├── PlaceOrderService.php
│   │   ├── ShipOrderService.php
│   │   └── DTOs/
│   │       └── PlaceOrderDTO.php
│   └── User/
│       └── ...
│
├── Infrastructure/            # Технические детали
│   ├── Persistence/           # Реализации репозиториев
│   │   ├── Eloquent/
│   │   │   └── EloquentOrderRepository.php
│   │   └── InMemory/
│   │       └── InMemoryOrderRepository.php
│   ├── External/              # Внешние сервисы
│   │   ├── PaymentGateway/
│   │   └── EmailService/
│   └── Queue/
│       └── ...
│
└── Presentation/              # UI Layer (Controllers, API)
    ├── Http/
    │   └── Controllers/
    │       ├── OrderController.php
    │       └── UserController.php
    └── Console/
        └── Commands/
```

---

## Building Blocks (Строительные блоки)

### 1. Entity (Сущность)

**Идентичность важнее атрибутов. Два объекта с одним ID — одна сущность.**

```php
<?php

namespace App\Domain\Order\Models;

use App\Domain\Order\ValueObjects\OrderStatus;
use App\Domain\Order\ValueObjects\Money;
use App\Domain\Order\Events\OrderPlaced;
use App\Domain\Order\Exceptions\InvalidOrderException;

class Order
{
    private string $id;
    private string $userId;
    private array $items = [];
    private OrderStatus $status;
    private Money $total;
    private \DateTimeImmutable $createdAt;
    private array $domainEvents = [];

    public function __construct(
        string $id,
        string $userId,
        \DateTimeImmutable $createdAt = null
    ) {
        $this->id = $id;
        $this->userId = $userId;
        $this->status = OrderStatus::pending();
        $this->total = new Money(0, 'USD');
        $this->createdAt = $createdAt ?? new \DateTimeImmutable();
    }

    public function id(): string
    {
        return $this->id;
    }

    public function addItem(OrderItem $item): void
    {
        if (!$this->status->isPending()) {
            throw new InvalidOrderException('Cannot add items to non-pending order');
        }

        $this->items[] = $item;
        $this->recalculateTotal();
    }

    public function place(): void
    {
        if (empty($this->items)) {
            throw new InvalidOrderException('Cannot place empty order');
        }

        if (!$this->status->isPending()) {
            throw new InvalidOrderException('Order already placed');
        }

        $this->status = OrderStatus::placed();
        
        // Domain Event
        $this->recordEvent(new OrderPlaced($this->id, $this->userId, $this->total));
    }

    public function ship(): void
    {
        if (!$this->status->canBeShipped()) {
            throw new InvalidOrderException('Order cannot be shipped in current status');
        }

        $this->status = OrderStatus::shipped();
    }

    private function recalculateTotal(): void
    {
        $total = 0;
        
        foreach ($this->items as $item) {
            $total += $item->subtotal()->amount();
        }

        $this->total = new Money($total, 'USD');
    }

    private function recordEvent(object $event): void
    {
        $this->domainEvents[] = $event;
    }

    public function releaseEvents(): array
    {
        $events = $this->domainEvents;
        $this->domainEvents = [];
        return $events;
    }

    // Getters
    public function status(): OrderStatus
    {
        return $this->status;
    }

    public function total(): Money
    {
        return $this->total;
    }

    public function items(): array
    {
        return $this->items;
    }
}
```

### 2. Value Object (Объект-значение)

**Не имеет идентичности. Два VO с одинаковыми атрибутами — равны.**

```php
<?php

namespace App\Domain\Order\ValueObjects;

class Money
{
    private float $amount;
    private string $currency;

    public function __construct(float $amount, string $currency)
    {
        if ($amount < 0) {
            throw new \InvalidArgumentException('Amount cannot be negative');
        }

        if (!in_array($currency, ['USD', 'EUR', 'RUB'])) {
            throw new \InvalidArgumentException("Invalid currency: {$currency}");
        }

        $this->amount = $amount;
        $this->currency = $currency;
    }

    public function amount(): float
    {
        return $this->amount;
    }

    public function currency(): string
    {
        return $this->currency;
    }

    public function add(Money $other): self
    {
        if ($this->currency !== $other->currency) {
            throw new \InvalidArgumentException('Cannot add different currencies');
        }

        return new self($this->amount + $other->amount, $this->currency);
    }

    public function multiply(float $multiplier): self
    {
        return new self($this->amount * $multiplier, $this->currency);
    }

    public function equals(Money $other): bool
    {
        return $this->amount === $other->amount 
            && $this->currency === $other->currency;
    }

    public function format(): string
    {
        return number_format($this->amount, 2) . ' ' . $this->currency;
    }
}
```

```php
<?php

namespace App\Domain\Order\ValueObjects;

class OrderStatus
{
    private const PENDING = 'pending';
    private const PLACED = 'placed';
    private const PAID = 'paid';
    private const SHIPPED = 'shipped';
    private const DELIVERED = 'delivered';
    private const CANCELLED = 'cancelled';

    private string $value;

    private function __construct(string $value)
    {
        $this->value = $value;
    }

    public static function pending(): self
    {
        return new self(self::PENDING);
    }

    public static function placed(): self
    {
        return new self(self::PLACED);
    }

    public static function shipped(): self
    {
        return new self(self::SHIPPED);
    }

    public function isPending(): bool
    {
        return $this->value === self::PENDING;
    }

    public function canBeShipped(): bool
    {
        return in_array($this->value, [self::PLACED, self::PAID]);
    }

    public function value(): string
    {
        return $this->value;
    }

    public function equals(OrderStatus $other): bool
    {
        return $this->value === $other->value;
    }
}
```

```php
<?php

namespace App\Domain\Shared\ValueObjects;

class Email
{
    private string $value;

    public function __construct(string $value)
    {
        if (!filter_var($value, FILTER_VALIDATE_EMAIL)) {
            throw new \InvalidArgumentException("Invalid email: {$value}");
        }

        $this->value = strtolower($value);
    }

    public function value(): string
    {
        return $this->value;
    }

    public function domain(): string
    {
        return substr($this->value, strpos($this->value, '@') + 1);
    }

    public function equals(Email $other): bool
    {
        return $this->value === $other->value;
    }

    public function __toString(): string
    {
        return $this->value;
    }
}
```

### 3. Aggregate (Агрегат)

**Кластер Entities и Value Objects с единым корнем (Aggregate Root).**

```php
<?php

namespace App\Domain\Order\Models;

// Order - это Aggregate Root
class Order
{
    private string $id;
    private array $items = []; // OrderItem - часть агрегата
    private OrderStatus $status;
    
    // Все изменения через Aggregate Root
    public function addItem(string $productId, int $quantity, Money $price): void
    {
        // Проверки инвариантов агрегата
        if (!$this->status->isPending()) {
            throw new InvalidOrderException('Cannot modify non-pending order');
        }

        $item = new OrderItem($productId, $quantity, $price);
        $this->items[] = $item;
        
        // Поддержка инвариантов
        $this->recalculateTotal();
    }

    // Прямой доступ к OrderItem запрещен!
    // Все через Order (Aggregate Root)
}

class OrderItem
{
    private string $productId;
    private int $quantity;
    private Money $price;

    public function __construct(string $productId, int $quantity, Money $price)
    {
        if ($quantity <= 0) {
            throw new \InvalidArgumentException('Quantity must be positive');
        }

        $this->productId = $productId;
        $this->quantity = $quantity;
        $this->price = $price;
    }

    public function subtotal(): Money
    {
        return $this->price->multiply($this->quantity);
    }

    public function productId(): string
    {
        return $this->productId;
    }

    public function quantity(): int
    {
        return $this->quantity;
    }
}
```

**Правила Aggregate:**
1. Внешний мир работает только с Aggregate Root
2. Entities внутри агрегата недоступны напрямую
3. Один транзакция = один агрегат
4. Ссылки между агрегатами — только по ID

### 4. Repository (Репозиторий)

**Абстракция доступа к данным. Работает с Aggregates.**

```php
<?php

namespace App\Domain\Order\Repositories;

use App\Domain\Order\Models\Order;

interface OrderRepositoryInterface
{
    public function findById(string $id): ?Order;
    
    public function findByUserId(string $userId): array;
    
    public function save(Order $order): void;
    
    public function delete(Order $order): void;
    
    public function nextIdentity(): string;
}
```

**Реализация (Infrastructure):**

```php
<?php

namespace App\Infrastructure\Persistence\Eloquent;

use App\Domain\Order\Models\Order;
use App\Domain\Order\Models\OrderItem;
use App\Domain\Order\ValueObjects\Money;
use App\Domain\Order\ValueObjects\OrderStatus;
use App\Domain\Order\Repositories\OrderRepositoryInterface;
use App\Infrastructure\Persistence\Eloquent\Models\OrderEloquent;
use Ramsey\Uuid\Uuid;

class EloquentOrderRepository implements OrderRepositoryInterface
{
    public function findById(string $id): ?Order
    {
        $eloquent = OrderEloquent::with('items')->find($id);
        
        if (!$eloquent) {
            return null;
        }

        return $this->toDomain($eloquent);
    }

    public function findByUserId(string $userId): array
    {
        return OrderEloquent::where('user_id', $userId)
            ->get()
            ->map(fn($eloquent) => $this->toDomain($eloquent))
            ->toArray();
    }

    public function save(Order $order): void
    {
        $eloquent = OrderEloquent::findOrNew($order->id());
        
        $eloquent->id = $order->id();
        $eloquent->user_id = $order->userId();
        $eloquent->status = $order->status()->value();
        $eloquent->total_amount = $order->total()->amount();
        $eloquent->total_currency = $order->total()->currency();
        
        $eloquent->save();

        // Сохранить items
        $eloquent->items()->delete();
        foreach ($order->items() as $item) {
            $eloquent->items()->create([
                'product_id' => $item->productId(),
                'quantity' => $item->quantity(),
                'price_amount' => $item->price()->amount(),
                'price_currency' => $item->price()->currency(),
            ]);
        }

        // Dispatch domain events
        foreach ($order->releaseEvents() as $event) {
            event($event);
        }
    }

    public function delete(Order $order): void
    {
        OrderEloquent::destroy($order->id());
    }

    public function nextIdentity(): string
    {
        return Uuid::uuid4()->toString();
    }

    private function toDomain(OrderEloquent $eloquent): Order
    {
        $order = new Order(
            $eloquent->id,
            $eloquent->user_id,
            $eloquent->created_at
        );

        foreach ($eloquent->items as $itemEloquent) {
            $item = new OrderItem(
                $itemEloquent->product_id,
                $itemEloquent->quantity,
                new Money($itemEloquent->price_amount, $itemEloquent->price_currency)
            );
            
            // Используем reflection для добавления items (обход инкапсуляции)
            $reflection = new \ReflectionClass($order);
            $property = $reflection->getProperty('items');
            $property->setAccessible(true);
            $items = $property->getValue($order);
            $items[] = $item;
            $property->setValue($order, $items);
        }

        return $order;
    }
}
```

### 5. Domain Service (Доменный сервис)

**Логика, которая не принадлежит конкретной Entity.**

```php
<?php

namespace App\Domain\Order\Services;

use App\Domain\Order\Models\Order;
use App\Domain\Order\ValueObjects\Money;

class OrderCalculator
{
    /**
     * Расчет налога (бизнес-логика, не привязанная к Order)
     */
    public function calculateTax(Order $order, string $country): Money
    {
        $taxRates = [
            'US' => 0.10,  // 10%
            'EU' => 0.20,  // 20%
            'RU' => 0.20,  // 20%
        ];

        $rate = $taxRates[$country] ?? 0;
        
        return $order->total()->multiply($rate);
    }

    /**
     * Расчет доставки
     */
    public function calculateShipping(Order $order, string $shippingMethod): Money
    {
        $baseRate = match($shippingMethod) {
            'standard' => 5.00,
            'express' => 15.00,
            'overnight' => 30.00,
            default => 0.00,
        };

        // Бесплатная доставка при заказе > $100
        if ($order->total()->amount() > 100) {
            return new Money(0, 'USD');
        }

        return new Money($baseRate, 'USD');
    }

    /**
     * Можно ли объединить заказы
     */
    public function canMergeOrders(Order $order1, Order $order2): bool
    {
        // Бизнес-правило: можно объединить только заказы одного пользователя
        return $order1->userId() === $order2->userId()
            && $order1->status()->isPending()
            && $order2->status()->isPending();
    }
}
```

### 6. Domain Events (Доменные события)

**Что-то важное произошло в домене.**

```php
<?php

namespace App\Domain\Order\Events;

use App\Domain\Order\ValueObjects\Money;

class OrderPlaced
{
    public function __construct(
        public readonly string $orderId,
        public readonly string $userId,
        public readonly Money $total,
        public readonly \DateTimeImmutable $occurredAt = new \DateTimeImmutable()
    ) {}
}
```

```php
<?php

namespace App\Domain\Order\Events;

class OrderShipped
{
    public function __construct(
        public readonly string $orderId,
        public readonly string $trackingNumber,
        public readonly \DateTimeImmutable $occurredAt = new \DateTimeImmutable()
    ) {}
}
```

**Обработка событий (Infrastructure):**

```php
<?php

namespace App\Application\Order\Listeners;

use App\Domain\Order\Events\OrderPlaced;
use App\Infrastructure\External\EmailService;

class SendOrderConfirmationEmail
{
    public function __construct(
        private EmailService $emailService
    ) {}

    public function handle(OrderPlaced $event): void
    {
        $this->emailService->send(
            to: $event->userId,
            subject: 'Order Confirmation',
            body: "Your order #{$event->orderId} has been placed. Total: {$event->total->format()}"
        );
    }
}
```

### 7. Specification (Спецификация)

**Инкапсуляция бизнес-правил.**

```php
<?php

namespace App\Domain\Order\Specifications;

use App\Domain\Order\Models\Order;

interface SpecificationInterface
{
    public function isSatisfiedBy(Order $order): bool;
}
```

```php
<?php

namespace App\Domain\Order\Specifications;

use App\Domain\Order\Models\Order;

class OrderCanBeShipped implements SpecificationInterface
{
    public function isSatisfiedBy(Order $order): bool
    {
        // Бизнес-правила для возможности отгрузки
        return $order->status()->canBeShipped()
            && $order->total()->amount() > 0
            && !empty($order->items());
    }
}
```

```php
<?php

namespace App\Domain\Order\Specifications;

use App\Domain\Order\Models\Order;

class HighValueOrder implements SpecificationInterface
{
    private const HIGH_VALUE_THRESHOLD = 1000;

    public function isSatisfiedBy(Order $order): bool
    {
        return $order->total()->amount() >= self::HIGH_VALUE_THRESHOLD;
    }
}
```

**Использование:**

```php
$canBeShipped = new OrderCanBeShipped();
$isHighValue = new HighValueOrder();

if ($canBeShipped->isSatisfiedBy($order)) {
    $order->ship();
    
    if ($isHighValue->isSatisfiedBy($order)) {
        // Особая обработка для дорогих заказов
        $this->assignPremiumCourier($order);
    }
}
```

---

## Application Layer (Use Cases)

```php
<?php

namespace App\Application\Order;

use App\Domain\Order\Models\Order;
use App\Domain\Order\Models\OrderItem;
use App\Domain\Order\Repositories\OrderRepositoryInterface;
use App\Domain\Order\ValueObjects\Money;
use App\Application\Order\DTOs\PlaceOrderDTO;

class PlaceOrderService
{
    public function __construct(
        private OrderRepositoryInterface $orderRepository
    ) {}

    public function execute(PlaceOrderDTO $dto): string
    {
        // Создать Order (Aggregate Root)
        $order = new Order(
            id: $this->orderRepository->nextIdentity(),
            userId: $dto->userId
        );

        // Добавить items
        foreach ($dto->items as $itemData) {
            $order->addItem(
                new OrderItem(
                    productId: $itemData['product_id'],
                    quantity: $itemData['quantity'],
                    price: new Money($itemData['price'], 'USD')
                )
            );
        }

        // Разместить заказ (бизнес-логика в домене)
        $order->place();

        // Сохранить
        $this->orderRepository->save($order);

        return $order->id();
    }
}
```

**DTO (Data Transfer Object):**

```php
<?php

namespace App\Application\Order\DTOs;

class PlaceOrderDTO
{
    public function __construct(
        public readonly string $userId,
        public readonly array $items, // [['product_id' => '...', 'quantity' => 1, 'price' => 10.00]]
    ) {}

    public static function fromArray(array $data): self
    {
        return new self(
            userId: $data['user_id'],
            items: $data['items']
        );
    }
}
```

---

## Bounded Context (Ограниченный контекст)

**Разделение системы на независимые контексты с собственным языком.**

```
E-commerce система:

1. Order Context (Контекст заказов)
   - Order, OrderItem, Payment
   - Язык: "place order", "ship order", "cancel order"

2. Catalog Context (Контекст каталога)
   - Product, Category, Price
   - Язык: "add product", "update price", "categorize"

3. User Context (Контекст пользователей)
   - User, Profile, Authentication
   - Язык: "register user", "login", "update profile"

4. Shipping Context (Контекст доставки)
   - Shipment, Tracking, Courier
   - Язык: "create shipment", "track package", "assign courier"
```

**Интеграция между контекстами:**

```php
// Order Context нужен Product, но не зависит от Catalog Context
// Используем Anti-Corruption Layer (ACL)

namespace App\Domain\Order\Services;

class ProductService
{
    public function __construct(
        private CatalogApiClient $catalogApi
    ) {}

    public function getProductPrice(string $productId): Money
    {
        // Получить из Catalog Context
        $product = $this->catalogApi->getProduct($productId);
        
        // Преобразовать в доменную модель Order Context
        return new Money($product['price'], $product['currency']);
    }
}
```

---

## Ubiquitous Language (Единый язык)

**Бизнес и разработчики используют одни термины.**

```
❌ ПЛОХО:
- Разработчик: "User создал CartItem и вызвал checkout()"
- Бизнес: "Клиент добавил товар в корзину и оформил заказ"

✅ ХОРОШО:
- И разработчик, и бизнес: "Customer placed an Order with OrderItems"
```

**В коде:**

```php
// ❌ ПЛОХО - технический язык
class UserCartItemCheckout { }

// ✅ ХОРОШО - бизнес-язык
class PlaceOrder { }
class Customer { }
class OrderItem { }
```

---

## DDD в Laravel

### Service Provider для DI

```php
<?php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use App\Domain\Order\Repositories\OrderRepositoryInterface;
use App\Infrastructure\Persistence\Eloquent\EloquentOrderRepository;

class DomainServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // Bind interfaces to implementations
        $this->app->bind(
            OrderRepositoryInterface::class,
            EloquentOrderRepository::class
        );
    }
}
```

### Controller (Presentation Layer)

```php
<?php

namespace App\Presentation\Http\Controllers;

use App\Application\Order\PlaceOrderService;
use App\Application\Order\DTOs\PlaceOrderDTO;
use Illuminate\Http\Request;

class OrderController extends Controller
{
    public function __construct(
        private PlaceOrderService $placeOrderService
    ) {}

    public function store(Request $request)
    {
        $validated = $request->validate([
            'user_id' => 'required|uuid',
            'items' => 'required|array',
            'items.*.product_id' => 'required|uuid',
            'items.*.quantity' => 'required|integer|min:1',
            'items.*.price' => 'required|numeric|min:0',
        ]);

        $dto = PlaceOrderDTO::fromArray($validated);
        
        $orderId = $this->placeOrderService->execute($dto);

        return response()->json([
            'order_id' => $orderId,
        ], 201);
    }
}
```

---

## Преимущества DDD

```
✅ Бизнес-логика изолирована от технических деталей
✅ Код отражает бизнес-процессы
✅ Легко тестировать (чистый PHP без зависимостей от фреймворка)
✅ Масштабируемость (bounded contexts)
✅ Единый язык с бизнесом

❌ Сложность для простых CRUD систем
❌ Больше кода (много слоев абстракции)
❌ Крутая кривая обучения
```

---

## Когда использовать DDD

```
✅ Используй DDD:
- Сложная бизнес-логика
- Долгоживущий проект
- Большая команда
- Нужна модульность

❌ НЕ используй DDD:
- Простой CRUD
- Прототип/MVP
- Маленькая команда (1-2 разработчика)
- Короткий срок жизни проекта
```

---

## Best Practices

### 1. Держи домен чистым

```php
// ✅ ХОРОШО - чистый PHP, нет зависимостей от Laravel
namespace App\Domain\Order\Models;

class Order
{
    // Никаких Eloquent, Facades, etc.
}

// ❌ ПЛОХО
class Order extends Model // Зависимость от Eloquent
{
    use SoftDeletes; // Технические детали в домене
}
```

### 2. Используй Value Objects

```php
// ✅ ХОРОШО
private Money $total;

// ❌ ПЛОХО
private float $total;
private string $currency;
```

### 3. Инварианты в Aggregate Root

```php
class Order
{
    public function addItem(OrderItem $item): void
    {
        // Проверка инвариантов
        if (!$this->canAddItems()) {
            throw new InvalidOrderException();
        }
        
        $this->items[] = $item;
        $this->recalculateTotal(); // Поддержка инвариантов
    }
}
```

### 4. Repository возвращает Domain Models

```php
// ✅ ХОРОШО
public function findById(string $id): ?Order; // Domain\Order\Models\Order

// ❌ ПЛОХО
public function findById(string $id): ?OrderEloquent; // Infrastructure модель
```

### 5. Domain Events для интеграции

```php
// Вместо прямых вызовов других контекстов
class Order
{
    public function place(): void
    {
        $this->status = OrderStatus::placed();
        
        // Не вызывай напрямую:
        // $this->inventoryService->reserve($this->items);
        
        // Используй событие:
        $this->recordEvent(new OrderPlaced($this->id));
    }
}

// Listener в Infrastructure
class ReserveInventoryOnOrderPlaced
{
    public function handle(OrderPlaced $event): void
    {
        $this->inventoryService->reserve($event->orderId);
    }
}
```

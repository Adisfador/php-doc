# Events - События в Laravel

Полный разбор событий: events/listeners, observers, broadcasting, event subscribers, testing, async processing.

---

## 🎯 Что такое Events?

**Event-driven architecture** - слабосвязанная архитектура через события.

**Преимущества:**
- 🔓 Decoupling - отделение логики
- 📢 Broadcasting - real-time уведомления
- ✅ Testability - легко тестировать
- 🔄 Extensibility - легко добавлять новые listeners

**Концепция:**
```
User Registered → Event Dispatched → Listeners Execute
                                    ├─ Send Welcome Email
                                    ├─ Create Profile
                                    └─ Log Registration
```

---

## 📢 Defining Events

### Создание Event

```bash
php artisan make:event UserRegistered
```

**app/Events/UserRegistered.php:**
```php
<?php

namespace App\Events;

use App\Models\User;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class UserRegistered
{
    use Dispatchable, InteractsWithSockets, SerializesModels;
    
    public function __construct(
        public User $user
    ) {}
}
```

**Traits:**
- `Dispatchable` - можно вызывать `Event::dispatch()`
- `InteractsWithSockets` - для broadcasting
- `SerializesModels` - сериализация моделей для queue

### Event с дополнительными данными

```php
class OrderShipped
{
    use Dispatchable, SerializesModels;
    
    public function __construct(
        public Order $order,
        public string $trackingNumber,
        public ?User $shippedBy = null
    ) {}
}
```

---

## 👂 Listeners

### Создание Listener

```bash
php artisan make:listener SendWelcomeEmail --event=UserRegistered
```

**app/Listeners/SendWelcomeEmail.php:**
```php
<?php

namespace App\Listeners;

use App\Events\UserRegistered;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Support\Facades\Mail;

class SendWelcomeEmail implements ShouldQueue
{
    use InteractsWithQueue;
    
    public function __construct()
    {
        // Зависимости через DI
    }
    
    public function handle(UserRegistered $event): void
    {
        Mail::to($event->user->email)
            ->send(new WelcomeEmail($event->user));
    }
    
    public function failed(UserRegistered $event, \Throwable $exception): void
    {
        // Обработка ошибки
        Log::error('Failed to send welcome email', [
            'user_id' => $event->user->id,
            'error' => $exception->getMessage(),
        ]);
    }
}
```

### ShouldQueue - Async Listeners

```php
class SendWelcomeEmail implements ShouldQueue
{
    use InteractsWithQueue;
    
    // Кастомная очередь
    public string $queue = 'emails';
    
    // Connection
    public string $connection = 'redis';
    
    // Задержка
    public int $delay = 60;  // 1 минута
    
    // Количество попыток
    public int $tries = 3;
    
    // Таймаут
    public int $timeout = 120;
    
    public function handle(UserRegistered $event): void
    {
        // Проверить количество попыток
        if ($this->attempts() > 2) {
            Log::warning('Multiple attempts to send email');
        }
        
        Mail::to($event->user->email)->send(new WelcomeEmail($event->user));
    }
}
```

### Stopping Event Propagation

```php
class SendWelcomeEmail
{
    public function handle(UserRegistered $event): bool
    {
        if ($event->user->email_verified) {
            return false;  // Остановить дальнейшие listeners
        }
        
        Mail::to($event->user->email)->send(new WelcomeEmail($event->user));
        
        return true;
    }
}
```

---

## 🔗 Регистрация Events/Listeners

### EventServiceProvider

**app/Providers/EventServiceProvider.php:**
```php
<?php

namespace App\Providers;

use App\Events\OrderShipped;
use App\Events\UserRegistered;
use App\Listeners\LogOrderShipment;
use App\Listeners\SendShipmentNotification;
use App\Listeners\SendWelcomeEmail;
use Illuminate\Auth\Events\Registered;
use Illuminate\Auth\Listeners\SendEmailVerificationNotification;
use Illuminate\Foundation\Support\Providers\EventServiceProvider as ServiceProvider;

class EventServiceProvider extends ServiceProvider
{
    protected $listen = [
        UserRegistered::class => [
            SendWelcomeEmail::class,
            CreateUserProfile::class,
            LogUserRegistration::class,
        ],
        
        OrderShipped::class => [
            SendShipmentNotification::class,
            LogOrderShipment::class,
        ],
        
        // Built-in Laravel events
        Registered::class => [
            SendEmailVerificationNotification::class,
        ],
    ];
    
    public function boot(): void
    {
        //
    }
    
    public function shouldDiscoverEvents(): bool
    {
        return true;  // Auto-discovery
    }
}
```

### Event Discovery (автоматическая регистрация)

```php
// Laravel автоматически найдет все Listeners в app/Listeners
// если они соответствуют naming convention

// app/Events/UserRegistered.php
// app/Listeners/SendWelcomeEmail.php ✅ (автоматически)
// app/Listeners/User/SendWelcomeEmail.php ✅ (тоже найдет)

protected function discoverEventsWithin(): array
{
    return [
        $this->app->path('Listeners'),
        $this->app->path('Domain/*/Listeners'),  // Custom structure
    ];
}
```

### Closure Listeners (простые случаи)

```php
use Illuminate\Support\Facades\Event;

Event::listen(UserRegistered::class, function (UserRegistered $event) {
    Log::info('User registered: ' . $event->user->email);
});

// В boot() EventServiceProvider
Event::listen('user.*', function (string $eventName, array $data) {
    // Wildcard listener для user.created, user.updated, etc.
});
```

---

## 🚀 Dispatching Events

### Event::dispatch()

```php
use App\Events\UserRegistered;
use Illuminate\Support\Facades\Event;

// Вариант 1
Event::dispatch(new UserRegistered($user));

// Вариант 2 (через trait Dispatchable)
UserRegistered::dispatch($user);

// Вариант 3 (helper)
event(new UserRegistered($user));
```

### Conditional Dispatching

```php
// Dispatch если условие выполнено
UserRegistered::dispatchIf($user->isActive(), $user);

// Dispatch если условие НЕ выполнено
UserRegistered::dispatchUnless($user->isGuest(), $user);
```

### Dispatching After DB Transaction

```php
use Illuminate\Events\Dispatcher;

class OrderShipped implements ShouldDispatchAfterCommit
{
    use Dispatchable;
    
    public function __construct(public Order $order) {}
}

// Или явно
DB::transaction(function() use ($order) {
    $order->update(['status' => 'shipped']);
    
    OrderShipped::dispatch($order);  // Выполнится после commit
});

// Если transaction откатится, событие не отправится
```

---

## 👀 Model Observers

**Альтернатива events для Eloquent моделей.**

### Создание Observer

```bash
php artisan make:observer UserObserver --model=User
```

**app/Observers/UserObserver.php:**
```php
<?php

namespace App\Observers;

use App\Models\User;
use Illuminate\Support\Facades\Cache;

class UserObserver
{
    public function creating(User $user): void
    {
        // Перед созданием
        $user->uuid = Str::uuid();
    }
    
    public function created(User $user): void
    {
        // После создания
        Cache::forget('users:count');
        
        // Отправить welcome email
        Mail::to($user->email)->send(new WelcomeEmail($user));
    }
    
    public function updating(User $user): void
    {
        // Перед обновлением
        if ($user->isDirty('email')) {
            $user->email_verified_at = null;
        }
    }
    
    public function updated(User $user): void
    {
        // После обновления
        Cache::forget("user:{$user->id}");
    }
    
    public function deleting(User $user): void
    {
        // Перед удалением
        $user->posts()->delete();
    }
    
    public function deleted(User $user): void
    {
        // После удаления
        Cache::forget("user:{$user->id}");
    }
    
    public function restored(User $user): void
    {
        // После восстановления (soft deletes)
    }
    
    public function forceDeleted(User $user): void
    {
        // После force delete
    }
}
```

### Регистрация Observer

```php
// app/Providers/EventServiceProvider.php
use App\Models\User;
use App\Observers\UserObserver;

public function boot(): void
{
    User::observe(UserObserver::class);
}

// Или в AppServiceProvider
public function boot(): void
{
    User::observe(UserObserver::class);
    Post::observe(PostObserver::class);
}
```

### Все методы Observer

```php
class UserObserver
{
    public function retrieved(User $user) {}      // После получения из БД
    public function creating(User $user) {}       // Перед INSERT
    public function created(User $user) {}        // После INSERT
    public function updating(User $user) {}       // Перед UPDATE
    public function updated(User $user) {}        // После UPDATE
    public function saving(User $user) {}         // Перед INSERT/UPDATE
    public function saved(User $user) {}          // После INSERT/UPDATE
    public function deleting(User $user) {}       // Перед DELETE
    public function deleted(User $user) {}        // После DELETE
    public function restoring(User $user) {}      // Перед restore()
    public function restored(User $user) {}       // После restore()
    public function forceDeleting(User $user) {}  // Перед forceDelete()
    public function forceDeleted(User $user) {}   // После forceDelete()
    public function replicating(User $user) {}    // При replicate()
}
```

---

## 📡 Broadcasting Events (WebSockets)

### Настройка Broadcasting

```bash
composer require pusher/pusher-php-server

# Или Laravel Reverb (официальный WebSocket server)
composer require laravel/reverb
php artisan reverb:install
```

**.env:**
```env
BROADCAST_DRIVER=reverb  # или pusher, ably

# Reverb
REVERB_APP_ID=my-app-id
REVERB_APP_KEY=my-app-key
REVERB_APP_SECRET=my-app-secret
REVERB_HOST=localhost
REVERB_PORT=8080
REVERB_SCHEME=http

# Pusher (альтернатива)
PUSHER_APP_ID=your-app-id
PUSHER_APP_KEY=your-app-key
PUSHER_APP_SECRET=your-app-secret
PUSHER_APP_CLUSTER=mt1
```

### ShouldBroadcast Event

```php
<?php

namespace App\Events;

use App\Models\Order;
use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Broadcasting\PresenceChannel;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class OrderShipped implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;
    
    public function __construct(
        public Order $order
    ) {}
    
    // Канал для broadcasting
    public function broadcastOn(): Channel|array
    {
        return new PrivateChannel('orders.' . $this->order->user_id);
    }
    
    // Имя события (по умолчанию = class name)
    public function broadcastAs(): string
    {
        return 'order.shipped';
    }
    
    // Данные для отправки
    public function broadcastWith(): array
    {
        return [
            'order_id' => $this->order->id,
            'tracking_number' => $this->order->tracking_number,
            'shipped_at' => $this->order->shipped_at,
        ];
    }
    
    // Условие для broadcasting
    public function broadcastWhen(): bool
    {
        return $this->order->user->notificationPreferences->push_enabled;
    }
}
```

### Типы каналов

**1. Public Channel:**
```php
public function broadcastOn(): Channel
{
    return new Channel('notifications');  // Доступен всем
}
```

**2. Private Channel:**
```php
public function broadcastOn(): Channel
{
    return new PrivateChannel('user.' . $this->user->id);
}

// routes/channels.php
Broadcast::channel('user.{id}', function ($user, $id) {
    return (int) $user->id === (int) $id;  // Authorization
});
```

**3. Presence Channel:**
```php
public function broadcastOn(): Channel
{
    return new PresenceChannel('chat.' . $this->chatRoom->id);
}

// routes/channels.php
Broadcast::channel('chat.{roomId}', function ($user, $roomId) {
    if ($user->canAccessChatRoom($roomId)) {
        return [
            'id' => $user->id,
            'name' => $user->name,
            'avatar' => $user->avatar_url,
        ];  // Данные о пользователе для других участников
    }
});
```

### Laravel Echo (JavaScript client)

```bash
npm install --save-dev laravel-echo pusher-js
```

**resources/js/bootstrap.js:**
```javascript
import Echo from 'laravel-echo';
import Pusher from 'pusher-js';

window.Pusher = Pusher;

window.Echo = new Echo({
    broadcaster: 'reverb',
    key: import.meta.env.VITE_REVERB_APP_KEY,
    wsHost: import.meta.env.VITE_REVERB_HOST,
    wsPort: import.meta.env.VITE_REVERB_PORT,
    wssPort: import.meta.env.VITE_REVERB_PORT,
    forceTLS: (import.meta.env.VITE_REVERB_SCHEME ?? 'https') === 'https',
    enabledTransports: ['ws', 'wss'],
});
```

**Подписка на события:**
```javascript
// Public channel
Echo.channel('notifications')
    .listen('OrderShipped', (e) => {
        console.log('Order shipped:', e.order_id);
    });

// Private channel
Echo.private('user.' + userId)
    .listen('.order.shipped', (e) => {
        alert('Your order has been shipped!');
        updateOrderStatus(e.order_id, 'shipped');
    });

// Presence channel
Echo.join('chat.' + roomId)
    .here((users) => {
        console.log('Users in room:', users);
    })
    .joining((user) => {
        console.log(user.name + ' joined');
    })
    .leaving((user) => {
        console.log(user.name + ' left');
    })
    .listen('MessageSent', (e) => {
        displayMessage(e.message);
    });

// Отписаться
Echo.leave('chat.' + roomId);
```

### Notifications Broadcasting

```php
// app/Notifications/OrderShipped.php
use Illuminate\Notifications\Messages\BroadcastMessage;

public function via($notifiable): array
{
    return ['database', 'broadcast'];
}

public function toBroadcast($notifiable): BroadcastMessage
{
    return new BroadcastMessage([
        'order_id' => $this->order->id,
        'message' => 'Your order has been shipped!',
    ]);
}

// JavaScript
Echo.private('App.Models.User.' + userId)
    .notification((notification) => {
        console.log(notification);
        showNotification(notification.message);
    });
```

---

## 📋 Event Subscribers

**Один класс для множества событий.**

### Создание Subscriber

```bash
php artisan make:listener UserEventSubscriber
```

**app/Listeners/UserEventSubscriber.php:**
```php
<?php

namespace App\Listeners;

use App\Events\UserRegistered;
use App\Events\UserDeleted;
use App\Events\UserUpdated;
use Illuminate\Events\Dispatcher;
use Illuminate\Support\Facades\Log;

class UserEventSubscriber
{
    public function handleUserRegistered(UserRegistered $event): void
    {
        Log::info('User registered', ['user_id' => $event->user->id]);
    }
    
    public function handleUserUpdated(UserUpdated $event): void
    {
        Log::info('User updated', ['user_id' => $event->user->id]);
    }
    
    public function handleUserDeleted(UserDeleted $event): void
    {
        Log::info('User deleted', ['user_id' => $event->user->id]);
    }
    
    public function subscribe(Dispatcher $events): array
    {
        return [
            UserRegistered::class => 'handleUserRegistered',
            UserUpdated::class => 'handleUserUpdated',
            UserDeleted::class => 'handleUserDeleted',
        ];
    }
}
```

### Регистрация Subscriber

```php
// app/Providers/EventServiceProvider.php
protected $subscribe = [
    UserEventSubscriber::class,
    OrderEventSubscriber::class,
];
```

---

## 🧪 Testing Events

### Event::fake()

```php
use App\Events\UserRegistered;
use Illuminate\Support\Facades\Event;

test('user registration dispatches event', function () {
    Event::fake();
    
    // Выполнить действие
    $response = $this->post('/register', [
        'name' => 'John Doe',
        'email' => 'john@example.com',
        'password' => 'password',
    ]);
    
    // Проверить что событие было отправлено
    Event::assertDispatched(UserRegistered::class);
    
    // С проверкой данных
    Event::assertDispatched(UserRegistered::class, function ($event) {
        return $event->user->email === 'john@example.com';
    });
    
    // Проверить количество вызовов
    Event::assertDispatched(UserRegistered::class, 1);
});
```

### Assertions

```php
// Событие отправлено
Event::assertDispatched(UserRegistered::class);

// Событие НЕ отправлено
Event::assertNotDispatched(UserDeleted::class);

// Событие отправлено N раз
Event::assertDispatchedTimes(UserRegistered::class, 3);

// Событие отправлено с условием
Event::assertDispatched(UserRegistered::class, function ($event) {
    return $event->user->isAdmin();
});

// Список всех отправленных событий
Event::assertListening(
    UserRegistered::class,
    SendWelcomeEmail::class
);
```

### Fake конкретные события

```php
// Fake только UserRegistered, остальные работают нормально
Event::fake([UserRegistered::class]);

// Fake все кроме OrderShipped
Event::fakeExcept([OrderShipped::class]);
```

### Testing Listeners

```php
test('welcome email is sent', function () {
    Mail::fake();
    
    $user = User::factory()->create();
    $event = new UserRegistered($user);
    
    $listener = new SendWelcomeEmail();
    $listener->handle($event);
    
    Mail::assertSent(WelcomeEmail::class, function ($mail) use ($user) {
        return $mail->hasTo($user->email);
    });
});
```

### Testing Observers

```php
test('user profile is created on registration', function () {
    $user = User::factory()->create();
    
    expect($user->profile)->not->toBeNull();
    expect($user->profile->user_id)->toBe($user->id);
});
```

### Testing Broadcasting

```php
use Illuminate\Support\Facades\Event;

test('order shipped event is broadcasted', function () {
    Event::fake();
    
    $order = Order::factory()->create();
    
    OrderShipped::dispatch($order);
    
    Event::assertDispatched(OrderShipped::class);
    
    // Проверить channel
    $event = new OrderShipped($order);
    expect($event->broadcastOn())->toBeInstanceOf(PrivateChannel::class);
    expect($event->broadcastOn()->name)->toBe('orders.' . $order->user_id);
});
```

---

## 🎯 Built-in Laravel Events

### Authentication Events

```php
// Illuminate\Auth\Events
Registered::class           // Пользователь зарегистрирован
Attempting::class           // Попытка входа
Authenticated::class        // Успешный вход
Login::class                // Login
Logout::class               // Logout
Failed::class               // Неудачная попытка входа
Validated::class            // Credentials validated
Verified::class             // Email verified
CurrentDeviceLogout::class  // Logout текущего устройства
OtherDeviceLogout::class    // Logout других устройств
Lockout::class              // Account locked (throttle)
PasswordReset::class        // Password reset
PasswordResetLinkSent::class // Reset link sent
```

**Пример:**
```php
use Illuminate\Auth\Events\Login;

Event::listen(Login::class, function (Login $event) {
    Log::info('User logged in', [
        'user_id' => $event->user->id,
        'ip' => request()->ip(),
        'remember' => $event->remember,
    ]);
});
```

### Database Events

```php
// Illuminate\Database\Events
QueryExecuted::class         // SQL query выполнен
TransactionBeginning::class  // Transaction started
TransactionCommitted::class  // Transaction committed
TransactionRolledBack::class // Transaction rolled back
MigrationsStarted::class     // Migrations started
MigrationsEnded::class       // Migrations ended
SchemaDumped::class          // Schema dumped
```

**Пример - логирование медленных запросов:**
```php
use Illuminate\Database\Events\QueryExecuted;

Event::listen(QueryExecuted::class, function (QueryExecuted $event) {
    if ($event->time > 1000) {  // Больше 1 секунды
        Log::warning('Slow query detected', [
            'sql' => $event->sql,
            'bindings' => $event->bindings,
            'time' => $event->time,
        ]);
    }
});
```

### Cache Events

```php
// Illuminate\Cache\Events
CacheHit::class         // Кэш найден
CacheMissed::class      // Кэш не найден
KeyForgotten::class     // Ключ удален
KeyWritten::class       // Ключ записан
```

### Queue Events

```php
// Illuminate\Queue\Events
JobProcessing::class     // Job начал обработку
JobProcessed::class      // Job завершен
JobFailed::class         // Job failed
JobQueued::class         // Job добавлен в очередь
WorkerStopping::class    // Worker останавливается
```

---

## 🔄 Events vs Observers

### Когда использовать Events?

✅ **Используй Events когда:**
- Нужно обрабатывать событие в нескольких местах
- Нужен broadcasting (WebSockets)
- Событие не связано с конкретной моделью
- Нужна гибкость (легко добавлять/удалять listeners)

```php
// Event для бизнес-логики
class PaymentProcessed
{
    public function __construct(
        public Order $order,
        public float $amount,
        public string $paymentMethod
    ) {}
}

// Listeners
SendPaymentConfirmation::class
UpdateInventory::class
CreateInvoice::class
NotifyAdmin::class
```

### Когда использовать Observers?

✅ **Используй Observers когда:**
- Логика тесно связана с моделью
- Нужно реагировать на CRUD операции
- Простые side-effects (cache clearing, logging)

```php
// Observer для модели User
class UserObserver
{
    public function created(User $user): void
    {
        // Создать profile
        $user->profile()->create();
    }
    
    public function updated(User $user): void
    {
        // Очистить кэш
        Cache::forget("user:{$user->id}");
    }
}
```

### Comparison

| Feature | Events | Observers |
|---------|--------|-----------|
| Scope | Любые события | Только Eloquent models |
| Broadcasting | ✅ | ❌ |
| Queuing | ✅ | ❌ (нужно вручную) |
| Flexibility | Высокая | Средняя |
| Use case | Бизнес-события | Model lifecycle |

---

## 🎓 Best Practices

### 1. Имена событий - past tense

```php
// ✅ Правильно (прошедшее время)
UserRegistered::class
OrderShipped::class
PaymentProcessed::class

// ❌ Неправильно
UserRegister::class
ShipOrder::class
ProcessPayment::class
```

### 2. События должны быть immutable

```php
// ✅ Readonly свойства
class OrderShipped
{
    public function __construct(
        public readonly Order $order,
        public readonly string $trackingNumber
    ) {}
}

// ❌ Mutable
class OrderShipped
{
    public Order $order;
    
    public function setOrder(Order $order) {
        $this->order = $order;  // Плохо
    }
}
```

### 3. ShouldQueue для тяжелых операций

```php
// Email отправка - async
class SendWelcomeEmail implements ShouldQueue
{
    public function handle(UserRegistered $event): void
    {
        Mail::to($event->user->email)->send(new WelcomeEmail($event->user));
    }
}

// Логирование - sync (быстро)
class LogUserRegistration
{
    public function handle(UserRegistered $event): void
    {
        Log::info('User registered', ['user_id' => $event->user->id]);
    }
}
```

### 4. Не злоупотребляй событиями

```php
// ❌ Слишком много событий
UserEmailChanged::class
UserPasswordChanged::class
UserNameChanged::class

// ✅ Одно событие для обновления
UserUpdated::class

// Listener решает что делать
class UserUpdatedListener
{
    public function handle(UserUpdated $event): void
    {
        if ($event->user->wasChanged('email')) {
            // Send verification email
        }
    }
}
```

### 5. Используй Event Discovery

```php
// EventServiceProvider
public function shouldDiscoverEvents(): bool
{
    return true;  // Автоматическая регистрация
}

// Laravel найдет:
// app/Events/UserRegistered.php
// app/Listeners/SendWelcomeEmail.php
```

---

## 🎓 Для собеседования: ключевые точки

1. **Events vs Observers** - Events для бизнес-логики, Observers для model lifecycle
2. **ShouldQueue** - async listeners через очереди
3. **Broadcasting** - ShouldBroadcast для WebSockets, Echo на frontend
4. **Channels** - Public (всем), Private (с authorization), Presence (с user info)
5. **Event Discovery** - автоматическая регистрация listeners
6. **Testing** - Event::fake(), assertDispatched()
7. **Built-in events** - Login, QueryExecuted, CacheHit и т.д.
8. **Best practices** - immutable events, past tense names, ShouldQueue для heavy tasks
9. **EventServiceProvider** - $listen array для регистрации
10. **Subscribers** - один класс для множества событий

**Главное:** Events для decoupling, Observers для simple model side-effects, Broadcasting для real-time.

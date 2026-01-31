# Architectural Patterns - Архитектурные паттерны

Полный разбор архитектурных паттернов: Layered, MVC, MVP, MVVM, Microservices, Event-Driven, CQRS, Event Sourcing.

---

## 🎯 Что такое Architectural Patterns?

**Архитектурные паттерны** - высокоуровневые стратегии организации кода всего приложения.

**Vs Design Patterns:**
- **Design Patterns** - решения для конкретных проблем (Strategy, Factory)
- **Architectural Patterns** - общая структура приложения (MVC, Microservices)

---

## 🏛️ Layered Architecture (N-Tier)

### Концепция

Разделение приложения на **слои (layers)** с четкими обязанностями.

```
┌─────────────────────────┐
│   Presentation Layer    │  ← UI, Controllers
├─────────────────────────┤
│   Business Logic Layer  │  ← Services, Domain
├─────────────────────────┤
│   Data Access Layer     │  ← Repositories, ORM
├─────────────────────────┤
│   Database              │  ← PostgreSQL, Redis
└─────────────────────────┘
```

**Правила:**
- Каждый слой зависит только от слоя ниже
- Верхние слои НЕ знают о деталях нижних
- Данные проходят через все слои

### Реализация в Laravel

**1. Presentation Layer (Controllers):**
```php
// app/Http/Controllers/UserController.php
class UserController extends Controller
{
    public function __construct(
        private UserService $userService
    ) {}
    
    public function store(CreateUserRequest $request): JsonResponse
    {
        $user = $this->userService->createUser($request->validated());
        
        return response()->json(new UserResource($user), 201);
    }
}
```

**2. Business Logic Layer (Services):**
```php
// app/Services/UserService.php
class UserService
{
    public function __construct(
        private UserRepository $userRepository,
        private EmailService $emailService
    ) {}
    
    public function createUser(array $data): User
    {
        $user = $this->userRepository->create([
            'name' => $data['name'],
            'email' => $data['email'],
            'password' => Hash::make($data['password']),
        ]);
        
        $this->emailService->sendWelcomeEmail($user);
        
        return $user;
    }
}
```

**3. Data Access Layer (Repositories):**
```php
// app/Repositories/UserRepository.php
class UserRepository
{
    public function create(array $data): User
    {
        return User::create($data);
    }
    
    public function findByEmail(string $email): ?User
    {
        return User::where('email', $email)->first();
    }
}
```

**Преимущества:**
- ✅ Separation of Concerns
- ✅ Testability (моки для каждого слоя)
- ✅ Reusability

**Недостатки:**
- ❌ Overhead для простых приложений
- ❌ Cascading changes (изменение в DB → все слои)

---

## 🎨 MVC - Model-View-Controller

### Концепция

**Laravel по умолчанию использует MVC.**

```
User Request
    ↓
[Controller] ←→ [Model]
    ↓              ↓
  [View]  ←────────┘
    ↓
Response
```

**Компоненты:**
- **Model** - данные и бизнес-логика
- **View** - представление (Blade templates)
- **Controller** - обработка запросов, связь Model↔View

### Реализация

**Model:**
```php
// app/Models/Post.php
class Post extends Model
{
    protected $fillable = ['title', 'content', 'user_id'];
    
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }
    
    public function scopePublished($query)
    {
        return $query->where('published', true);
    }
}
```

**View:**
```blade
{{-- resources/views/posts/index.blade.php --}}
@extends('layouts.app')

@section('content')
    <h1>Posts</h1>
    
    @foreach($posts as $post)
        <article>
            <h2>{{ $post->title }}</h2>
            <p>{{ $post->content }}</p>
            <small>By {{ $post->user->name }}</small>
        </article>
    @endforeach
@endsection
```

**Controller:**
```php
// app/Http/Controllers/PostController.php
class PostController extends Controller
{
    public function index()
    {
        $posts = Post::with('user')->published()->get();
        
        return view('posts.index', compact('posts'));
    }
    
    public function store(Request $request)
    {
        $validated = $request->validate([
            'title' => 'required|max:255',
            'content' => 'required',
        ]);
        
        $post = auth()->user()->posts()->create($validated);
        
        return redirect()->route('posts.show', $post);
    }
}
```

**Проблема Fat Controllers:**
```php
// ❌ Плохо - вся логика в контроллере
class UserController extends Controller
{
    public function register(Request $request)
    {
        $validated = $request->validate([...]);
        
        $user = User::create([
            'name' => $validated['name'],
            'email' => $validated['email'],
            'password' => Hash::make($validated['password']),
        ]);
        
        $profile = Profile::create(['user_id' => $user->id]);
        
        Mail::to($user->email)->send(new WelcomeEmail($user));
        
        event(new UserRegistered($user));
        
        Log::info('User registered', ['user_id' => $user->id]);
        
        return redirect('/dashboard');
    }
}

// ✅ Хорошо - тонкий контроллер + Service Layer
class UserController extends Controller
{
    public function __construct(
        private UserService $userService
    ) {}
    
    public function register(RegisterRequest $request)
    {
        $user = $this->userService->registerUser($request->validated());
        
        return redirect('/dashboard');
    }
}
```

---

## 📱 MVP - Model-View-Presenter

### Концепция

**Эволюция MVC** с более четким разделением.

```
[View] ←→ [Presenter] ←→ [Model]
```

**Отличия от MVC:**
- View НЕ знает о Model (только Presenter)
- Presenter содержит presentation logic
- View максимально "глупый" (только отображение)

### Реализация

**View Interface:**
```php
interface PostListView
{
    public function showPosts(array $posts): void;
    public function showError(string $message): void;
    public function showLoading(): void;
}
```

**Presenter:**
```php
class PostListPresenter
{
    public function __construct(
        private PostListView $view,
        private PostRepository $repository
    ) {}
    
    public function loadPosts(): void
    {
        $this->view->showLoading();
        
        try {
            $posts = $this->repository->getPublished();
            $this->view->showPosts($posts);
        } catch (\Exception $e) {
            $this->view->showError('Failed to load posts');
        }
    }
}
```

**View (Livewire component):**
```php
class PostList extends Component implements PostListView
{
    public array $posts = [];
    public string $error = '';
    public bool $loading = false;
    
    private PostListPresenter $presenter;
    
    public function mount()
    {
        $this->presenter = new PostListPresenter($this, app(PostRepository::class));
        $this->presenter->loadPosts();
    }
    
    public function showPosts(array $posts): void
    {
        $this->posts = $posts;
        $this->loading = false;
    }
    
    public function showError(string $message): void
    {
        $this->error = $message;
        $this->loading = false;
    }
    
    public function showLoading(): void
    {
        $this->loading = true;
    }
}
```

**Преимущества:**
- ✅ Testability (Presenter легко тестировать)
- ✅ View не зависит от Model

**Недостатки:**
- ❌ Больше boilerplate кода
- ❌ Редко используется в веб-приложениях (чаще в мобильных)

---

## 🎯 MVVM - Model-View-ViewModel

### Концепция

```
[View] ←→ [ViewModel] ←→ [Model]
         (two-way binding)
```

**ViewModel:**
- Представление данных для View
- Two-way data binding
- Популярно в Vue.js, Angular, React

### Реализация с Livewire

**ViewModel (Livewire Component):**
```php
class CreatePost extends Component
{
    // Two-way binding
    public string $title = '';
    public string $content = '';
    public array $tags = [];
    
    protected $rules = [
        'title' => 'required|max:255',
        'content' => 'required|min:10',
    ];
    
    public function save()
    {
        $validated = $this->validate();
        
        $post = Post::create([
            'title' => $validated['title'],
            'content' => $validated['content'],
            'user_id' => auth()->id(),
        ]);
        
        $post->tags()->sync($this->tags);
        
        session()->flash('message', 'Post created successfully!');
        
        return redirect()->route('posts.index');
    }
}
```

**View:**
```blade
<form wire:submit.prevent="save">
    {{-- Two-way binding --}}
    <input wire:model="title" type="text" />
    @error('title') <span>{{ $message }}</span> @enderror
    
    <textarea wire:model="content"></textarea>
    @error('content') <span>{{ $message }}</span> @enderror
    
    <button type="submit">Save</button>
</form>
```

---

## 🏢 Microservices Architecture

### Концепция

Разбиение приложения на **независимые сервисы**.

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   User       │    │   Order      │    │   Payment    │
│   Service    │    │   Service    │    │   Service    │
│              │    │              │    │              │
│  DB: users   │    │  DB: orders  │    │  DB: payments│
└──────┬───────┘    └──────┬───────┘    └──────┬───────┘
       │                   │                   │
       └───────────────────┴───────────────────┘
                           │
                    ┌──────▼───────┐
                    │  API Gateway │
                    └──────────────┘
```

**Характеристики:**
- Каждый сервис = отдельное приложение
- Собственная БД для каждого сервиса
- Коммуникация через API (REST/gRPC)
- Независимое развертывание

### Реализация

**User Service:**
```php
// User Service (Laravel app #1)
Route::group(['prefix' => 'api/users'], function () {
    Route::get('/{id}', [UserController::class, 'show']);
    Route::post('/', [UserController::class, 'create']);
});

class UserController extends Controller
{
    public function show(int $id): JsonResponse
    {
        $user = User::findOrFail($id);
        return response()->json($user);
    }
}
```

**Order Service (вызывает User Service):**
```php
// Order Service (Laravel app #2)
class OrderService
{
    public function __construct(
        private HttpClient $httpClient
    ) {}
    
    public function createOrder(int $userId, array $items): Order
    {
        // Вызов User Service
        $response = $this->httpClient->get("http://user-service/api/users/{$userId}");
        
        if (!$response->successful()) {
            throw new UserNotFoundException();
        }
        
        $user = $response->json();
        
        $order = Order::create([
            'user_id' => $userId,
            'items' => $items,
            'total' => $this->calculateTotal($items),
        ]);
        
        // Event для Payment Service
        event(new OrderCreated($order));
        
        return $order;
    }
}
```

**Inter-service communication:**

1. **Synchronous (REST/gRPC):**
```php
$response = Http::get('http://product-service/api/products/123');
$product = $response->json();
```

2. **Asynchronous (Message Queue):**
```php
// Order Service публикует событие
event(new OrderCreated($order));

// Payment Service подписан на событие
class PaymentListener
{
    public function handle(OrderCreated $event)
    {
        $this->processPayment($event->order);
    }
}
```

**Преимущества:**
- ✅ Scalability (масштабируй только нужные сервисы)
- ✅ Independent deployment
- ✅ Technology diversity (разные языки/фреймворки)
- ✅ Fault isolation

**Недостатки:**
- ❌ Сложность (распределенные системы)
- ❌ Network latency
- ❌ Distributed transactions (eventual consistency)
- ❌ Debugging сложнее

**Когда использовать:**
- ✅ Большие команды (>50 developers)
- ✅ Разные части системы масштабируются по-разному
- ✅ Частые релизы

**Когда НЕ использовать:**
- ❌ Маленькая команда (<10 developers)
- ❌ Простое приложение
- ❌ Нет опыта с distributed systems

---

## 📨 Event-Driven Architecture

### Концепция

Компоненты взаимодействуют через **события (events)**.

```
[User Service] ─── UserCreated ───> [Email Service]
                                 └─> [Notification Service]
                                 └─> [Analytics Service]
```

**Характеристики:**
- Loose coupling (сервисы не знают друг о друге)
- Async processing
- Event store (история событий)

### Реализация

**Event Publisher:**
```php
class UserService
{
    public function createUser(array $data): User
    {
        $user = User::create($data);
        
        // Публикация события
        event(new UserCreated($user));
        
        return $user;
    }
}
```

**Event Subscribers:**
```php
// Email Service
class SendWelcomeEmail
{
    public function handle(UserCreated $event)
    {
        Mail::to($event->user->email)->send(new WelcomeEmail($event->user));
    }
}

// Notification Service
class CreateNotification
{
    public function handle(UserCreated $event)
    {
        Notification::create([
            'user_id' => $event->user->id,
            'message' => 'Welcome to our platform!',
        ]);
    }
}

// Analytics Service
class TrackUserRegistration
{
    public function handle(UserCreated $event)
    {
        Analytics::track('user_registered', [
            'user_id' => $event->user->id,
            'source' => $event->user->registration_source,
        ]);
    }
}
```

**Event Bus (RabbitMQ/Kafka):**
```php
// Publisher
class EventPublisher
{
    public function publish(string $eventName, array $payload): void
    {
        RabbitMQ::publish('events', $eventName, $payload);
    }
}

// Subscriber
class EventSubscriber
{
    public function subscribe(string $eventName, callable $handler): void
    {
        RabbitMQ::subscribe('events', $eventName, $handler);
    }
}
```

**Преимущества:**
- ✅ Decoupling
- ✅ Scalability (async processing)
- ✅ Extensibility (легко добавить новые подписчики)

**Недостатки:**
- ❌ Eventual consistency
- ❌ Debugging сложнее (event flow)
- ❌ Event versioning

---

## 🔀 CQRS - Command Query Responsibility Segregation

### Концепция

Разделение **чтения (Query)** и **записи (Command)**.

```
Commands (Write)              Queries (Read)
     ↓                             ↓
[Write Model]                 [Read Model]
     ↓                             ↑
[Write DB]  ───sync/async───> [Read DB]
```

**Зачем:**
- Оптимизация для разных нагрузок
- Read DB может быть денормализована
- Разные модели для чтения/записи

### Реализация

**Command (запись):**
```php
// Command
class CreateUserCommand
{
    public function __construct(
        public string $name,
        public string $email,
        public string $password
    ) {}
}

// Command Handler
class CreateUserHandler
{
    public function handle(CreateUserCommand $command): void
    {
        $user = User::create([
            'name' => $command->name,
            'email' => $command->email,
            'password' => Hash::make($command->password),
        ]);
        
        // Обновить Read Model
        UserReadModel::create([
            'id' => $user->id,
            'name' => $user->name,
            'email' => $user->email,
            'registered_at' => now(),
        ]);
    }
}
```

**Query (чтение):**
```php
// Query
class GetUserByIdQuery
{
    public function __construct(
        public int $userId
    ) {}
}

// Query Handler
class GetUserByIdHandler
{
    public function handle(GetUserByIdQuery $query): ?UserReadModel
    {
        return UserReadModel::find($query->userId);
    }
}
```

**Bus:**
```php
class CommandBus
{
    public function dispatch(object $command): void
    {
        $handler = $this->resolveHandler($command);
        $handler->handle($command);
    }
}

class QueryBus
{
    public function ask(object $query): mixed
    {
        $handler = $this->resolveHandler($query);
        return $handler->handle($query);
    }
}
```

**Использование:**
```php
// Write
$commandBus->dispatch(new CreateUserCommand(
    name: 'John Doe',
    email: 'john@example.com',
    password: 'secret'
));

// Read
$user = $queryBus->ask(new GetUserByIdQuery(userId: 123));
```

**Преимущества:**
- ✅ Оптимизация для чтения/записи отдельно
- ✅ Scalability (масштабируй read/write независимо)
- ✅ Separation of concerns

**Недостатки:**
- ❌ Complexity
- ❌ Eventual consistency (sync между write/read DB)

---

## 📜 Event Sourcing

### Концепция

Хранить **события (events)** вместо текущего состояния.

```
State = f(events)

UserCreated(name: "John")
EmailChanged(email: "new@example.com")
PasswordChanged(hash: "...")
→ Current State: {name: "John", email: "new@example.com", password: "..."}
```

**Event Store:**
```
| id | aggregate_id | event_type     | payload                | created_at |
|----|--------------|----------------|------------------------|------------|
| 1  | user-123     | UserCreated    | {name: "John", ...}    | 2024-01-01 |
| 2  | user-123     | EmailChanged   | {email: "new@..."}     | 2024-01-02 |
| 3  | user-123     | PasswordChanged| {hash: "..."}          | 2024-01-03 |
```

### Реализация

**Event Store:**
```php
class EventStore
{
    public function append(string $aggregateId, Event $event): void
    {
        DB::table('event_store')->insert([
            'aggregate_id' => $aggregateId,
            'event_type' => get_class($event),
            'payload' => json_encode($event),
            'created_at' => now(),
        ]);
    }
    
    public function getEvents(string $aggregateId): Collection
    {
        return DB::table('event_store')
            ->where('aggregate_id', $aggregateId)
            ->orderBy('id')
            ->get()
            ->map(fn($row) => $this->deserializeEvent($row));
    }
}
```

**Aggregate:**
```php
class User
{
    private string $id;
    private string $name;
    private string $email;
    private array $uncommittedEvents = [];
    
    public static function create(string $id, string $name, string $email): self
    {
        $user = new self();
        $user->recordThat(new UserCreated($id, $name, $email));
        return $user;
    }
    
    public function changeEmail(string $email): void
    {
        $this->recordThat(new EmailChanged($this->id, $email));
    }
    
    private function recordThat(Event $event): void
    {
        $this->uncommittedEvents[] = $event;
        $this->apply($event);
    }
    
    private function apply(Event $event): void
    {
        match (get_class($event)) {
            UserCreated::class => $this->applyUserCreated($event),
            EmailChanged::class => $this->applyEmailChanged($event),
        };
    }
    
    private function applyUserCreated(UserCreated $event): void
    {
        $this->id = $event->id;
        $this->name = $event->name;
        $this->email = $event->email;
    }
    
    private function applyEmailChanged(EmailChanged $event): void
    {
        $this->email = $event->email;
    }
    
    public function getUncommittedEvents(): array
    {
        return $this->uncommittedEvents;
    }
}
```

**Repository:**
```php
class UserRepository
{
    public function __construct(
        private EventStore $eventStore
    ) {}
    
    public function save(User $user): void
    {
        foreach ($user->getUncommittedEvents() as $event) {
            $this->eventStore->append($user->id, $event);
        }
    }
    
    public function find(string $id): User
    {
        $events = $this->eventStore->getEvents($id);
        
        $user = new User();
        foreach ($events as $event) {
            $user->apply($event);
        }
        
        return $user;
    }
}
```

**Преимущества:**
- ✅ Complete audit log (вся история изменений)
- ✅ Time travel (состояние на любой момент времени)
- ✅ Event replay (пересоздать read model)

**Недостатки:**
- ❌ Сложность
- ❌ Performance (нужен snapshot для больших aggregates)
- ❌ Event versioning при изменениях

---

## 📊 Сравнительная таблица

| Pattern | Use Case | Complexity | Scalability |
|---------|----------|------------|-------------|
| Layered | Средние приложения | Средняя | Средняя |
| MVC | Веб-приложения | Низкая | Средняя |
| Microservices | Большие системы | Высокая | Очень высокая |
| Event-Driven | Async processing | Средняя | Высокая |
| CQRS | Разные нагрузки read/write | Высокая | Высокая |
| Event Sourcing | Audit log, temporal queries | Очень высокая | Средняя |

---

## 🎓 Для собеседования: ключевые точки

1. **Layered** - Presentation → Business → Data Access
2. **MVC** - Model-View-Controller, Laravel по умолчанию
3. **Microservices** - независимые сервисы, своя БД, REST/gRPC
4. **Event-Driven** - loose coupling через события
5. **CQRS** - разделение read/write моделей
6. **Event Sourcing** - хранение событий вместо состояния
7. **Когда использовать** - Microservices для больших команд, CQRS для read-heavy, Event Sourcing для audit
8. **Trade-offs** - сложность vs scalability

**Главное:** Выбирай простейшую архитектуру для своих требований, не переусложняй.

# Porto Architecture (Laravel)

Полный разбор Porto Software Architectural Pattern - модульная архитектура специально для Laravel.

---

## 🎯 Что такое Porto?

**Porto** - Software Architectural Pattern, созданный **Mahmoud Zalt** специально для Laravel.

**Цель:** Организация больших Laravel приложений с **модульным подходом** и принципами **DDD**.

**Отличия от MVC:**
- MVC: `app/Models`, `app/Controllers` (по типу файлов)
- Porto: `app/Containers/User`, `app/Containers/Order` (по фичам)

```
Porto = MVC + DDD + Modularity
```

---

## 🏗️ Структура Porto

```
app/
├── Ship/                           ← Shared code (переиспользуемый)
│   ├── Core/
│   │   ├── Abstracts/
│   │   │   ├── Controllers/
│   │   │   │   └── ApiController.php
│   │   │   ├── Requests/
│   │   │   │   └── Request.php
│   │   │   ├── Models/
│   │   │   │   └── Model.php
│   │   │   └── Actions/
│   │   │       └── Action.php
│   │   ├── Exceptions/
│   │   │   └── CoreException.php
│   │   └── Traits/
│   │       ├── HashIdTrait.php
│   │       └── ResponseTrait.php
│   └── Middlewares/
│       └── RateLimitMiddleware.php
│
└── Containers/                     ← Feature containers (фичи)
    ├── User/
    │   ├── Actions/               ← Business logic
    │   │   ├── CreateUserAction.php
    │   │   ├── UpdateUserAction.php
    │   │   └── DeleteUserAction.php
    │   ├── Tasks/                 ← Reusable operations
    │   │   ├── CreateUserTask.php
    │   │   ├── FindUserByIdTask.php
    │   │   └── SendWelcomeEmailTask.php
    │   ├── Models/
    │   │   └── User.php
    │   ├── Data/
    │   │   ├── Migrations/
    │   │   ├── Seeders/
    │   │   └── Repositories/
    │   │       └── UserRepository.php
    │   ├── UI/
    │   │   ├── API/
    │   │   │   ├── Controllers/
    │   │   │   │   └── UserController.php
    │   │   │   ├── Requests/
    │   │   │   │   ├── CreateUserRequest.php
    │   │   │   │   └── UpdateUserRequest.php
    │   │   │   ├── Routes/
    │   │   │   │   └── api.php
    │   │   │   └── Transformers/
    │   │   │       └── UserTransformer.php
    │   │   └── WEB/
    │   │       ├── Controllers/
    │   │       ├── Views/
    │   │       └── Routes/
    │   ├── Tests/
    │   │   └── Unit/
    │   │       └── CreateUserActionTest.php
    │   ├── Configs/
    │   │   └── user.php
    │   └── Events/
    │       └── UserCreatedEvent.php
    │
    └── Order/
        ├── Actions/
        ├── Tasks/
        ├── Models/
        └── ...
```

---

## 🧩 Основные компоненты

### 1️⃣ Ship (Корабль)

**Shared code** - переиспользуемый код для всех Containers.

**Ship/Core/Abstracts/Controllers/ApiController.php:**
```php
namespace App\Ship\Core\Abstracts\Controllers;

use App\Ship\Core\Traits\ResponseTrait;
use Illuminate\Foundation\Auth\Access\AuthorizesRequests;
use Illuminate\Foundation\Validation\ValidatesRequests;
use Illuminate\Routing\Controller;

abstract class ApiController extends Controller
{
    use AuthorizesRequests, ValidatesRequests, ResponseTrait;
    
    protected function created($data, $message = 'Resource created successfully')
    {
        return $this->json($data, 201, $message);
    }
    
    protected function deleted($message = 'Resource deleted successfully')
    {
        return $this->json(null, 200, $message);
    }
}
```

**Ship/Core/Traits/ResponseTrait.php:**
```php
namespace App\Ship\Core\Traits;

use Illuminate\Http\JsonResponse;

trait ResponseTrait
{
    protected function json($data, int $status = 200, ?string $message = null): JsonResponse
    {
        $response = [
            'success' => $status >= 200 && $status < 300,
            'data' => $data,
        ];
        
        if ($message) {
            $response['message'] = $message;
        }
        
        return response()->json($response, $status);
    }
}
```

### 2️⃣ Containers (Контейнеры)

**Self-contained modules** - каждый Container = одна фича (User, Order, Payment).

### 3️⃣ Actions

**Business logic** - главная бизнес-логика фичи.

**Containers/User/Actions/CreateUserAction.php:**
```php
namespace App\Containers\User\Actions;

use App\Containers\User\Models\User;
use App\Containers\User\Tasks\CreateUserTask;
use App\Containers\User\Tasks\SendWelcomeEmailTask;
use App\Ship\Core\Abstracts\Actions\Action;

class CreateUserAction extends Action
{
    public function __construct(
        private CreateUserTask $createUserTask,
        private SendWelcomeEmailTask $sendWelcomeEmailTask
    ) {}
    
    public function run(array $data): User
    {
        // Validate unique email
        if (User::where('email', $data['email'])->exists()) {
            throw new \Exception('Email already exists');
        }
        
        // Create user (Task)
        $user = $this->createUserTask->run($data);
        
        // Send welcome email (Task)
        $this->sendWelcomeEmailTask->run($user);
        
        // Dispatch event
        event(new \App\Containers\User\Events\UserCreatedEvent($user));
        
        return $user;
    }
}
```

**Правила Actions:**
- Один Action = один use case
- Orchestrates Tasks (не содержит низкоуровневых операций)
- Вызывается из Controllers

### 4️⃣ Tasks

**Reusable operations** - переиспользуемые операции.

**Containers/User/Tasks/CreateUserTask.php:**
```php
namespace App\Containers\User\Tasks;

use App\Containers\User\Data\Repositories\UserRepository;
use App\Containers\User\Models\User;
use Illuminate\Support\Facades\Hash;

class CreateUserTask
{
    public function __construct(
        private UserRepository $repository
    ) {}
    
    public function run(array $data): User
    {
        return $this->repository->create([
            'name' => $data['name'],
            'email' => $data['email'],
            'password' => Hash::make($data['password']),
        ]);
    }
}
```

**Containers/User/Tasks/FindUserByIdTask.php:**
```php
namespace App\Containers\User\Tasks;

use App\Containers\User\Data\Repositories\UserRepository;
use App\Containers\User\Models\User;

class FindUserByIdTask
{
    public function __construct(
        private UserRepository $repository
    ) {}
    
    public function run(int $id): ?User
    {
        return $this->repository->find($id);
    }
}
```

**Containers/User/Tasks/SendWelcomeEmailTask.php:**
```php
namespace App\Containers\User\Tasks;

use App\Containers\User\Models\User;
use Illuminate\Support\Facades\Mail;

class SendWelcomeEmailTask
{
    public function run(User $user): void
    {
        Mail::to($user->email)->send(new \App\Containers\User\Mails\WelcomeEmail($user));
    }
}
```

**Правила Tasks:**
- Простые операции (create, find, send email)
- Переиспользуются между Actions
- НЕ вызывают другие Tasks

### 5️⃣ Controllers

**HTTP layer** - обрабатывают запросы, вызывают Actions.

**Containers/User/UI/API/Controllers/UserController.php:**
```php
namespace App\Containers\User\UI\API\Controllers;

use App\Containers\User\Actions\CreateUserAction;
use App\Containers\User\Actions\UpdateUserAction;
use App\Containers\User\Actions\DeleteUserAction;
use App\Containers\User\UI\API\Requests\CreateUserRequest;
use App\Containers\User\UI\API\Requests\UpdateUserRequest;
use App\Containers\User\UI\API\Transformers\UserTransformer;
use App\Containers\User\Tasks\FindUserByIdTask;
use App\Ship\Core\Abstracts\Controllers\ApiController;

class UserController extends ApiController
{
    public function store(CreateUserRequest $request, CreateUserAction $action)
    {
        $user = $action->run($request->validated());
        
        return $this->created(new UserTransformer($user));
    }
    
    public function update(int $id, UpdateUserRequest $request, UpdateUserAction $action)
    {
        $user = $action->run($id, $request->validated());
        
        return $this->json(new UserTransformer($user));
    }
    
    public function show(int $id, FindUserByIdTask $task)
    {
        $user = $task->run($id);
        
        if (!$user) {
            return $this->json(null, 404, 'User not found');
        }
        
        return $this->json(new UserTransformer($user));
    }
    
    public function destroy(int $id, DeleteUserAction $action)
    {
        $action->run($id);
        
        return $this->deleted();
    }
}
```

**Правила Controllers:**
- Тонкие (thin) - только валидация и вызов Actions
- НЕ содержат бизнес-логику

### 6️⃣ Requests

**Validation** - правила валидации.

**Containers/User/UI/API/Requests/CreateUserRequest.php:**
```php
namespace App\Containers\User\UI\API\Requests;

use App\Ship\Core\Abstracts\Requests\Request;

class CreateUserRequest extends Request
{
    public function authorize(): bool
    {
        return true;
    }
    
    public function rules(): array
    {
        return [
            'name' => 'required|string|max:255',
            'email' => 'required|email|unique:users,email',
            'password' => 'required|string|min:8|confirmed',
        ];
    }
}
```

### 7️⃣ Transformers (Fractal)

**Data transformation** - форматирование ответов API.

**Containers/User/UI/API/Transformers/UserTransformer.php:**
```php
namespace App\Containers\User\UI\API\Transformers;

use App\Containers\User\Models\User;
use League\Fractal\TransformerAbstract;

class UserTransformer extends TransformerAbstract
{
    public function transform(User $user): array
    {
        return [
            'id' => $user->id,
            'name' => $user->name,
            'email' => $user->email,
            'created_at' => $user->created_at->toIso8601String(),
        ];
    }
}
```

### 8️⃣ Routes

**API routes** внутри Container.

**Containers/User/UI/API/Routes/api.php:**
```php
use App\Containers\User\UI\API\Controllers\UserController;
use Illuminate\Support\Facades\Route;

Route::prefix('users')->group(function () {
    Route::get('/', [UserController::class, 'index']);
    Route::post('/', [UserController::class, 'store']);
    Route::get('/{id}', [UserController::class, 'show']);
    Route::put('/{id}', [UserController::class, 'update']);
    Route::delete('/{id}', [UserController::class, 'destroy']);
});
```

**Регистрация в RouteServiceProvider:**
```php
// app/Providers/RouteServiceProvider.php
public function boot(): void
{
    $this->routes(function () {
        // Load all Container routes
        $containers = File::directories(app_path('Containers'));
        
        foreach ($containers as $container) {
            $apiRoutes = $container . '/UI/API/Routes/api.php';
            
            if (File::exists($apiRoutes)) {
                Route::prefix('api')
                    ->middleware('api')
                    ->group($apiRoutes);
            }
        }
    });
}
```

### 9️⃣ Models

**Eloquent models** - обычные Laravel модели.

**Containers/User/Models/User.php:**
```php
namespace App\Containers\User\Models;

use App\Ship\Core\Abstracts\Models\Model;
use Illuminate\Foundation\Auth\User as Authenticatable;

class User extends Authenticatable
{
    protected $fillable = [
        'name',
        'email',
        'password',
    ];
    
    protected $hidden = [
        'password',
        'remember_token',
    ];
}
```

### 🔟 Repositories

**Data access layer** - работа с БД.

**Containers/User/Data/Repositories/UserRepository.php:**
```php
namespace App\Containers\User\Data\Repositories;

use App\Containers\User\Models\User;

class UserRepository
{
    public function create(array $data): User
    {
        return User::create($data);
    }
    
    public function find(int $id): ?User
    {
        return User::find($id);
    }
    
    public function update(User $user, array $data): User
    {
        $user->update($data);
        return $user->fresh();
    }
    
    public function delete(User $user): void
    {
        $user->delete();
    }
}
```

---

## 🔄 Полный пример: Update User

### 1. Request

```php
namespace App\Containers\User\UI\API\Requests;

class UpdateUserRequest extends Request
{
    public function rules(): array
    {
        return [
            'name' => 'sometimes|string|max:255',
            'email' => 'sometimes|email|unique:users,email,' . $this->route('id'),
        ];
    }
}
```

### 2. Controller

```php
public function update(int $id, UpdateUserRequest $request, UpdateUserAction $action)
{
    $user = $action->run($id, $request->validated());
    
    return $this->json(new UserTransformer($user));
}
```

### 3. Action

```php
namespace App\Containers\User\Actions;

class UpdateUserAction extends Action
{
    public function __construct(
        private FindUserByIdTask $findUserTask,
        private UpdateUserTask $updateUserTask
    ) {}
    
    public function run(int $id, array $data): User
    {
        $user = $this->findUserTask->run($id);
        
        if (!$user) {
            throw new \Exception('User not found');
        }
        
        $user = $this->updateUserTask->run($user, $data);
        
        event(new UserUpdatedEvent($user));
        
        return $user;
    }
}
```

### 4. Tasks

```php
// FindUserByIdTask
class FindUserByIdTask
{
    public function run(int $id): ?User
    {
        return User::find($id);
    }
}

// UpdateUserTask
class UpdateUserTask
{
    public function __construct(
        private UserRepository $repository
    ) {}
    
    public function run(User $user, array $data): User
    {
        return $this->repository->update($user, $data);
    }
}
```

---

## 📊 Porto vs MVC vs DDD

| Aspect | Laravel MVC | Porto | DDD |
|--------|-------------|-------|-----|
| **Организация** | По типу (Models, Controllers) | По фичам (Containers) | По доменам (Bounded Contexts) |
| **Complexity** | Низкая | Средняя | Высокая |
| **Scalability** | Средняя | Высокая | Очень высокая |
| **Learning Curve** | Низкая | Средняя | Высокая |
| **Testability** | Средняя | Высокая | Очень высокая |
| **Use Case** | Малые/средние проекты | Большие Laravel проекты | Enterprise приложения |

---

## ✅ Преимущества Porto

1. **Modularity** - каждый Container независим
2. **Scalability** - добавляй Containers без影响 других
3. **Team Organization** - разные команды работают над разными Containers
4. **Code Reusability** - Tasks переиспользуются
5. **Testability** - каждый компонент легко тестировать
6. **Clear Structure** - понятная организация больших проектов

---

## ❌ Недостатки Porto

1. **Complexity** - больше файлов и слоев
2. **Learning Curve** - новички должны изучить структуру
3. **Overhead** - для маленьких проектов избыточно
4. **Boilerplate** - много кода для простых CRUD
5. **Not Standard** - не все developers знакомы с Porto

---

## 🎯 Когда использовать Porto?

### ✅ Используй когда:

- **Большое Laravel приложение** (>100k lines of code)
- **Много фич** (User, Order, Payment, Inventory, Shipping, etc.)
- **Большая команда** (>10 developers)
- **Долгосрочный проект** (5+ years)
- **Micro-frontend** approach (разные UI для одного API)

### ❌ НЕ используй когда:

- **Маленький проект** (<10k lines)
- **Простой CRUD**
- **MVP / Prototype**
- **Solo developer**
- **Стандартный MVC достаточен**

---

## 🚀 Миграция с MVC на Porto

### Было (MVC):

```
app/
├── Http/Controllers/
│   └── UserController.php
├── Models/
│   └── User.php
└── Services/
    └── UserService.php
```

### Стало (Porto):

```
app/
├── Ship/
│   └── Core/Abstracts/...
└── Containers/
    └── User/
        ├── Actions/
        │   └── CreateUserAction.php
        ├── Tasks/
        │   ├── CreateUserTask.php
        │   └── FindUserTask.php
        ├── Models/
        │   └── User.php
        └── UI/API/
            ├── Controllers/
            │   └── UserController.php
            └── Routes/
                └── api.php
```

**Шаги миграции:**

1. **Создать Ship** с базовыми абстракциями
2. **Создать Container** для первой фичи
3. **Переместить Model** в Container
4. **Разбить Service** на Action + Tasks
5. **Переместить Controller** в UI/API
6. **Переместить Routes** в Container
7. **Повторить** для других фич

---

## 🧪 Тестирование

### Unit Test (Task):

```php
namespace App\Containers\User\Tests\Unit;

use App\Containers\User\Tasks\CreateUserTask;
use Tests\TestCase;

class CreateUserTaskTest extends TestCase
{
    public function test_creates_user()
    {
        $task = app(CreateUserTask::class);
        
        $user = $task->run([
            'name' => 'John Doe',
            'email' => 'john@example.com',
            'password' => 'secret123',
        ]);
        
        $this->assertEquals('john@example.com', $user->email);
        $this->assertDatabaseHas('users', ['email' => 'john@example.com']);
    }
}
```

### Feature Test (Action):

```php
namespace App\Containers\User\Tests\Feature;

use App\Containers\User\Actions\CreateUserAction;
use Tests\TestCase;

class CreateUserActionTest extends TestCase
{
    public function test_creates_user_and_sends_email()
    {
        Mail::fake();
        
        $action = app(CreateUserAction::class);
        
        $user = $action->run([
            'name' => 'John',
            'email' => 'john@example.com',
            'password' => 'secret',
        ]);
        
        $this->assertDatabaseHas('users', ['email' => 'john@example.com']);
        Mail::assertSent(WelcomeEmail::class);
    }
}
```

### API Test (Controller):

```php
namespace App\Containers\User\Tests\Feature;

use Tests\TestCase;

class UserControllerTest extends TestCase
{
    public function test_creates_user_via_api()
    {
        $response = $this->postJson('/api/users', [
            'name' => 'John Doe',
            'email' => 'john@example.com',
            'password' => 'secret123',
            'password_confirmation' => 'secret123',
        ]);
        
        $response->assertStatus(201)
            ->assertJsonStructure([
                'success',
                'data' => ['id', 'name', 'email'],
            ]);
    }
}
```

---

## 📚 Ресурсы

- [Porto SAP Official Docs](https://github.com/Mahmoudz/Porto)
- [Apiato](https://github.com/apiato/apiato) - Full Porto framework (готовое решение)

---

## 🎓 Для собеседования: ключевые точки

1. **Porto** - модульная архитектура для Laravel по фичам
2. **Ship** - shared code, Containers - фичи
3. **Actions** - business logic (use cases)
4. **Tasks** - переиспользуемые операции
5. **Containers** - самодостаточные модули (User, Order)
6. **Преимущества** - modularity, scalability, team organization
7. **Когда использовать** - большие проекты, большие команды
8. **vs MVC** - по фичам vs по типу файлов

**Главное:** Porto организует код по **фичам** (Containers), а не по **типам** (Models/Controllers), что лучше для больших проектов.

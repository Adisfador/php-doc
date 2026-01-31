# Clean Code - Чистый код

Полный разбор принципов Clean Code от Robert Martin (Uncle Bob): именование, функции, комментарии, форматирование, SOLID.

---

## 📖 Что такое Clean Code?

**Clean Code** - код, который легко читать, понимать и модифицировать.

**Автор:** Robert C. Martin (Uncle Bob) - книга "Clean Code" (2008)

**Зачем:**
- Код читают в 10 раз чаще, чем пишут
- Легче поддерживать и расширять
- Меньше багов
- Быстрее onboarding новых developers

**Главный принцип:**
> "Any fool can write code that a computer can understand. Good programmers write code that humans can understand."

---

## 🏷️ Meaningful Names (Осмысленные имена)

### 1. Intention-Revealing Names

**Имя должно объяснять, что делает переменная/функция.**

```php
// ❌ Плохо
$d; // elapsed time in days
$list;
$theList;

// ✅ Хорошо
$elapsedTimeInDays;
$customers;
$activeCustomers;
```

### 2. Avoid Disinformation

**Не вводи в заблуждение.**

```php
// ❌ Плохо - это не array
$accountList; // на самом деле Collection

// ✅ Хорошо
$accounts;
$accountCollection;
```

### 3. Pronounceable Names

**Имена должны произноситься.**

```php
// ❌ Плохо
$genymdhms; // generation year month day hour minute second
$modymdhms;

// ✅ Хорошо
$generationTimestamp;
$modificationTimestamp;
```

### 4. Searchable Names

**Имена должны легко находиться.**

```php
// ❌ Плохо - как найти все места, где используется 7?
for ($j = 0; $j < 34; $j++) {
    $s += ($t[$j] * 4) / 5;
}

// ✅ Хорошо
const WORK_DAYS_PER_WEEK = 5;
const NUMBER_OF_TASKS = 34;

$sum = 0;
for ($taskIndex = 0; $taskIndex < NUMBER_OF_TASKS; $taskIndex++) {
    $sum += ($tasks[$taskIndex] * 4) / WORK_DAYS_PER_WEEK;
}
```

### 5. Avoid Mental Mapping

**Не заставляй читателя переводить в уме.**

```php
// ❌ Плохо - что такое $r?
foreach ($users as $r) {
    // ... 50 lines
    echo $r->name; // что такое $r?
}

// ✅ Хорошо
foreach ($users as $user) {
    // ...
    echo $user->name;
}
```

### 6. Class Names - существительные

```php
// ❌ Плохо - глаголы
class ProcessData {}
class Manager {}
class Info {}

// ✅ Хорошо - существительные
class Customer {}
class Account {}
class PaymentProcessor {}
class OrderManager {}
```

### 7. Method Names - глаголы

```php
// ❌ Плохо
public function user() {}
public function name() {}

// ✅ Хорошо
public function getUser() {}
public function setName($name) {}
public function isActive() {}
public function hasPermission($permission) {}
public function deletePage($page) {}
```

### 8. Pick One Word Per Concept

**Одна концепция = одно слово.**

```php
// ❌ Плохо - fetch, retrieve, get для одной концепции
$user = $userRepository->fetch($id);
$order = $orderRepository->retrieve($id);
$product = $productRepository->get($id);

// ✅ Хорошо - везде get
$user = $userRepository->get($id);
$order = $orderRepository->get($id);
$product = $productRepository->get($id);
```

### 9. Add Meaningful Context

**Добавляй контекст через префиксы/классы.**

```php
// ❌ Плохо - что за state?
$state = 'CA';

// ✅ Хорошо - понятен контекст
$addressState = 'CA';

// ✅ Еще лучше - класс
class Address {
    private string $street;
    private string $city;
    private string $state; // понятно, что state адреса
    private string $zipCode;
}
```

---

## 🔧 Functions (Функции)

### 1. Small! (Маленькие)

**Функции должны быть МАЛЕНЬКИМИ.**

```php
// ❌ Плохо - 100+ строк
public function processOrder($orderId) {
    $order = Order::find($orderId);
    
    // Validate
    if (!$order) throw new Exception();
    if ($order->status === 'cancelled') throw new Exception();
    if ($order->total < 0) throw new Exception();
    
    // Calculate discount
    $discount = 0;
    if ($order->user->isVip()) {
        $discount = $order->total * 0.1;
    }
    
    // Process payment
    $stripe = new \Stripe\StripeClient(config('stripe.key'));
    $charge = $stripe->charges->create([...]);
    
    // Update inventory
    foreach ($order->items as $item) {
        $product = Product::find($item->product_id);
        $product->stock -= $item->quantity;
        $product->save();
    }
    
    // Send emails
    Mail::to($order->user->email)->send(new OrderConfirmation($order));
    Mail::to('admin@example.com')->send(new AdminNotification($order));
    
    // ... еще 50 строк
}

// ✅ Хорошо - разбито на маленькие функции
public function processOrder(int $orderId): void
{
    $order = $this->validateOrder($orderId);
    $discount = $this->calculateDiscount($order);
    $this->processPayment($order, $discount);
    $this->updateInventory($order);
    $this->sendNotifications($order);
}

private function validateOrder(int $orderId): Order
{
    $order = Order::findOrFail($orderId);
    
    if ($order->status === 'cancelled') {
        throw new OrderCancelledException();
    }
    
    return $order;
}

private function calculateDiscount(Order $order): float
{
    return $order->user->isVip() ? $order->total * 0.1 : 0;
}
```

**Правило:** Функция должна помещаться на один экран (20-30 строк max).

### 2. Do One Thing

**Функция должна делать ОДНО.**

```php
// ❌ Плохо - делает 3 вещи
public function saveAndEmailUser(array $data)
{
    // 1. Валидация
    if (!isset($data['email'])) throw new Exception();
    
    // 2. Сохранение
    $user = User::create($data);
    
    // 3. Отправка email
    Mail::to($user->email)->send(new WelcomeEmail($user));
    
    return $user;
}

// ✅ Хорошо - каждая функция делает одно
public function createUser(array $data): User
{
    return User::create($data);
}

public function sendWelcomeEmail(User $user): void
{
    Mail::to($user->email)->send(new WelcomeEmail($user));
}

// Использование
$user = $this->createUser($data);
$this->sendWelcomeEmail($user);
```

### 3. One Level of Abstraction

**Один уровень абстракции.**

```php
// ❌ Плохо - смешаны высокоуровневые и низкоуровневые операции
public function renderPage()
{
    $html = "<html><body>";
    
    $users = User::all(); // высокоуровневая
    
    foreach ($users as $user) { // низкоуровневая
        $html .= "<div>" . htmlspecialchars($user->name) . "</div>";
    }
    
    $html .= "</body></html>";
    
    return $html;
}

// ✅ Хорошо - один уровень абстракции
public function renderPage(): string
{
    $users = $this->getUsers();
    return $this->renderUserList($users);
}

private function getUsers(): Collection
{
    return User::all();
}

private function renderUserList(Collection $users): string
{
    return view('users.list', ['users' => $users])->render();
}
```

### 4. Function Arguments

**Идеально: 0 аргументов. Хорошо: 1-2. Терпимо: 3. >3 - плохо.**

```php
// ❌ Плохо - 5 аргументов
public function createUser(
    string $name,
    string $email,
    string $password,
    string $phone,
    string $address
) {
    // ...
}

// ✅ Хорошо - объект
public function createUser(UserData $userData)
{
    // ...
}

class UserData
{
    public function __construct(
        public string $name,
        public string $email,
        public string $password,
        public string $phone,
        public string $address
    ) {}
}

// Использование
$userData = new UserData(
    name: 'John',
    email: 'john@example.com',
    password: 'secret',
    phone: '123456',
    address: '123 Main St'
);

$this->createUser($userData);
```

### 5. No Flag Arguments

**НЕТ boolean флагам.**

```php
// ❌ Плохо - флаг означает "функция делает 2 вещи"
public function render(bool $isAdmin)
{
    if ($isAdmin) {
        return $this->renderAdminDashboard();
    }
    
    return $this->renderUserDashboard();
}

// ✅ Хорошо - 2 отдельные функции
public function renderAdminDashboard(): string
{
    // ...
}

public function renderUserDashboard(): string
{
    // ...
}
```

### 6. No Side Effects

**Функция НЕ должна иметь побочных эффектов.**

```php
// ❌ Плохо - checkPassword() МЕНЯЕТ состояние
public function checkPassword(string $username, string $password): bool
{
    $user = User::where('username', $username)->first();
    
    if ($user && Hash::check($password, $user->password)) {
        Session::initialize(); // ← ПОБОЧНЫЙ ЭФФЕКТ!
        return true;
    }
    
    return false;
}

// ✅ Хорошо - разделены проверка и побочные эффекты
public function checkPassword(string $username, string $password): bool
{
    $user = User::where('username', $username)->first();
    return $user && Hash::check($password, $user->password);
}

public function login(string $username, string $password): bool
{
    if ($this->checkPassword($username, $password)) {
        Session::initialize();
        return true;
    }
    
    return false;
}
```

### 7. Command Query Separation

**Функция должна либо ЧТО-ТО ДЕЛАТЬ, либо ЧТО-ТО ВОЗВРАЩАТЬ, но не оба.**

```php
// ❌ Плохо - set() и возвращает bool (Command + Query)
public function set(string $attribute, string $value): bool
{
    if ($this->attributeExists($attribute)) {
        $this->attributes[$attribute] = $value;
        return true; // успех
    }
    
    return false; // атрибут не найден
}

// Использование непонятно
if ($user->set('name', 'John')) { // set name to John? или проверка?
    // ...
}

// ✅ Хорошо - разделены
public function setAttribute(string $attribute, string $value): void
{
    if (!$this->attributeExists($attribute)) {
        throw new AttributeNotFoundException();
    }
    
    $this->attributes[$attribute] = $value;
}

public function hasAttribute(string $attribute): bool
{
    return $this->attributeExists($attribute);
}

// Использование понятно
if ($user->hasAttribute('name')) {
    $user->setAttribute('name', 'John');
}
```

### 8. Extract Try/Catch

**Error handling = одно дело.**

```php
// ❌ Плохо - смешаны бизнес-логика и error handling
public function deleteUser(int $id)
{
    try {
        $user = User::findOrFail($id);
        
        if ($user->hasOrders()) {
            throw new Exception('Cannot delete user with orders');
        }
        
        $user->delete();
        
        Log::info("User {$id} deleted");
    } catch (\Exception $e) {
        Log::error($e->getMessage());
        throw $e;
    }
}

// ✅ Хорошо - разделены
public function deleteUser(int $id): void
{
    try {
        $this->performDelete($id);
    } catch (\Exception $e) {
        $this->handleDeleteError($e, $id);
        throw $e;
    }
}

private function performDelete(int $id): void
{
    $user = User::findOrFail($id);
    
    if ($user->hasOrders()) {
        throw new UserHasOrdersException();
    }
    
    $user->delete();
    Log::info("User {$id} deleted");
}

private function handleDeleteError(\Exception $e, int $id): void
{
    Log::error("Failed to delete user {$id}: " . $e->getMessage());
}
```

---

## 💬 Comments (Комментарии)

### Правило: Код должен быть self-explanatory

**Хороший код не нуждается в комментариях.**

```php
// ❌ Плохо - комментарий объясняет плохой код
// Check if user is eligible for discount
if ($u->a > 65 || $u->m === true || $u->o > 100000) {
    // ...
}

// ✅ Хорошо - код сам объясняет себя
if ($user->isEligibleForDiscount()) {
    // ...
}

private function isEligibleForDiscount(): bool
{
    return $this->age > 65
        || $this->isMember
        || $this->totalOrders > 100000;
}
```

### Хорошие комментарии

**1. Legal Comments:**
```php
// Copyright (C) 2024 Company Inc. All rights reserved.
```

**2. Informative Comments:**
```php
// Returns timestamp in milliseconds since epoch
public function getTimestamp(): int
{
    return (int)(microtime(true) * 1000);
}
```

**3. Explanation of Intent:**
```php
// We use MD5 here for backward compatibility with legacy system.
// TODO: Migrate to bcrypt after legacy system is decommissioned.
$hash = md5($password);
```

**4. Warning of Consequences:**
```php
// WARNING: This test takes 10 minutes to run
public function testFullSystemIntegration() {}

// Don't run this in production - it truncates all tables
public function resetDatabase() {}
```

**5. TODO Comments:**
```php
// TODO: Refactor to use Repository pattern
$users = User::where('active', true)->get();

// FIXME: This breaks when user has no orders
$lastOrder = $user->orders->last();
```

### Плохие комментарии

**1. Mumbling (Бормотание):**
```php
// ❌ Плохо
// This is important
$x = $y + $z;
```

**2. Redundant (Избыточные):**
```php
// ❌ Плохо - комментарий дублирует код
// Get user by ID
$user = User::find($id);

// Increment counter by 1
$counter++;
```

**3. Misleading (Вводящие в заблуждение):**
```php
// ❌ Плохо - комментарий врет
// Returns user or null
public function getUser(): User // ← не может вернуть null!
{
    return User::firstOrFail(); // ← throws exception
}
```

**4. Commented-Out Code:**
```php
// ❌ Плохо - закомментированный код
public function calculate()
{
    $result = $this->step1();
    // $result = $this->step2($result);
    // $result = $this->step3($result);
    return $result;
}

// ✅ Хорошо - удали мертвый код (есть Git)
public function calculate()
{
    return $this->step1();
}
```

---

## 📐 Formatting (Форматирование)

### 1. Vertical Formatting

**Файл должен быть не слишком большим (200-500 строк).**

**Пустые строки разделяют концепции:**
```php
// ✅ Хорошо
namespace App\Services;

use App\Models\User;
use Illuminate\Support\Facades\Hash;

class UserService
{
    public function createUser(array $data): User
    {
        $this->validateData($data);
        
        $user = User::create([
            'name' => $data['name'],
            'email' => $data['email'],
            'password' => Hash::make($data['password']),
        ]);
        
        $this->sendWelcomeEmail($user);
        
        return $user;
    }
    
    private function validateData(array $data): void
    {
        // ...
    }
}
```

**Связанный код должен быть близко:**
```php
// ✅ Хорошо - вызываемая функция сразу после вызывающей
public function processOrder(Order $order): void
{
    $this->validateOrder($order);
    // ... остальная логика
}

private function validateOrder(Order $order): void
{
    // ...
}
```

### 2. Horizontal Formatting

**Строки не должны быть слишком длинными (80-120 символов).**

```php
// ❌ Плохо
$this->userRepository->updateUserProfileWithNewDataIncludingEmailPhoneAndAddress($userId, $email, $phone, $address, $city, $state, $zipCode, $country);

// ✅ Хорошо
$this->userRepository->updateProfile(
    userId: $userId,
    email: $email,
    phone: $phone,
    address: new Address($address, $city, $state, $zipCode, $country)
);
```

### 3. Indentation

**Используй 4 пробела (PSR-12).**

```php
// ✅ Хорошо
class Example
{
    public function method()
    {
        if ($condition) {
            // 4 spaces
            foreach ($items as $item) {
                // 8 spaces
                echo $item;
            }
        }
    }
}
```

---

## 🏗️ Objects vs Data Structures

### Objects - hide data, expose behavior

```php
// ✅ Object - инкапсуляция
class BankAccount
{
    private float $balance;
    
    public function deposit(float $amount): void
    {
        if ($amount <= 0) {
            throw new InvalidArgumentException();
        }
        
        $this->balance += $amount;
    }
    
    public function withdraw(float $amount): void
    {
        if ($amount > $this->balance) {
            throw new InsufficientFundsException();
        }
        
        $this->balance -= $amount;
    }
    
    public function getBalance(): float
    {
        return $this->balance;
    }
}
```

### Data Structures - expose data, no behavior

```php
// ✅ Data Structure - просто данные
class UserData
{
    public string $name;
    public string $email;
    public int $age;
}
```

### Law of Demeter - Don't talk to strangers

**Метод должен вызывать методы только:**
- своего объекта
- объектов, созданных методом
- объектов, переданных как аргументы
- объектов, хранящихся в полях класса

```php
// ❌ Плохо - нарушение Law of Demeter
$street = $user->getAddress()->getStreet();
$city = $user->getAddress()->getCity();

// ✅ Хорошо - Tell, Don't Ask
$address = $user->getFullAddress(); // "123 Main St, New York"
```

---

## 🎯 Error Handling

### 1. Use Exceptions, Not Return Codes

```php
// ❌ Плохо - error codes
public function deleteUser(int $id): int
{
    $user = User::find($id);
    
    if (!$user) return -1; // not found
    if ($user->hasOrders()) return -2; // has orders
    
    $user->delete();
    return 0; // success
}

// Использование
$result = $this->deleteUser($id);
if ($result === -1) {
    // handle not found
} elseif ($result === -2) {
    // handle has orders
}

// ✅ Хорошо - exceptions
public function deleteUser(int $id): void
{
    $user = User::findOrFail($id); // throws ModelNotFoundException
    
    if ($user->hasOrders()) {
        throw new UserHasOrdersException();
    }
    
    $user->delete();
}

// Использование
try {
    $this->deleteUser($id);
} catch (ModelNotFoundException $e) {
    // handle not found
} catch (UserHasOrdersException $e) {
    // handle has orders
}
```

### 2. Provide Context with Exceptions

```php
// ❌ Плохо
throw new Exception('Error');

// ✅ Хорошо
throw new InsufficientFundsException(
    "Insufficient funds: attempted to withdraw {$amount}, balance is {$this->balance}"
);
```

### 3. Don't Return Null

```php
// ❌ Плохо - возвращает null
public function getUsers(): ?array
{
    $users = User::all();
    
    if ($users->isEmpty()) {
        return null;
    }
    
    return $users->toArray();
}

// Использование - приходится проверять null
$users = $this->getUsers();
if ($users !== null) {
    foreach ($users as $user) {
        // ...
    }
}

// ✅ Хорошо - возвращает пустую коллекцию
public function getUsers(): Collection
{
    return User::all(); // всегда Collection, даже если пустая
}

// Использование - не нужна проверка
foreach ($this->getUsers() as $user) {
    // ...
}
```

---

## 🧹 Code Smells (Запахи кода)

### 1. Long Method

**Метод >20-30 строк - разбей на меньшие.**

### 2. Large Class

**Класс >300 строк - разбей на несколько.**

### 3. Primitive Obsession

```php
// ❌ Плохо - примитивы везде
public function createOrder(string $email, float $amount, string $currency) {}

// ✅ Хорошо - Value Objects
public function createOrder(Email $email, Money $amount) {}
```

### 4. Long Parameter List

**>3 параметров - используй объект.**

### 5. Divergent Change

**Один класс меняется по разным причинам - нарушение SRP.**

### 6. Feature Envy

```php
// ❌ Плохо - OrderService завидует данным Order
class OrderService
{
    public function calculateTotal(Order $order): float
    {
        $total = 0;
        foreach ($order->getItems() as $item) {
            $total += $item->getPrice() * $item->getQuantity();
        }
        return $total;
    }
}

// ✅ Хорошо - логика внутри Order
class Order
{
    public function calculateTotal(): float
    {
        return $this->items->sum(fn($item) => $item->getPrice() * $item->getQuantity());
    }
}
```

### 7. Data Clumps

```php
// ❌ Плохо - одни и те же параметры везде
public function method1(string $street, string $city, string $state, string $zip) {}
public function method2(string $street, string $city, string $state, string $zip) {}

// ✅ Хорошо - объект Address
public function method1(Address $address) {}
public function method2(Address $address) {}
```

---

## 🧪 Unit Tests (FIRST Principles)

**F - Fast** - быстрые (секунды, не минуты)
**I - Independent** - независимые (любой порядок)
**R - Repeatable** - повторяемые (одинаковый результат)
**S - Self-Validating** - самопроверяющиеся (pass/fail, не ручная проверка)
**T - Timely** - своевременные (TDD - пишутся ДО кода)

```php
// ✅ Хороший тест
public function test_creates_user_with_hashed_password()
{
    // Arrange
    $data = ['name' => 'John', 'email' => 'john@example.com', 'password' => 'secret'];
    
    // Act
    $user = $this->userService->createUser($data);
    
    // Assert
    $this->assertEquals('john@example.com', $user->email);
    $this->assertTrue(Hash::check('secret', $user->password));
}
```

---

## 🎓 Для собеседования: ключевые точки

1. **Meaningful Names** - intention-revealing, pronounceable, searchable
2. **Functions** - small (20-30 строк), do one thing, 0-2 аргумента
3. **Comments** - код должен быть self-explanatory, комментарии только для важного
4. **Formatting** - vertical (пустые строки), horizontal (80-120 символов)
5. **Objects** - hide data expose behavior, Law of Demeter
6. **Error Handling** - exceptions не return codes, не return null
7. **Code Smells** - long method/class, primitive obsession, feature envy
8. **SOLID** - применяй принципы
9. **DRY** - Don't Repeat Yourself
10. **KISS** - Keep It Simple, Stupid

**Главное:** Код пишется один раз, читается 10 раз. Пиши для людей, не для компилятора.

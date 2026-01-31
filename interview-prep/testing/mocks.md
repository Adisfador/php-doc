# Mocking и Test Doubles

## Test Doubles (Тестовые дублёры)

### Типы Test Doubles

#### Dummy
- Заглушка, которая не используется
- Передаётся только для удовлетворения сигнатуры
- Не имеет реальной логики
```php
// Dummy - просто заполняет параметр
$dummyLogger = new class implements LoggerInterface {
    public function log($level, $message, array $context = []) {}
};

$service = new UserService($repository, $dummyLogger);
```

#### Stub
- Предоставляет заранее определённые ответы
- Не проверяет вызовы
- Используется для изоляции от зависимостей
```php
$stub = $this->createStub(UserRepository::class);
$stub->method('find')
     ->willReturn(new User(['id' => 1, 'name' => 'John']));

// Всегда вернёт того же пользователя
$user = $stub->find(1);
$user = $stub->find(999); // тот же результат
```

#### Spy
- Записывает информацию о том, как был вызван
- Позволяет проверить вызовы после факта
- Комбинация stub + верификация
```php
$spy = $this->createMock(EventDispatcher::class);

// Используем
$service->registerUser($data);

// Проверяем что было вызвано
$spy->expects($this->once())
    ->method('dispatch')
    ->with($this->isInstanceOf(UserRegistered::class));
```

#### Mock
- Проверяет взаимодействие (как вызывался)
- Настраивается с ожиданиями
- Fail если ожидания не выполнены
```php
$mock = $this->createMock(UserRepository::class);

// Настройка ожиданий
$mock->expects($this->once())
     ->method('save')
     ->with($this->callback(function ($user) {
         return $user->email === 'test@example.com';
     }));

$service->createUser(['email' => 'test@example.com']);
// Если save не вызван или вызван с другими параметрами - тест провален
```

#### Fake
- Рабочая упрощённая реализация
- Имеет настоящую логику, но упрощённую
- Например: in-memory database, файловая система в памяти
```php
class FakeEmailService implements EmailServiceInterface
{
    public array $sent = [];
    
    public function send(Email $email): void
    {
        $this->sent[] = $email;
    }
    
    public function assertSent(string $to): void
    {
        $found = array_filter($this->sent, fn($e) => $e->to === $to);
        PHPUnit::assertNotEmpty($found, "Email to {$to} was not sent");
    }
}

// Использование
$fakeEmail = new FakeEmailService();
$service = new UserService($repository, $fakeEmail);
$service->registerUser(['email' => 'test@example.com']);

$fakeEmail->assertSent('test@example.com');
```

## Mocking с PHPUnit

### Создание Mock
```php
// Простой mock
$mock = $this->createMock(UserRepository::class);

// Mock с конструктором
$mock = $this->getMockBuilder(UserRepository::class)
             ->disableOriginalConstructor()
             ->getMock();

// Mock только определённых методов
$mock = $this->getMockBuilder(UserRepository::class)
             ->onlyMethods(['find', 'save'])
             ->getMock();
```

### Expectations - ожидаемое количество вызовов
```php
$mock->expects($this->once())        // ровно 1 раз
$mock->expects($this->never())       // ни разу
$mock->expects($this->exactly(3))    // ровно 3 раза
$mock->expects($this->atLeast(2))    // минимум 2 раза
$mock->expects($this->atLeastOnce()) // минимум 1 раз
$mock->expects($this->atMost(5))     // максимум 5 раз
$mock->expects($this->any())         // любое количество
```

### Проверка параметров
```php
// Точное соответствие
->with($this->equalTo(123))
->with(123) // короткая форма

// Тип
->with($this->isInstanceOf(User::class))
->with($this->isType('string'))

// Callback
->with($this->callback(function ($user) {
    return $user->age >= 18;
}))

// Любое значение
->with($this->anything())

// Множественные параметры
->with(
    $this->equalTo(123),
    $this->isInstanceOf(User::class)
)

// Constraints
->with($this->greaterThan(10))
->with($this->stringContains('test'))
->with($this->arrayHasKey('email'))
```

### Возвращаемые значения
```php
// Статическое значение
->willReturn($value)

// Разные значения при разных вызовах
->willReturnOnConsecutiveCalls($value1, $value2, $value3)

// Возврат себя (fluent interface)
->willReturnSelf()

// Возврат аргумента
->willReturnArgument(0)

// Map аргументов к результатам
->willReturnMap([
    [1, 'user1'],
    [2, 'user2'],
])

// Callback
->willReturnCallback(function ($arg) {
    return $arg * 2;
})

// Exception
->willThrowException(new RuntimeException('Error'))
```

### Проверка порядка вызовов
```php
$mock->expects($this->at(0))->method('method1');
$mock->expects($this->at(1))->method('method2');
$mock->expects($this->at(2))->method('method1');
```

## Mocking в Laravel

### Facade Mocking
```php
use Illuminate\Support\Facades\Mail;
use App\Mail\WelcomeEmail;

Mail::fake();

// Код который отправляет email
$user->sendWelcomeEmail();

// Проверки
Mail::assertSent(WelcomeEmail::class, function ($mail) use ($user) {
    return $mail->user->id === $user->id;
});

Mail::assertSent(WelcomeEmail::class, 1); // отправлен 1 раз
Mail::assertNotSent(GoodbyeEmail::class);
Mail::assertNothingSent();
Mail::assertQueued(WelcomeEmail::class);
```

### Event Mocking
```php
Event::fake([UserRegistered::class]);

// Или все события
Event::fake();

// Код
$user = User::create($data);

// Проверки
Event::assertDispatched(UserRegistered::class, function ($event) use ($user) {
    return $event->user->id === $user->id;
});

Event::assertNotDispatched(UserDeleted::class);
Event::assertListening(UserRegistered::class, SendWelcomeEmail::class);
```

### Queue Mocking
```php
Queue::fake();

// Код
ProcessPodcast::dispatch($podcast);

// Проверки
Queue::assertPushed(ProcessPodcast::class);
Queue::assertPushed(ProcessPodcast::class, function ($job) use ($podcast) {
    return $job->podcast->id === $podcast->id;
});

Queue::assertPushedOn('processing', ProcessPodcast::class);
Queue::assertNotPushed(AnotherJob::class);
```

### Storage Mocking
```php
Storage::fake('public');

// Код
$file = UploadedFile::fake()->image('photo.jpg');
$path = $file->store('photos', 'public');

// Проверки
Storage::disk('public')->assertExists($path);
Storage::disk('public')->assertMissing('missing.jpg');
```

### HTTP Client Mocking
```php
Http::fake([
    'api.example.com/*' => Http::response(['data' => 'value'], 200),
    'github.com/*' => Http::response(['error' => 'Not found'], 404),
    '*' => Http::response(['default' => 'response'], 200),
]);

// Или sequence
Http::fake([
    'api.example.com/*' => Http::sequence()
        ->push(['first' => 'response'], 200)
        ->push(['second' => 'response'], 200)
        ->pushStatus(404),
]);

// Проверки
Http::assertSent(function ($request) {
    return $request->url() == 'https://api.example.com/users' &&
           $request['name'] == 'John';
});

Http::assertNotSent(function ($request) {
    return $request->url() == 'https://bad-url.com';
});

Http::assertSentCount(3);
```

## Partial Mocking

### Частичный mock (некоторые методы настоящие)
```php
$mock = $this->getMockBuilder(UserService::class)
             ->onlyMethods(['sendEmail'])
             ->getMock();

// sendEmail - замокан
$mock->method('sendEmail')->willReturn(true);

// остальные методы работают как обычно
$result = $mock->createUser($data);
```

### Spy на реальный объект
```php
$service = new UserService();
$spy = $this->createPartialMock(UserService::class, ['sendEmail']);

// Реальные методы вызываются, но можем проверить вызовы
$spy->expects($this->once())->method('sendEmail');
```

## Best Practices

### Не переиспользуй mock
```php
// Плохо - один mock для разных тестов
protected UserRepository $repository;

public function setUp(): void
{
    $this->repository = $this->createMock(UserRepository::class);
}

// Хорошо - новый mock для каждого теста
public function testSomething(): void
{
    $repository = $this->createMock(UserRepository::class);
    $repository->method('find')->willReturn(new User());
}
```

### Mock внешние зависимости, не свой код
```php
// Хорошо - mock внешнего API
$httpClient = $this->createMock(HttpClient::class);

// Плохо - mock своего репозитория (лучше использовать реальный с БД)
// Исключение: медленные или сложные операции
```

### Используй Fake где возможно
```php
// Вместо mock
$mock = $this->createMock(EmailService::class);
$mock->method('send')->willReturn(true);

// Лучше Fake
Mail::fake();
```

### Проверяй поведение, не реализацию
```php
// Плохо - проверяем внутренние детали
$mock->expects($this->once())
     ->method('query')
     ->with('SELECT * FROM users WHERE id = ?', [1]);

// Хорошо - проверяем результат
$user = $service->findUser(1);
$this->assertEquals('John', $user->name);
```

### Минимум expectations
```php
// Плохо - слишком детально
$mock->expects($this->once())
     ->method('find')
     ->with($this->equalTo(1))
     ->willReturn($user);
$mock->expects($this->once())
     ->method('save')
     ->with($this->equalTo($user));

// Хорошо - только важное
$mock->method('find')->willReturn($user);
// Проверяем только результат действия
$this->assertTrue($service->updateUser(1, $data));
```

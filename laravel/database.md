# Laravel Database - Query Builder, Migrations, Seeds

Laravel предоставляет мощные инструменты для работы с базами данных.

---

## Query Builder

### Базовые запросы

```php
<?php

use Illuminate\Support\Facades\DB;

// SELECT * FROM users
$users = DB::table('users')->get();

// SELECT * FROM users WHERE age > 18
$users = DB::table('users')
    ->where('age', '>', 18)
    ->get();

// SELECT name, email FROM users
$users = DB::table('users')
    ->select('name', 'email')
    ->get();

// SELECT DISTINCT status FROM orders
$statuses = DB::table('orders')
    ->distinct()
    ->pluck('status');

// SELECT * FROM users WHERE id = 1
$user = DB::table('users')->where('id', 1)->first();

// SELECT * FROM users LIMIT 1
$user = DB::table('users')->first();

// Получить одно значение
$email = DB::table('users')->where('id', 1)->value('email');

// Найти по ID
$user = DB::table('users')->find(1);
```

### WHERE условия

```php
// Простые условия
DB::table('users')->where('status', 'active')->get();
DB::table('users')->where('votes', '>', 100)->get();
DB::table('users')->where('name', 'like', 'John%')->get();

// Множественные условия (AND)
DB::table('users')
    ->where('status', 'active')
    ->where('votes', '>', 100)
    ->get();

// OR условия
DB::table('users')
    ->where('votes', '>', 100)
    ->orWhere('name', 'John')
    ->get();

// WHERE IN
DB::table('users')
    ->whereIn('id', [1, 2, 3])
    ->get();

DB::table('users')
    ->whereNotIn('status', ['banned', 'suspended'])
    ->get();

// WHERE BETWEEN
DB::table('users')
    ->whereBetween('votes', [1, 100])
    ->get();

DB::table('users')
    ->whereNotBetween('votes', [1, 100])
    ->get();

// WHERE NULL
DB::table('users')
    ->whereNull('deleted_at')
    ->get();

DB::table('users')
    ->whereNotNull('email_verified_at')
    ->get();

// WHERE DATE
DB::table('users')
    ->whereDate('created_at', '2024-01-01')
    ->get();

DB::table('users')
    ->whereMonth('created_at', 1)
    ->get();

DB::table('users')
    ->whereYear('created_at', 2024)
    ->get();

DB::table('users')
    ->whereTime('created_at', '>=', '09:00')
    ->get();

// WHERE с замыканием (группировка)
DB::table('users')
    ->where('status', 'active')
    ->where(function ($query) {
        $query->where('votes', '>', 100)
              ->orWhere('premium', true);
    })
    ->get();
// WHERE status = 'active' AND (votes > 100 OR premium = true)

// JSON WHERE (PostgreSQL, MySQL 5.7+)
DB::table('users')
    ->where('preferences->notifications->email', true)
    ->get();

DB::table('users')
    ->whereJsonContains('options->languages', 'en')
    ->get();
```

### ORDER BY, LIMIT, OFFSET

```php
// ORDER BY
DB::table('users')
    ->orderBy('name', 'asc')
    ->get();

DB::table('users')
    ->orderBy('votes', 'desc')
    ->orderBy('name', 'asc')
    ->get();

// Shortcuts
DB::table('users')->latest()->get(); // ORDER BY created_at DESC
DB::table('users')->oldest()->get(); // ORDER BY created_at ASC

DB::table('users')
    ->latest('updated_at')
    ->get();

// Random order
DB::table('users')
    ->inRandomOrder()
    ->first();

// LIMIT & OFFSET
DB::table('users')
    ->skip(10)
    ->take(5)
    ->get();

// Пагинация
$users = DB::table('users')->paginate(15);
$users = DB::table('users')->simplePaginate(15);
```

### GROUP BY & HAVING

```php
// GROUP BY
$orders = DB::table('orders')
    ->select('user_id', DB::raw('count(*) as total'))
    ->groupBy('user_id')
    ->get();

// HAVING
$users = DB::table('orders')
    ->select('user_id', DB::raw('SUM(price) as total_spent'))
    ->groupBy('user_id')
    ->having('total_spent', '>', 1000)
    ->get();

// HAVING с RAW
DB::table('orders')
    ->groupBy('user_id')
    ->havingRaw('SUM(price) > ?', [1000])
    ->get();
```

### JOINS

```php
// INNER JOIN
$users = DB::table('users')
    ->join('orders', 'users.id', '=', 'orders.user_id')
    ->select('users.*', 'orders.price')
    ->get();

// LEFT JOIN
$users = DB::table('users')
    ->leftJoin('orders', 'users.id', '=', 'orders.user_id')
    ->get();

// RIGHT JOIN
$users = DB::table('users')
    ->rightJoin('orders', 'users.id', '=', 'orders.user_id')
    ->get();

// CROSS JOIN
$combinations = DB::table('sizes')
    ->crossJoin('colors')
    ->get();

// JOIN с дополнительными условиями
DB::table('users')
    ->join('orders', function ($join) {
        $join->on('users.id', '=', 'orders.user_id')
             ->where('orders.status', '=', 'completed');
    })
    ->get();

// Subquery JOIN
$latestOrders = DB::table('orders')
    ->select('user_id', DB::raw('MAX(created_at) as last_order_date'))
    ->groupBy('user_id');

$users = DB::table('users')
    ->joinSub($latestOrders, 'latest_orders', function ($join) {
        $join->on('users.id', '=', 'latest_orders.user_id');
    })
    ->get();
```

### Агрегатные функции

```php
// COUNT
$count = DB::table('users')->count();
$activeCount = DB::table('users')->where('status', 'active')->count();

// MAX, MIN, AVG, SUM
$maxPrice = DB::table('orders')->max('price');
$minPrice = DB::table('orders')->min('price');
$avgPrice = DB::table('orders')->avg('price');
$totalPrice = DB::table('orders')->sum('price');

// Проверка существования
$exists = DB::table('users')->where('email', 'test@example.com')->exists();
$doesntExist = DB::table('users')->where('email', 'test@example.com')->doesntExist();
```

### INSERT

```php
// Одна запись
DB::table('users')->insert([
    'name' => 'John Doe',
    'email' => 'john@example.com',
    'created_at' => now(),
]);

// Множественные записи
DB::table('users')->insert([
    ['name' => 'John', 'email' => 'john@example.com'],
    ['name' => 'Jane', 'email' => 'jane@example.com'],
]);

// INSERT и получить ID
$id = DB::table('users')->insertGetId([
    'name' => 'John Doe',
    'email' => 'john@example.com',
]);

// INSERT OR IGNORE (если уникальный ключ конфликтует)
DB::table('users')->insertOrIgnore([
    ['email' => 'john@example.com', 'name' => 'John'],
    ['email' => 'jane@example.com', 'name' => 'Jane'],
]);
```

### UPDATE

```php
// UPDATE
DB::table('users')
    ->where('id', 1)
    ->update(['status' => 'active']);

// UPDATE с инкрементом
DB::table('users')->increment('votes'); // votes + 1
DB::table('users')->increment('votes', 5); // votes + 5
DB::table('users')->decrement('votes'); // votes - 1
DB::table('users')->decrement('votes', 5); // votes - 5

// Increment с дополнительными полями
DB::table('users')
    ->where('id', 1)
    ->increment('votes', 1, ['status' => 'active']);

// UPDATE OR INSERT (upsert)
DB::table('users')->updateOrInsert(
    ['email' => 'john@example.com'], // Условие поиска
    ['name' => 'John Doe', 'votes' => 1] // Данные для update/insert
);

// Bulk UPSERT (Laravel 8+)
DB::table('users')->upsert(
    [
        ['email' => 'john@example.com', 'name' => 'John', 'votes' => 1],
        ['email' => 'jane@example.com', 'name' => 'Jane', 'votes' => 2],
    ],
    ['email'], // Уникальные ключи
    ['name', 'votes'] // Какие поля обновлять при конфликте
);
```

### DELETE

```php
// DELETE
DB::table('users')->where('id', 1)->delete();

// DELETE всех записей
DB::table('users')->delete();

// TRUNCATE (быстрее чем DELETE для всей таблицы)
DB::table('users')->truncate();
```

### Transactions (Транзакции)

```php
use Illuminate\Support\Facades\DB;

// Автоматическая транзакция
DB::transaction(function () {
    DB::table('users')->update(['votes' => 1]);
    DB::table('posts')->delete();
});

// С количеством попыток (deadlock retry)
DB::transaction(function () {
    // ...
}, 5); // Попытается 5 раз при deadlock

// Ручное управление
DB::beginTransaction();

try {
    DB::table('users')->update(['votes' => 1]);
    DB::table('posts')->delete();
    
    DB::commit();
} catch (\Exception $e) {
    DB::rollBack();
    throw $e;
}

// Проверка в транзакции
if (DB::transactionLevel() > 0) {
    // Внутри транзакции
}
```

### Raw Expressions

```php
// Raw SELECT
$users = DB::table('users')
    ->select(DB::raw('count(*) as user_count, status'))
    ->groupBy('status')
    ->get();

// Raw WHERE
DB::table('users')
    ->whereRaw('price > IF(state = "TX", ?, 100)', [200])
    ->get();

// Raw ORDER BY
DB::table('users')
    ->orderByRaw('updated_at - created_at DESC')
    ->get();

// Raw HAVING
DB::table('orders')
    ->select('department', DB::raw('SUM(price) as total_sales'))
    ->groupBy('department')
    ->havingRaw('SUM(price) > ?', [2500])
    ->get();
```

### Pessimistic Locking

```php
// FOR UPDATE (блокировка для обновления)
DB::table('users')
    ->where('votes', '>', 100)
    ->lockForUpdate()
    ->get();

// FOR SHARE (блокировка для чтения)
DB::table('users')
    ->where('votes', '>', 100)
    ->sharedLock()
    ->get();

// Пример использования
DB::transaction(function () {
    $user = DB::table('users')
        ->where('id', 1)
        ->lockForUpdate()
        ->first();
    
    // Никто другой не может изменить этого пользователя
    DB::table('users')
        ->where('id', 1)
        ->update(['balance' => $user->balance - 100]);
});
```

---

## Migrations (Миграции)

### Создание миграции

```bash
# Создать миграцию для новой таблицы
php artisan make:migration create_users_table

# Создать миграцию для изменения таблицы
php artisan make:migration add_status_to_users_table --table=users

# Создать миграцию с созданием модели
php artisan make:model Post -m
```

### Структура миграции

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('users', function (Blueprint $table) {
            $table->id(); // bigint unsigned auto_increment primary key
            $table->string('name');
            $table->string('email')->unique();
            $table->timestamp('email_verified_at')->nullable();
            $table->string('password');
            $table->rememberToken();
            $table->timestamps(); // created_at, updated_at
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('users');
    }
};
```

### Типы колонок

```php
Schema::create('posts', function (Blueprint $table) {
    // Числовые
    $table->id(); // BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY
    $table->bigInteger('votes');
    $table->integer('votes');
    $table->smallInteger('votes');
    $table->tinyInteger('votes');
    $table->mediumInteger('votes');
    $table->unsignedBigInteger('user_id');
    $table->decimal('amount', 8, 2); // DECIMAL(8,2)
    $table->double('amount', 8, 2);
    $table->float('amount', 8, 2);
    
    // Строки
    $table->string('name'); // VARCHAR(255)
    $table->string('name', 100); // VARCHAR(100)
    $table->text('description'); // TEXT
    $table->mediumText('description'); // MEDIUMTEXT
    $table->longText('description'); // LONGTEXT
    $table->char('code', 4); // CHAR(4)
    
    // Даты и время
    $table->date('birth_date'); // DATE
    $table->datetime('created_at'); // DATETIME
    $table->time('sunrise'); // TIME
    $table->timestamp('added_at'); // TIMESTAMP
    $table->timestamps(); // created_at, updated_at TIMESTAMP
    $table->timestampsTz(); // created_at, updated_at TIMESTAMP WITH TIME ZONE
    $table->softDeletes(); // deleted_at TIMESTAMP NULL
    $table->year('birth_year'); // YEAR
    
    // Булевы
    $table->boolean('confirmed'); // BOOLEAN (TINYINT(1))
    
    // JSON
    $table->json('options'); // JSON
    $table->jsonb('options'); // JSONB (PostgreSQL)
    
    // Бинарные
    $table->binary('data'); // BLOB
    
    // Специальные
    $table->uuid('id'); // UUID (CHAR(36))
    $table->ulid('id'); // ULID (CHAR(26))
    $table->ipAddress('visitor'); // IP адрес
    $table->macAddress('device'); // MAC адрес
    $table->enum('status', ['pending', 'active', 'inactive']);
    $table->set('flavors', ['strawberry', 'vanilla']);
    
    // Геометрия (MySQL/PostgreSQL)
    $table->geometry('positions');
    $table->point('position');
    $table->lineString('path');
    $table->polygon('area');
});
```

### Модификаторы колонок

```php
$table->string('email')->nullable(); // NULL
$table->string('name')->default('Guest'); // DEFAULT 'Guest'
$table->integer('votes')->unsigned(); // UNSIGNED
$table->decimal('amount', 8, 2)->unsigned()->default(0);
$table->string('name')->comment('User full name'); // COMMENT
$table->timestamp('created_at')->useCurrent(); // DEFAULT CURRENT_TIMESTAMP
$table->timestamp('updated_at')->useCurrentOnUpdate(); // ON UPDATE CURRENT_TIMESTAMP
$table->bigInteger('user_id')->index(); // INDEX
$table->string('email')->unique(); // UNIQUE
$table->text('bio')->fulltext(); // FULLTEXT INDEX (MySQL)
$table->foreignId('user_id')->constrained(); // FOREIGN KEY
$table->string('name')->after('email'); // Позиция колонки
$table->string('name')->first(); // Первая колонка
$table->string('name')->invisible(); // Невидимая колонка (MySQL 8.0.23+)
$table->integer('order')->storedAs('price * quantity'); // Generated column
$table->integer('total')->virtualAs('price + tax'); // Virtual column
```

### Индексы

```php
Schema::table('users', function (Blueprint $table) {
    // Обычный индекс
    $table->index('email');
    $table->index(['user_id', 'status']); // Составной индекс
    
    // Уникальный индекс
    $table->unique('email');
    $table->unique(['email', 'account_id'], 'unique_email_account');
    
    // Primary key
    $table->primary('id');
    $table->primary(['id', 'parent_id']);
    
    // Fulltext (MySQL)
    $table->fulltext('bio');
    $table->fulltext(['title', 'description']);
    
    // Spatial index (MySQL)
    $table->spatialIndex('location');
    
    // Удаление индексов
    $table->dropIndex(['email']); // DROP INDEX users_email_index
    $table->dropUnique(['email']); // DROP INDEX users_email_unique
    $table->dropPrimary(['id']);
    $table->dropFulltext(['bio']);
    $table->dropSpatialIndex(['location']);
});
```

### Foreign Keys

```php
Schema::create('posts', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained();
    // Создаст колонку user_id и FK к таблице users(id)
    
    // С кастомными опциями
    $table->foreignId('user_id')
        ->constrained()
        ->onUpdate('cascade')
        ->onDelete('cascade');
    
    // Явное указание таблицы и колонки
    $table->foreignId('author_id')
        ->constrained('users', 'id')
        ->onDelete('set null');
    
    // Старый синтаксис
    $table->unsignedBigInteger('user_id');
    $table->foreign('user_id')
        ->references('id')
        ->on('users')
        ->onDelete('cascade');
    
    // Удаление FK
    $table->dropForeign(['user_id']); // posts_user_id_foreign
    $table->dropForeign('posts_user_id_foreign'); // Явное имя
});

// onDelete/onUpdate опции:
// - cascade: каскадное удаление/обновление
// - set null: установить NULL
// - restrict: запретить удаление/обновление
// - no action: ничего не делать
```

### Изменение таблицы

```php
Schema::table('users', function (Blueprint $table) {
    // Добавить колонку
    $table->string('phone')->nullable();
    
    // Изменить колонку (требует doctrine/dbal)
    $table->string('name', 100)->change();
    $table->integer('votes')->unsigned()->default(0)->change();
    
    // Переименовать колонку
    $table->renameColumn('from', 'to');
    
    // Удалить колонку
    $table->dropColumn('votes');
    $table->dropColumn(['votes', 'avatar']);
    
    // Переименовать таблицу
    Schema::rename('from', 'to');
    
    // Проверка существования
    if (Schema::hasTable('users')) {
        // Таблица существует
    }
    
    if (Schema::hasColumn('users', 'email')) {
        // Колонка существует
    }
});
```

### Запуск миграций

```bash
# Выполнить все миграции
php artisan migrate

# Откатить последний batch
php artisan migrate:rollback

# Откатить последние 5 миграций
php artisan migrate:rollback --step=5

# Откатить ВСЕ миграции
php artisan migrate:reset

# Откатить все и выполнить заново
php artisan migrate:refresh

# Refresh + запустить сиды
php artisan migrate:refresh --seed

# Drop все таблицы и выполнить миграции заново
php artisan migrate:fresh

# Fresh + seeds
php artisan migrate:fresh --seed

# Статус миграций
php artisan migrate:status

# Симуляция (не выполнять реально)
php artisan migrate --pretend
```

---

## Seeders (Сиды)

### Создание сидера

```bash
php artisan make:seeder UserSeeder
```

### Базовая структура

```php
<?php

namespace Database\Seeders;

use Illuminate\Database\Seeder;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Hash;

class UserSeeder extends Seeder
{
    public function run(): void
    {
        // Query Builder
        DB::table('users')->insert([
            'name' => 'Admin',
            'email' => 'admin@example.com',
            'password' => Hash::make('password'),
            'created_at' => now(),
        ]);
        
        // Eloquent
        User::create([
            'name' => 'Admin',
            'email' => 'admin@example.com',
            'password' => Hash::make('password'),
        ]);
        
        // Множественные записи
        User::insert([
            [
                'name' => 'User 1',
                'email' => 'user1@example.com',
                'password' => Hash::make('password'),
                'created_at' => now(),
            ],
            [
                'name' => 'User 2',
                'email' => 'user2@example.com',
                'password' => Hash::make('password'),
                'created_at' => now(),
            ],
        ]);
    }
}
```

### Вызов других сидеров

```php
class DatabaseSeeder extends Seeder
{
    public function run(): void
    {
        $this->call([
            UserSeeder::class,
            PostSeeder::class,
            CommentSeeder::class,
        ]);
    }
}
```

### Model Factories

```bash
php artisan make:factory UserFactory --model=User
```

**database/factories/UserFactory.php:**

```php
<?php

namespace Database\Factories;

use Illuminate\Database\Eloquent\Factories\Factory;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Str;

class UserFactory extends Factory
{
    protected static ?string $password;

    public function definition(): array
    {
        return [
            'name' => fake()->name(),
            'email' => fake()->unique()->safeEmail(),
            'email_verified_at' => now(),
            'password' => static::$password ??= Hash::make('password'),
            'remember_token' => Str::random(10),
        ];
    }

    // State - модификация фабрики
    public function unverified(): static
    {
        return $this->state(fn (array $attributes) => [
            'email_verified_at' => null,
        ]);
    }

    public function admin(): static
    {
        return $this->state(fn (array $attributes) => [
            'role' => 'admin',
        ]);
    }
}
```

**Использование фабрик:**

```php
// В сидере
class UserSeeder extends Seeder
{
    public function run(): void
    {
        // Создать 10 пользователей
        User::factory()->count(10)->create();
        
        // С кастомными атрибутами
        User::factory()->create([
            'email' => 'admin@example.com',
        ]);
        
        // Со state
        User::factory()->unverified()->create();
        User::factory()->admin()->create();
        
        // С отношениями
        User::factory()
            ->has(Post::factory()->count(3))
            ->create();
        
        // Или через for()
        Post::factory()
            ->count(3)
            ->for(User::factory())
            ->create();
        
        // Many-to-many
        User::factory()
            ->hasAttached(
                Role::factory()->count(3),
                ['expires_at' => now()->addYear()]
            )
            ->create();
    }
}

// В тестах
public function test_user_can_be_created()
{
    $user = User::factory()->make(); // Только в памяти
    $user = User::factory()->create(); // Сохранить в БД
    
    $users = User::factory()->count(5)->create();
}
```

### Запуск сидеров

```bash
# Запустить DatabaseSeeder
php artisan db:seed

# Запустить конкретный сидер
php artisan db:seed --class=UserSeeder

# В production (требует --force)
php artisan db:seed --force
```

---

## Database Console Commands

```bash
# Удалить БД и создать заново
php artisan db:wipe

# Мониторинг БД соединений
php artisan db:monitor --max=100

# Показать информацию о БД
php artisan db:show
php artisan db:show --database=mysql

# Показать таблицы
php artisan db:table users
```

---

## Best Practices

### 1. Используй транзакции для критичных операций

```php
DB::transaction(function () {
    $user = User::create([...]);
    $user->profile()->create([...]);
    Order::create(['user_id' => $user->id]);
});
```

### 2. Индексы для часто используемых колонок

```php
// В миграции
$table->index('email');
$table->index('status');
$table->index(['user_id', 'created_at']); // Составной
```

### 3. Используй chunking для больших объемов

```php
// ❌ ПЛОХО: загрузит все в память
$users = DB::table('users')->get();

// ✅ ХОРОШО: по кускам
DB::table('users')->orderBy('id')->chunk(200, function ($users) {
    foreach ($users as $user) {
        // Обработка
    }
});
```

### 4. Soft Deletes где нужно

```php
// В миграции
$table->softDeletes();

// В модели
use SoftDeletes;

// Использование
$user->delete(); // Установит deleted_at
$user->forceDelete(); // Реальное удаление
User::withTrashed()->get(); // Включая удаленные
User::onlyTrashed()->get(); // Только удаленные
```

### 5. Используй Eloquent вместо Query Builder где возможно

```php
// ❌ Query Builder
$user = DB::table('users')->where('id', 1)->first();
$posts = DB::table('posts')->where('user_id', $user->id)->get();

// ✅ Eloquent (с Eager Loading)
$user = User::with('posts')->find(1);
$posts = $user->posts;
```

### 6. Называй миграции описательно

```bash
# ✅ ХОРОШО
2024_01_01_000001_create_users_table.php
2024_01_02_000001_add_status_to_users_table.php
2024_01_03_000001_create_posts_table.php

# ❌ ПЛОХО
migration1.php
update_users.php
```

### 7. Используй Foreign Keys для целостности данных

```php
$table->foreignId('user_id')
    ->constrained()
    ->onDelete('cascade'); // Автоматически удалит связанные записи
```

# Eloquent ORM - Deep Dive

Eloquent - это ActiveRecord ORM реализация в Laravel. Каждая модель представляет таблицу в БД.

---

## Базовые концепции

### Создание модели

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;

class Post extends Model
{
    use SoftDeletes;

    // Явное указание таблицы (опционально, по умолчанию snake_case множественное число)
    protected $table = 'posts';

    // Primary key (по умолчанию 'id')
    protected $primaryKey = 'post_id';

    // Автоинкремент (по умолчанию true)
    public $incrementing = true;

    // Тип primary key (по умолчанию 'int')
    protected $keyType = 'int';

    // Timestamps (created_at, updated_at)
    public $timestamps = true;

    // Формат дат (по умолчанию 'Y-m-d H:i:s')
    protected $dateFormat = 'U';

    // Какие поля можно массово заполнять
    protected $fillable = ['title', 'content', 'user_id', 'status'];

    // Какие поля НЕЛЬЗЯ массово заполнять (противоположность fillable)
    protected $guarded = ['id', 'user_id'];

    // Скрытые поля (не попадут в JSON/Array)
    protected $hidden = ['password', 'remember_token'];

    // Всегда видимые в JSON/Array
    protected $visible = ['id', 'title', 'created_at'];

    // Дополнительные поля в JSON/Array (accessors/appends)
    protected $appends = ['full_title'];

    // Автоматическое приведение типов
    protected $casts = [
        'is_published' => 'boolean',
        'published_at' => 'datetime',
        'metadata' => 'array', // JSON в массив
        'views_count' => 'integer',
        'rating' => 'decimal:2',
        'tags' => 'collection', // Laravel 10+
    ];

    // Значения по умолчанию
    protected $attributes = [
        'status' => 'draft',
        'views_count' => 0,
    ];
}
```

---

## Отношения (Relationships)

### One-to-One

```php
// User hasOne Profile
class User extends Model
{
    public function profile()
    {
        return $this->hasOne(Profile::class);
        // return $this->hasOne(Profile::class, 'user_id', 'id');
    }
}

class Profile extends Model
{
    public function user()
    {
        return $this->belongsTo(User::class);
        // return $this->belongsTo(User::class, 'user_id', 'id');
    }
}

// Использование
$user = User::find(1);
$profile = $user->profile; // Автоматический SELECT * FROM profiles WHERE user_id = 1

$profile = Profile::find(1);
$user = $profile->user; // SELECT * FROM users WHERE id = ?
```

### One-to-Many

```php
// User hasMany Posts
class User extends Model
{
    public function posts()
    {
        return $this->hasMany(Post::class);
    }
}

class Post extends Model
{
    public function user()
    {
        return $this->belongsTo(User::class);
    }
}

// Использование
$posts = User::find(1)->posts; // Collection

// Создание связанной записи
$user->posts()->create([
    'title' => 'New Post',
    'content' => 'Content',
]);

// Или через модель
$post = new Post(['title' => 'New', 'content' => 'Text']);
$user->posts()->save($post);
```

### Many-to-Many

```php
// User belongsToMany Roles (через pivot таблицу role_user)
class User extends Model
{
    public function roles()
    {
        return $this->belongsToMany(Role::class)
            ->withPivot('expires_at', 'assigned_by') // Дополнительные поля pivot
            ->withTimestamps() // created_at, updated_at в pivot
            ->using(RoleUser::class); // Кастомная Pivot модель
    }
}

class Role extends Model
{
    public function users()
    {
        return $this->belongsToMany(User::class);
    }
}

// Кастомная Pivot модель
class RoleUser extends Pivot
{
    protected $casts = [
        'expires_at' => 'datetime',
    ];

    public function assignedBy()
    {
        return $this->belongsTo(User::class, 'assigned_by');
    }
}

// Использование
$roles = $user->roles;

// Доступ к pivot данным
foreach ($user->roles as $role) {
    echo $role->pivot->expires_at;
    echo $role->pivot->assigned_by;
}

// Присоединение (attach)
$user->roles()->attach($roleId); // INSERT в role_user
$user->roles()->attach($roleId, ['expires_at' => now()->addYear()]);
$user->roles()->attach([1, 2, 3]); // Множественное

// Отсоединение (detach)
$user->roles()->detach($roleId); // DELETE FROM role_user
$user->roles()->detach([1, 2]); // Множественное
$user->roles()->detach(); // Удалить все связи

// Синхронизация (sync) - заменяет все связи
$user->roles()->sync([1, 2, 3]);
$user->roles()->syncWithoutDetaching([4, 5]); // Добавить, не удаляя старые

// Toggle (переключение)
$user->roles()->toggle([1, 2]); // Если есть - удалить, нет - добавить
```

### Has-Many-Through

```php
// Country -> Users -> Posts (Country hasMany Posts through Users)
class Country extends Model
{
    public function posts()
    {
        return $this->hasManyThrough(
            Post::class,      // Конечная модель
            User::class,      // Промежуточная модель
            'country_id',     // FK в users
            'user_id',        // FK в posts
            'id',             // Local key в countries
            'id'              // Local key в users
        );
    }
}

// Использование
$country = Country::find(1);
$posts = $country->posts; // Все посты всех пользователей страны
```

### Polymorphic Relations

```php
// Один Comment может принадлежать Post или Video
class Comment extends Model
{
    public function commentable()
    {
        return $this->morphTo(); // commentable_type, commentable_id
    }
}

class Post extends Model
{
    public function comments()
    {
        return $this->morphMany(Comment::class, 'commentable');
    }
}

class Video extends Model
{
    public function comments()
    {
        return $this->morphMany(Comment::class, 'commentable');
    }
}

// Использование
$post = Post::find(1);
$comments = $post->comments; // WHERE commentable_type = 'App\Models\Post' AND commentable_id = 1

$comment = Comment::find(1);
$commentable = $comment->commentable; // Post или Video

// Создание
$post->comments()->create(['body' => 'Great post!']);
```

### Many-to-Many Polymorphic

```php
// Post и Video могут иметь много Tags
class Post extends Model
{
    public function tags()
    {
        return $this->morphToMany(Tag::class, 'taggable');
    }
}

class Video extends Model
{
    public function tags()
    {
        return $this->morphToMany(Tag::class, 'taggable');
    }
}

class Tag extends Model
{
    public function posts()
    {
        return $this->morphedByMany(Post::class, 'taggable');
    }

    public function videos()
    {
        return $this->morphedByMany(Video::class, 'taggable');
    }
}

// Таблица taggables: taggable_id, taggable_type, tag_id
```

---

## N+1 Problem - Критическая проблема производительности!

### Проблема

```php
// ❌ ПЛОХО: N+1 запросов
$posts = Post::all(); // 1 запрос: SELECT * FROM posts

foreach ($posts as $post) {
    echo $post->user->name; // N запросов: SELECT * FROM users WHERE id = ?
}
// Итого: 1 + 100 = 101 запрос для 100 постов!
```

### Решение: Eager Loading

```php
// ✅ ХОРОШО: 2 запроса
$posts = Post::with('user')->get();
// 1: SELECT * FROM posts
// 2: SELECT * FROM users WHERE id IN (1, 2, 3, ...)

foreach ($posts as $post) {
    echo $post->user->name; // Данные уже загружены, нет запросов
}
```

### Множественные отношения

```php
// Несколько отношений
$posts = Post::with(['user', 'comments', 'tags'])->get();

// Вложенные отношения (nested)
$posts = Post::with('comments.user')->get();
// Загружает посты -> комментарии -> пользователей комментариев

// Глубокая вложенность
$countries = Country::with('users.posts.comments.user')->get();
```

### Eager Loading с условиями

```php
// Загрузить только определенные комментарии
$posts = Post::with(['comments' => function ($query) {
    $query->where('approved', true)
          ->orderBy('created_at', 'desc')
          ->limit(5);
}])->get();

// Подсчет связанных записей
$posts = Post::withCount('comments')->get();
echo $posts[0]->comments_count; // Без загрузки самих комментариев

// С условием
$posts = Post::withCount(['comments' => function ($query) {
    $query->where('approved', true);
}])->get();

// Агрегации
$posts = Post::withMax('comments', 'created_at')
    ->withMin('comments', 'created_at')
    ->withAvg('comments', 'rating')
    ->withSum('comments', 'votes')
    ->get();

echo $posts[0]->comments_max_created_at;
```

### Lazy Eager Loading

```php
$posts = Post::all();

// Загрузить позже, если забыли
if ($needUsers) {
    $posts->load('user');
}

// С условием
$posts->load(['comments' => function ($query) {
    $query->recent();
}]);

// Загрузить если еще не загружено
$posts->loadMissing('user');
```

### Preventing Lazy Loading (Laravel 8.43+)

```php
// В AppServiceProvider
use Illuminate\Database\Eloquent\Model;

public function boot()
{
    // Выбрасывать исключение при Lazy Loading в не-production
    Model::preventLazyLoading(!app()->isProduction());
    
    // Или всегда запрещать
    Model::preventLazyLoading();
}

// Теперь это вызовет исключение:
$post = Post::first();
$post->user; // LazyLoadingViolationException
```

---

## Accessors & Mutators

### Accessors (Геттеры)

```php
class User extends Model
{
    // Старый синтаксис (Laravel < 9)
    public function getFirstNameAttribute($value)
    {
        return ucfirst($value);
    }

    // Новый синтаксис (Laravel 9+)
    protected function firstName(): Attribute
    {
        return Attribute::make(
            get: fn ($value) => ucfirst($value),
        );
    }

    // Вычисляемый атрибут
    protected function fullName(): Attribute
    {
        return Attribute::make(
            get: fn () => "{$this->first_name} {$this->last_name}",
        );
    }
}

// Использование
$user = User::find(1);
echo $user->first_name; // Автоматически через accessor
echo $user->full_name;  // Вычисляемое свойство

// Добавить в JSON
protected $appends = ['full_name'];
```

### Mutators (Сеттеры)

```php
class User extends Model
{
    // Старый синтаксис
    public function setFirstNameAttribute($value)
    {
        $this->attributes['first_name'] = strtolower($value);
    }

    // Новый синтаксис (Laravel 9+)
    protected function firstName(): Attribute
    {
        return Attribute::make(
            get: fn ($value) => ucfirst($value),
            set: fn ($value) => strtolower($value),
        );
    }

    // Только setter
    protected function password(): Attribute
    {
        return Attribute::make(
            set: fn ($value) => bcrypt($value),
        );
    }
}

// Использование
$user = new User();
$user->first_name = 'JOHN'; // Сохранится как 'john'
$user->password = 'secret'; // Автоматически bcrypt
```

---

## Scopes

### Local Scopes

```php
class Post extends Model
{
    // Local scope (метод scope*)
    public function scopePublished($query)
    {
        return $query->where('status', 'published');
    }

    public function scopePopular($query)
    {
        return $query->where('views', '>', 1000);
    }

    // С параметрами
    public function scopeOfType($query, $type)
    {
        return $query->where('type', $type);
    }

    // Сложный scope
    public function scopeRecent($query, $days = 7)
    {
        return $query->where('created_at', '>=', now()->subDays($days))
                     ->orderBy('created_at', 'desc');
    }
}

// Использование
$posts = Post::published()->get();
$posts = Post::published()->popular()->get(); // Цепочка
$posts = Post::ofType('article')->recent(30)->get();
```

### Global Scopes

```php
// Создать scope класс
namespace App\Models\Scopes;

use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Scope;

class PublishedScope implements Scope
{
    public function apply(Builder $builder, Model $model)
    {
        $builder->where('status', 'published');
    }
}

// Применить в модели
class Post extends Model
{
    protected static function booted()
    {
        static::addGlobalScope(new PublishedScope);
        
        // Или анонимный scope
        static::addGlobalScope('published', function (Builder $builder) {
            $builder->where('status', 'published');
        });
    }
}

// Теперь ВСЕ запросы будут фильтроваться
$posts = Post::all(); // WHERE status = 'published'

// Отключить global scope
$posts = Post::withoutGlobalScope(PublishedScope::class)->get();
$posts = Post::withoutGlobalScope('published')->get();
$posts = Post::withoutGlobalScopes()->get(); // Все scopes
```

---

## Query Builder в Eloquent

```php
// WHERE
Post::where('status', 'published')->get();
Post::where('views', '>', 100)->get();
Post::where('status', '!=', 'draft')->get();

// Несколько условий
Post::where('status', 'published')
    ->where('views', '>', 100)
    ->get();

// OR
Post::where('status', 'published')
    ->orWhere('featured', true)
    ->get();

// WHERE IN
Post::whereIn('status', ['published', 'featured'])->get();
Post::whereNotIn('status', ['draft', 'pending'])->get();

// NULL проверки
Post::whereNull('deleted_at')->get();
Post::whereNotNull('published_at')->get();

// BETWEEN
Post::whereBetween('created_at', [$start, $end])->get();

// LIKE
Post::where('title', 'like', '%Laravel%')->get();

// JSON (PostgreSQL/MySQL 5.7+)
Post::where('metadata->author->name', 'John')->get();

// Группировка условий
Post::where('status', 'published')
    ->where(function ($query) {
        $query->where('views', '>', 1000)
              ->orWhere('featured', true);
    })
    ->get();
// WHERE status = 'published' AND (views > 1000 OR featured = true)

// ORDER BY
Post::orderBy('created_at', 'desc')->get();
Post::orderBy('views', 'desc')->orderBy('title')->get();
Post::latest()->get(); // ORDER BY created_at DESC
Post::oldest()->get(); // ORDER BY created_at ASC

// LIMIT / OFFSET
Post::limit(10)->get();
Post::take(10)->get(); // Алиас для limit
Post::skip(20)->take(10)->get(); // OFFSET 20 LIMIT 10

// GROUP BY
Post::select('user_id', DB::raw('count(*) as posts_count'))
    ->groupBy('user_id')
    ->get();

// HAVING
Post::groupBy('user_id')
    ->having('posts_count', '>', 5)
    ->get();

// DISTINCT
Post::distinct()->select('user_id')->get();

// Aggregate функции
$count = Post::count();
$max = Post::max('views');
$min = Post::min('views');
$avg = Post::avg('views');
$sum = Post::sum('views');

// Проверки
$exists = Post::where('slug', $slug)->exists();
$doesntExist = Post::where('slug', $slug)->doesntExist();
```

---

## Продвинутые техники

### Chunking - обработка больших данных

```php
// ❌ ПЛОХО: Загрузит ВСЕ записи в память
$posts = Post::all(); // 1 миллион записей = Out of Memory!

// ✅ ХОРОШО: По кускам
Post::chunk(200, function ($posts) {
    foreach ($posts as $post) {
        // Обработка
    }
});

// Lazy loading (Laravel 8+) - лучше для больших данных
Post::lazy(200)->each(function ($post) {
    // Обработка по одной записи
});

// LazyById - безопаснее при обновлениях
Post::lazyById(200)->each(function ($post) {
    $post->update(['processed' => true]);
});

// Cursor - еще меньше памяти (один запрос, стриминг)
foreach (Post::cursor() as $post) {
    // Минимальное потребление памяти
}
```

### Subqueries

```php
use Illuminate\Database\Eloquent\Builder;

// Подзапросы в SELECT
$users = User::select([
    'users.*',
    'last_post_created_at' => Post::select('created_at')
        ->whereColumn('user_id', 'users.id')
        ->latest()
        ->limit(1)
])->get();

// Подзапросы в WHERE
$users = User::where('id', function ($query) {
    $query->select('user_id')
          ->from('posts')
          ->where('status', 'published');
})->get();

// whereExists
$users = User::whereExists(function ($query) {
    $query->select(DB::raw(1))
          ->from('posts')
          ->whereColumn('posts.user_id', 'users.id');
})->get();
```

### Conditional Queries

```php
// when() - условное добавление запроса
$posts = Post::query()
    ->when($request->status, function ($query, $status) {
        $query->where('status', $status);
    })
    ->when($request->author, function ($query, $author) {
        $query->where('user_id', $author);
    })
    ->get();

// unless() - противоположность when
$posts = Post::unless($includeUnpublished, function ($query) {
    $query->where('status', 'published');
})->get();
```

### Upsert (Laravel 8+)

```php
// INSERT или UPDATE если есть
Post::upsert(
    [
        ['slug' => 'post-1', 'title' => 'Title 1', 'views' => 100],
        ['slug' => 'post-2', 'title' => 'Title 2', 'views' => 200],
    ],
    ['slug'], // Уникальные поля для проверки
    ['title', 'views'] // Какие поля обновлять
);
```

---

## События (Events)

```php
class Post extends Model
{
    protected static function booted()
    {
        // Срабатывает перед созданием
        static::creating(function ($post) {
            $post->slug = Str::slug($post->title);
        });

        // После создания
        static::created(function ($post) {
            // Отправить уведомление
        });

        // Перед обновлением
        static::updating(function ($post) {
            if ($post->isDirty('status')) {
                // Статус изменился
            }
        });

        // После обновления
        static::updated(function ($post) {
            Cache::forget("post.{$post->id}");
        });

        // Перед сохранением (create ИЛИ update)
        static::saving(function ($post) {
            // Выполнится всегда
        });

        // После сохранения
        static::saved(function ($post) {
            //
        });

        // Перед удалением
        static::deleting(function ($post) {
            // Удалить связанные данные
            $post->comments()->delete();
        });

        // После удаления
        static::deleted(function ($post) {
            //
        });

        // Восстановление (SoftDeletes)
        static::restoring(function ($post) {
            //
        });

        static::restored(function ($post) {
            //
        });
    }
}
```

---

## Performance Tips

### 1. Select только нужные колонки

```php
// ❌ ПЛОХО
$users = User::all(); // SELECT *

// ✅ ХОРОШО
$users = User::select(['id', 'name', 'email'])->get();
```

### 2. Используйте exists() вместо count()

```php
// ❌ ПЛОХО
if (Post::where('user_id', $userId)->count() > 0) { }

// ✅ ХОРОШО
if (Post::where('user_id', $userId)->exists()) { }
```

### 3. Индексируйте внешние ключи

```php
// В миграции
$table->foreignId('user_id')->constrained()->onDelete('cascade');
// Автоматически создаст индекс

// Или вручную
$table->index('user_id');
```

### 4. Используйте Eager Loading (избегайте N+1)

```php
// ❌ ПЛОХО
$posts = Post::all();
foreach ($posts as $post) {
    echo $post->user->name; // N+1
}

// ✅ ХОРОШО
$posts = Post::with('user')->all();
```

### 5. Кеширование запросов

```php
$posts = Cache::remember('posts.recent', 3600, function () {
    return Post::with('user')->latest()->limit(10)->get();
});
```

---

## Best Practices

1. **Всегда используйте Eager Loading** для отношений
2. **Используйте scopes** для повторяющихся запросов
3. **Индексируйте** часто используемые в WHERE колонки
4. **Select только нужные поля**, не `SELECT *`
5. **Используйте chunk/lazy** для больших объемов данных
6. **Кешируйте** результаты тяжелых запросов
7. **Проверяйте isDirty()** перед обновлениями в событиях
8. **Используйте preventLazyLoading()** в разработке
9. **Создавайте индексы** для внешних ключей
10. **Используйте transactions** для множественных операций

---

## Дебаггинг

```php
// Получить SQL запрос
$query = Post::where('status', 'published')->toSql();
dd($query); // "select * from posts where status = ?"

// С bindings
$query = Post::where('status', 'published');
dd($query->toRawSql()); // Laravel 10.15+

// Логирование всех запросов
DB::enableQueryLog();
$posts = Post::with('user')->get();
dd(DB::getQueryLog());

// Explain
$posts = Post::where('status', 'published')->explain();
dd($posts);
```

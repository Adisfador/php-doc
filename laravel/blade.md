# Blade Templates - Шаблонизатор Laravel

Полный разбор Blade - мощного и элегантного шаблонизатора Laravel.

---

## 🎯 Что такое Blade?

**Blade** - шаблонизатор Laravel с компиляцией в чистый PHP.

**Преимущества:**
- 🚀 Быстрый (компилируется в PHP, кешируется)
- 💡 Выразительный синтаксис
- 🔒 Автоматическое экранирование XSS
- 🧩 Компонентная архитектура
- 🔄 Наследование шаблонов

```php
// resources/views/welcome.blade.php
<!DOCTYPE html>
<html>
<head>
    <title>{{ $title }}</title>
</head>
<body>
    <h1>Hello, {{ $name }}!</h1>
</body>
</html>
```

---

## 📄 Основы

### Вывод данных

```blade
{{-- Экранированный вывод (безопасно) --}}
<h1>{{ $title }}</h1>
<p>{{ $user->name }}</p>

{{-- Неэкранированный вывод (опасно!) --}}
<div>{!! $htmlContent !!}</div>

{{-- Вывод с fallback --}}
<p>{{ $name ?? 'Guest' }}</p>

{{-- Blade директивы внутри атрибутов --}}
<input type="text" value="{{ old('name', $user->name) }}">
```

### Комментарии

```blade
{{-- Blade комментарий (не попадает в HTML) --}}

<!-- HTML комментарий (попадает в HTML) -->
```

### Сырой PHP

```blade
@php
    $counter = 0;
    $total = count($items);
@endphp

{{-- Или inline --}}
@php($counter = 0)
```

---

## 🔀 Условия

### @if / @elseif / @else

```blade
@if ($user->isAdmin())
    <p>Welcome, Admin!</p>
@elseif ($user->isModerator())
    <p>Welcome, Moderator!</p>
@else
    <p>Welcome, User!</p>
@endif
```

### @unless

```blade
@unless ($user->isSubscribed())
    <p>Please subscribe to continue.</p>
@endunless

{{-- Эквивалентно --}}
@if (!$user->isSubscribed())
    <p>Please subscribe to continue.</p>
@endif
```

### @isset / @empty

```blade
@isset($user)
    <p>User: {{ $user->name }}</p>
@endisset

@empty($posts)
    <p>No posts found.</p>
@endempty
```

### @auth / @guest

```blade
@auth
    <p>Welcome back, {{ auth()->user()->name }}!</p>
@endauth

@guest
    <a href="{{ route('login') }}">Login</a>
@endguest

{{-- С guard --}}
@auth('admin')
    <p>Admin panel</p>
@endauth
```

### @can / @cannot (Авторизация)

```blade
@can('update', $post)
    <a href="{{ route('posts.edit', $post) }}">Edit</a>
@endcan

@cannot('delete', $post)
    <p>You cannot delete this post.</p>
@endcannot

{{-- С @else --}}
@can('update', $post)
    <a href="{{ route('posts.edit', $post) }}">Edit</a>
@else
    <p>You cannot edit this post.</p>
@endcan
```

### @hasSection

```blade
@hasSection('navigation')
    <div class="nav">
        @yield('navigation')
    </div>
@endif
```

### @production / @env

```blade
@production
    {{-- Только в production --}}
    <script src="analytics.js"></script>
@endproduction

@env('staging')
    {{-- Только в staging --}}
    <div class="staging-banner">Staging Environment</div>
@endenv

{{-- Несколько окружений --}}
@env(['local', 'staging'])
    <div class="debug-panel"></div>
@endenv
```

---

## 🔁 Циклы

### @foreach

```blade
@foreach ($users as $user)
    <li>{{ $user->name }}</li>
@endforeach

{{-- С индексом --}}
@foreach ($users as $index => $user)
    <li>{{ $index }}: {{ $user->name }}</li>
@endforeach
```

### @forelse

```blade
@forelse ($posts as $post)
    <article>
        <h2>{{ $post->title }}</h2>
        <p>{{ $post->excerpt }}</p>
    </article>
@empty
    <p>No posts available.</p>
@endforelse
```

### @for / @while

```blade
@for ($i = 0; $i < 10; $i++)
    <p>Item {{ $i }}</p>
@endfor

@while ($counter < 10)
    <p>{{ $counter }}</p>
    @php($counter++)
@endwhile
```

### $loop переменная

```blade
@foreach ($users as $user)
    <li>
        {{ $user->name }}
        
        @if ($loop->first)
            (First)
        @endif
        
        @if ($loop->last)
            (Last)
        @endif
        
        {{-- Доступные свойства --}}
        Index: {{ $loop->index }}       {{-- 0, 1, 2... --}}
        Iteration: {{ $loop->iteration }}  {{-- 1, 2, 3... --}}
        Remaining: {{ $loop->remaining }}  {{-- Осталось --}}
        Count: {{ $loop->count }}       {{-- Всего --}}
        
        @if ($loop->even)
            (Even row)
        @endif
        
        @if ($loop->odd)
            (Odd row)
        @endif
        
        Depth: {{ $loop->depth }}       {{-- Уровень вложенности --}}
        Parent: {{ $loop->parent }}     {{-- Родительский $loop --}}
    </li>
@endforeach
```

### @break / @continue

```blade
@foreach ($users as $user)
    @if ($loop->index === 5)
        @break
    @endif
    
    @if ($user->isInactive())
        @continue
    @endif
    
    <li>{{ $user->name }}</li>
@endforeach

{{-- С условием --}}
@foreach ($users as $user)
    @continue($user->isInactive())
    @break($loop->index === 5)
    
    <li>{{ $user->name }}</li>
@endforeach
```

---

## 📦 Подключение шаблонов

### @include

```blade
@include('partials.header')

@include('partials.sidebar', ['active' => 'dashboard'])

{{-- С проверкой существования --}}
@includeIf('partials.footer')

{{-- Первый найденный --}}
@includeFirst(['custom.header', 'partials.header'])

{{-- Когда условие true --}}
@includeWhen($user->isAdmin(), 'partials.admin-menu')

{{-- Если НЕ выполнено условие --}}
@includeUnless($user->isGuest(), 'partials.user-menu')
```

### @each

```blade
{{-- Цикл с view --}}
@each('partials.user-card', $users, 'user')

{{-- С fallback если массив пустой --}}
@each('partials.user-card', $users, 'user', 'partials.no-users')
```

---

## 🏗️ Layouts (Наследование)

### Родительский layout

```blade
{{-- resources/views/layouts/app.blade.php --}}
<!DOCTYPE html>
<html>
<head>
    <title>@yield('title', 'Default Title')</title>
    @stack('styles')
</head>
<body>
    <nav>
        @section('navigation')
            <a href="/">Home</a>
            <a href="/about">About</a>
        @show
    </nav>
    
    <main>
        @yield('content')
    </main>
    
    <footer>
        @yield('footer')
    </footer>
    
    @stack('scripts')
</body>
</html>
```

### Дочерний шаблон

```blade
{{-- resources/views/pages/home.blade.php --}}
@extends('layouts.app')

@section('title', 'Home Page')

@section('navigation')
    @parent  {{-- Показать содержимое родительского @section --}}
    <a href="/contact">Contact</a>
@endsection

@section('content')
    <h1>Welcome Home!</h1>
    <p>This is the home page.</p>
@endsection

@section('footer')
    <p>&copy; 2026 My Company</p>
@endsection

@push('styles')
    <link rel="stylesheet" href="home.css">
@endpush

@push('scripts')
    <script src="home.js"></script>
@endpush
```

### @yield vs @section

```blade
{{-- @yield - простая вставка --}}
@yield('content')

{{-- @section...@show - с возможностью расширения через @parent --}}
@section('content')
    Default content
@show
```

### @stack / @push / @prepend

```blade
{{-- Layout --}}
<head>
    @stack('styles')
</head>
<body>
    @stack('scripts')
</body>

{{-- View --}}
@push('scripts')
    <script src="first.js"></script>
@endpush

@push('scripts')
    <script src="second.js"></script>
@endpush

{{-- Добавить в начало --}}
@prepend('scripts')
    <script src="should-be-first.js"></script>
@endprepend

{{-- Результат: --}}
<script src="should-be-first.js"></script>
<script src="first.js"></script>
<script src="second.js"></script>
```

---

## 🧩 Components (Компоненты)

### Class-based Components

**Создание:**
```bash
php artisan make:component Alert
```

**app/View/Components/Alert.php:**
```php
<?php

namespace App\View\Components;

use Illuminate\View\Component;

class Alert extends Component
{
    public function __construct(
        public string $type = 'info',
        public string $message = ''
    ) {}
    
    public function render()
    {
        return view('components.alert');
    }
    
    // Computed свойства
    public function iconClass(): string
    {
        return match($this->type) {
            'success' => 'icon-check',
            'error' => 'icon-x',
            default => 'icon-info',
        };
    }
}
```

**resources/views/components/alert.blade.php:**
```blade
<div class="alert alert-{{ $type }}">
    <span class="{{ $iconClass() }}"></span>
    {{ $message }}
    {{ $slot }}
</div>
```

**Использование:**
```blade
<x-alert type="success" message="Operation completed!" />

<x-alert type="error">
    <strong>Error!</strong> Something went wrong.
</x-alert>
```

### Anonymous Components

**resources/views/components/button.blade.php:**
```blade
@props(['type' => 'button', 'color' => 'primary'])

<button type="{{ $type }}" {{ $attributes->merge(['class' => "btn btn-$color"]) }}>
    {{ $slot }}
</button>
```

**Использование:**
```blade
<x-button>Click me</x-button>
<x-button type="submit" color="success">Submit</x-button>
<x-button class="extra-class" id="my-btn">Custom</x-button>
```

### @props - Атрибуты компонента

```blade
@props([
    'type' => 'info',        // С default
    'dismissible' => false,
    'title'                  // Обязательный
])

<div class="alert alert-{{ $type }} {{ $dismissible ? 'alert-dismissible' : '' }}">
    @isset($title)
        <h4>{{ $title }}</h4>
    @endisset
    {{ $slot }}
</div>
```

### Slots (Слоты)

**Компонент с несколькими слотами:**
```blade
{{-- components/card.blade.php --}}
<div class="card">
    <div class="card-header">
        {{ $header }}
    </div>
    
    <div class="card-body">
        {{ $slot }}
    </div>
    
    @isset($footer)
        <div class="card-footer">
            {{ $footer }}
        </div>
    @endisset
</div>
```

**Использование:**
```blade
<x-card>
    <x-slot:header>
        Card Title
    </x-slot:header>
    
    This is the card body content.
    
    <x-slot:footer>
        <button>Action</button>
    </x-slot:footer>
</x-card>
```

### $attributes - Управление атрибутами

```blade
{{-- Передать все атрибуты --}}
<div {{ $attributes }}>
    {{ $slot }}
</div>

{{-- Merge классов --}}
<button {{ $attributes->merge(['class' => 'btn']) }}>
    {{ $slot }}
</button>
{{-- <x-button class="btn-lg"> → class="btn btn-lg" --}}

{{-- Условный merge --}}
<button {{ $attributes->class(['btn', 'btn-primary' => $primary]) }}>
    {{ $slot }}
</button>

{{-- Только определенные атрибуты --}}
<div {{ $attributes->only(['id', 'class']) }}>
    {{ $slot }}
</div>

{{-- Все кроме определенных --}}
<div {{ $attributes->except(['wire:model']) }}>
    {{ $slot }}
</div>

{{-- Проверка наличия --}}
@if ($attributes->has('disabled'))
    {{-- ... --}}
@endif

{{-- Получить значение --}}
{{ $attributes->get('id') }}

{{-- Фильтрация --}}
<div {{ $attributes->filter(fn($value, $key) => $key !== 'wire:model') }}>
    {{ $slot }}
</div>
```

### Component Attributes

```blade
{{-- components/input.blade.php --}}
@props(['disabled' => false])

<input {{ $attributes->merge([
    'class' => 'form-control',
    'disabled' => $disabled
]) }}>

{{-- Использование --}}
<x-input type="email" name="email" required />
<x-input disabled />
```

---

## 🎨 Директивы форм

### @csrf

```blade
<form method="POST" action="/profile">
    @csrf
    {{-- Генерирует: <input type="hidden" name="_token" value="..."> --}}
</form>
```

### @method

```blade
<form method="POST" action="/posts/1">
    @csrf
    @method('PUT')
    {{-- Генерирует: <input type="hidden" name="_method" value="PUT"> --}}
</form>

{{-- Или DELETE, PATCH --}}
<form method="POST" action="/posts/1">
    @csrf
    @method('DELETE')
</form>
```

### @error

```blade
<input type="text" name="email" value="{{ old('email') }}">
@error('email')
    <div class="alert alert-danger">{{ $message }}</div>
@enderror

{{-- Для конкретного bag --}}
@error('email', 'login')
    <div>{{ $message }}</div>
@enderror
```

### old()

```blade
<input type="text" name="username" value="{{ old('username') }}">
<input type="email" name="email" value="{{ old('email', $user->email) }}">
```

---

## 🔧 Вспомогательные директивы

### @json

```blade
<script>
    var users = @json($users);
    var config = @json($config, JSON_PRETTY_PRINT);
</script>
```

### @dd / @dump

```blade
@dd($users)  {{-- Die and dump --}}
@dump($users)  {{-- Dump and continue --}}
```

### @class / @style

```blade
{{-- Условные классы --}}
<div @class([
    'alert',
    'alert-success' => $success,
    'alert-danger' => !$success,
    'dismissible' => true
])>
    Alert
</div>

{{-- Условные стили --}}
<span @style([
    'color: red' => $hasError,
    'font-weight: bold',
    'display: none' => !$visible
])>
    Text
</span>
```

### @checked / @selected / @disabled / @readonly / @required

```blade
{{-- Checkbox/Radio --}}
<input type="checkbox" name="active" @checked(old('active', $user->active)) />

{{-- Select --}}
<select name="role">
    <option @selected(old('role') === 'admin')>Admin</option>
    <option @selected(old('role') === 'user')>User</option>
</select>

{{-- Disabled --}}
<input type="text" @disabled($user->isLocked()) />

{{-- Readonly --}}
<input type="text" @readonly($user->isVerified()) />

{{-- Required --}}
<input type="text" name="name" @required(true) />
```

---

## 🌐 Blade Directives (Пользовательские)

### Создание директивы

```php
// AppServiceProvider::boot()
use Illuminate\Support\Facades\Blade;

Blade::directive('datetime', function ($expression) {
    return "<?php echo ($expression)->format('Y-m-d H:i'); ?>";
});

// Использование
@datetime($post->created_at)
```

### If директива

```php
Blade::if('admin', function () {
    return auth()->check() && auth()->user()->isAdmin();
});

// Использование
@admin
    <p>Admin content</p>
@endadmin

@admin
    <p>Admin content</p>
@else
    <p>Regular user content</p>
@endadmin
```

### Примеры директив

```php
// @svg('icon-name')
Blade::directive('svg', function ($expression) {
    return "<?php echo file_get_contents(public_path('icons/' . $expression . '.svg')); ?>";
});

// @money($amount)
Blade::directive('money', function ($expression) {
    return "<?php echo number_format($expression, 2); ?>";
});

// @format_phone($phone)
Blade::directive('format_phone', function ($expression) {
    return "<?php echo preg_replace('/(\d{3})(\d{3})(\d{4})/', '($1) $2-$3', $expression); ?>";
});
```

---

## 📋 Дополнительные возможности

### Blade::render() - Рендер строки

```php
use Illuminate\Support\Facades\Blade;

$rendered = Blade::render('Hello, {{ $name }}', ['name' => 'John']);
// "Hello, John"
```

### @once - Выполнить один раз

```blade
@once
    <script src="jquery.js"></script>
@endonce

{{-- Даже если компонент используется много раз, скрипт загрузится 1 раз --}}
```

### @verbatim - Игнорировать Blade

```blade
@verbatim
    <div class="container">
        Hello, {{ name }}  {{-- Не обрабатывается Blade --}}
    </div>
@endverbatim

{{-- Полезно для Vue.js, Alpine.js и т.д. --}}
```

### @aware - Доступ к данным родителя

```blade
{{-- components/menu-item.blade.php --}}
@aware(['color' => 'gray'])

<li class="text-{{ $color }}">
    {{ $slot }}
</li>

{{-- Использование --}}
<x-menu color="blue">
    <x-menu-item>Home</x-menu-item>  {{-- Унаследует color="blue" --}}
    <x-menu-item>About</x-menu-item>
</x-menu>
```

---

## 🚀 Оптимизация

### Компиляция шаблонов

```bash
# Скомпилировать все Blade шаблоны
php artisan view:cache

# Очистить кеш
php artisan view:clear
```

### Precompiling

```php
// В сервис-провайдере
use Illuminate\Support\Facades\Blade;

public function boot()
{
    Blade::precompiler(function ($string) {
        // Препроцессинг перед компиляцией
        return str_replace('[[', '{{', $string);
    });
}
```

---

## 🎓 Best Practices

### 1. Используй компоненты для переиспользования

```blade
{{-- ❌ ПЛОХО - дублирование --}}
<div class="alert alert-success">Success!</div>
<div class="alert alert-error">Error!</div>

{{-- ✅ ХОРОШО - компонент --}}
<x-alert type="success">Success!</x-alert>
<x-alert type="error">Error!</x-alert>
```

### 2. Логику в контроллер/компонент, не в шаблон

```blade
{{-- ❌ ПЛОХО --}}
@php
    $total = 0;
    foreach ($items as $item) {
        $total += $item->price * $item->quantity;
    }
@endphp

{{-- ✅ ХОРОШО - передай из контроллера --}}
{{ $total }}
```

### 3. Используй @forelse вместо @if + @foreach

```blade
{{-- ❌ ПЛОХО --}}
@if (count($posts) > 0)
    @foreach ($posts as $post)
        <article>{{ $post->title }}</article>
    @endforeach
@else
    <p>No posts</p>
@endif

{{-- ✅ ХОРОШО --}}
@forelse ($posts as $post)
    <article>{{ $post->title }}</article>
@empty
    <p>No posts</p>
@endforelse
```

### 4. Экранируй все пользовательские данные

```blade
{{-- ✅ БЕЗОПАСНО (auto-escaped) --}}
{{ $userInput }}

{{-- ❌ ОПАСНО - только для доверенного HTML --}}
{!! $userInput !!}
```

### 5. Разделяй шаблоны на компоненты

```blade
{{-- ❌ ПЛОХО - всё в одном файле --}}
<div class="page">
    <nav>...</nav>
    <main>...</main>
    <footer>...</footer>
</div>

{{-- ✅ ХОРОШО --}}
@include('partials.nav')
<main>@yield('content')</main>
@include('partials.footer')
```

---

## 🎓 Для собеседования: ключевые точки

1. **{{ }} vs {!! !!}** - экранированный vs неэкранированный вывод
2. **@yield vs @section** - простая вставка vs расширяемая секция с @parent
3. **@include vs Components** - простое подключение vs переиспользуемые компоненты
4. **Class vs Anonymous Components** - с логикой vs простые шаблоны
5. **$attributes** - передача HTML атрибутов в компоненты, merge/filter/only
6. **@props** - определение props компонента с defaults
7. **Slots** - именованные слоты для гибких компонентов
8. **$loop** - переменная внутри циклов (first/last/index/parent)
9. **@stack/@push** - накопление контента (scripts/styles)
10. **@once** - выполнить только один раз (даже при множественном использовании)

**Главное:** Blade компилируется в чистый PHP и кешируется, поэтому быстрый. Компоненты делают код модульным и переиспользуемым.

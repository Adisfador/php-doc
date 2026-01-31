# Composer - Dependency Manager

Полный разбор Composer: установка, composer.json, версии, autoloading, команды, оптимизация.

---

## 🎯 Что такое Composer?

**Composer** - менеджер зависимостей для PHP (аналог npm для Node.js, pip для Python).

**Основные задачи:**
- 📦 Управление пакетами (установка, обновление, удаление)
- 🔄 Разрешение зависимостей (dependency resolution)
- 🔐 Autoloading (PSR-4, classmap, files)
- 📋 Версионирование (semver)

**Установка:**
```bash
# Linux/macOS
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
php composer-setup.php
php -r "unlink('composer-setup.php');"
sudo mv composer.phar /usr/local/bin/composer

# Проверка
composer --version
```

---

## 📄 composer.json - конфигурация проекта

### Базовая структура

```json
{
    "name": "vendor/package-name",
    "description": "Package description",
    "type": "library",
    "license": "MIT",
    "authors": [
        {
            "name": "John Doe",
            "email": "john@example.com",
            "homepage": "https://example.com",
            "role": "Developer"
        }
    ],
    "require": {
        "php": "^8.1",
        "monolog/monolog": "^3.0",
        "guzzlehttp/guzzle": "^7.5"
    },
    "require-dev": {
        "phpunit/phpunit": "^10.0",
        "phpstan/phpstan": "^1.10"
    },
    "autoload": {
        "psr-4": {
            "App\\": "src/"
        },
        "files": [
            "src/helpers.php"
        ]
    },
    "autoload-dev": {
        "psr-4": {
            "Tests\\": "tests/"
        }
    },
    "scripts": {
        "test": "phpunit",
        "stan": "phpstan analyse src",
        "post-install-cmd": [
            "php artisan clear-compiled"
        ]
    },
    "config": {
        "optimize-autoloader": true,
        "preferred-install": "dist",
        "sort-packages": true,
        "allow-plugins": {
            "pestphp/pest-plugin": true
        }
    },
    "minimum-stability": "stable",
    "prefer-stable": true
}
```

### Ключевые поля

**name** - уникальное имя пакета (vendor/package):
```json
{
    "name": "laravel/framework"
}
```

**type** - тип пакета:
- `library` - библиотека (по умолчанию)
- `project` - полноценный проект (Laravel app)
- `metapackage` - группа зависимостей
- `composer-plugin` - плагин Composer

**require** - production зависимости:
```json
{
    "require": {
        "php": "^8.1",
        "ext-mbstring": "*",
        "monolog/monolog": "^3.0"
    }
}
```

**require-dev** - development зависимости (тесты, анализаторы):
```json
{
    "require-dev": {
        "phpunit/phpunit": "^10.0",
        "mockery/mockery": "^1.5",
        "phpstan/phpstan": "^1.10"
    }
}
```

---

## 📦 Semantic Versioning (Semver)

### Формат версии: MAJOR.MINOR.PATCH

```
1.2.3
│ │ │
│ │ └─ PATCH - исправления багов (backward compatible)
│ └─── MINOR - новые функции (backward compatible)
└───── MAJOR - breaking changes (несовместимые изменения)
```

**Примеры:**
- `1.0.0` → `1.0.1` - патч (fix бага)
- `1.0.1` → `1.1.0` - новая функция (без breaking changes)
- `1.1.0` → `2.0.0` - breaking change

### Операторы версий

**Caret (^) - совместимые версии:**
```json
{
    "require": {
        "monolog/monolog": "^3.0"
    }
}
```
- `^3.0` = `>=3.0.0 <4.0.0` - любая 3.x версия
- `^3.2.1` = `>=3.2.1 <4.0.0`
- `^0.3` = `>=0.3.0 <0.4.0` - для 0.x строже!

**Tilde (~) - ближайшая совместимая версия:**
```json
{
    "require": {
        "symfony/console": "~5.4"
    }
}
```
- `~5.4` = `>=5.4.0 <6.0.0` - любая 5.x >= 5.4
- `~5.4.3` = `>=5.4.3 <5.5.0` - изменяет только PATCH

**Wildcard (*) - любая версия:**
```json
{
    "require": {
        "ext-mbstring": "*"
    }
}
```

**Точная версия:**
```json
{
    "require": {
        "some/package": "1.2.3"
    }
}
```

**Диапазон версий:**
```json
{
    "require": {
        "php": ">=8.1 <8.3",
        "guzzle": ">=7.0 <8.0 || ^9.0"
    }
}
```

**Stability flags:**
```json
{
    "require": {
        "monolog/monolog": "3.0@beta",
        "vendor/package": "dev-main"
    },
    "minimum-stability": "stable",
    "prefer-stable": true
}
```

Уровни стабильности: `dev` < `alpha` < `beta` < `RC` < `stable`

---

## 🔄 composer install vs composer update

### composer install

```bash
composer install
```

**Что делает:**
1. Читает `composer.lock` (если существует)
2. Устанавливает **точные версии** из lock-файла
3. Если lock нет → создает его из `composer.json`

**Когда использовать:**
- ✅ Production deployment
- ✅ CI/CD pipeline
- ✅ Новый разработчик клонирует проект
- ✅ Всегда для воспроизводимых builds

**Опции:**
```bash
composer install --no-dev           # без dev-зависимостей
composer install --optimize-autoloader  # оптимизация autoloader
composer install --no-scripts       # без scripts
composer install --prefer-dist      # скачать zip вместо git clone
```

### composer update

```bash
composer update
```

**Что делает:**
1. **Игнорирует** composer.lock
2. Разрешает зависимости из `composer.json`
3. Обновляет пакеты до **новейших совместимых версий**
4. Перезаписывает composer.lock

**Когда использовать:**
- ✅ Обновление зависимостей
- ✅ Development environment
- ⚠️ НИКОГДА на production!

**Опции:**
```bash
composer update monolog/monolog     # обновить только один пакет
composer update --with-dependencies # с зависимостями
composer update --prefer-lowest     # минимальные версии (для тестов)
composer update --lock              # только lock файл без установки
```

---

## 🔐 Autoloading

### PSR-4 Autoloading (рекомендуется)

**composer.json:**
```json
{
    "autoload": {
        "psr-4": {
            "App\\": "src/",
            "App\\Controllers\\": "app/Controllers/",
            "Database\\": "database/"
        }
    }
}
```

**Маппинг:**
```
App\Models\User         → src/Models/User.php
App\Controllers\HomeController → app/Controllers/HomeController.php
Database\Seeders\UserSeeder    → database/Seeders/UserSeeder.php
```

**Правило:**
- Namespace prefix (`App\`) → base directory (`src/`)
- `\` заменяется на `/`
- Добавляется `.php`

### Classmap Autoloading (для legacy кода)

```json
{
    "autoload": {
        "classmap": [
            "legacy/",
            "vendor/some-library/src/"
        ]
    }
}
```

**Как работает:**
- Composer сканирует директории
- Находит все классы
- Создает map `class => file`
- **Минус:** нужно пересоздавать при добавлении классов

```bash
composer dump-autoload  # пересоздать classmap
```

### Files Autoloading (для helpers)

```json
{
    "autoload": {
        "files": [
            "src/helpers.php",
            "config/constants.php"
        ]
    }
}
```

**Загружаются всегда** при каждом запросе (даже если не используются).

**src/helpers.php:**
```php
<?php

if (!function_exists('array_get')) {
    function array_get(array $array, string $key, mixed $default = null): mixed {
        return $array[$key] ?? $default;
    }
}
```

### Exclude from classmap

```json
{
    "autoload": {
        "classmap": ["src/"],
        "exclude-from-classmap": [
            "src/Tests/",
            "src/Migrations/"
        ]
    }
}
```

### Оптимизация Autoloader

```bash
# Для production
composer dump-autoload --optimize
# Или
composer dump-autoload -o

# Еще быстрее (authoritative)
composer dump-autoload --classmap-authoritative
# Или
composer dump-autoload -a
```

**Уровни оптимизации:**

1. **По умолчанию** - PSR-4 lookup через filesystem
2. **--optimize/-o** - создает classmap для PSR-4 классов
3. **--classmap-authoritative/-a** - только classmap, без fallback (самый быстрый, но не найдет новые классы)

**Laravel production:**
```bash
composer install --no-dev --optimize-autoloader
php artisan config:cache
php artisan route:cache
php artisan view:cache
```

---

## 🛠️ Основные команды

### require - добавить пакет

```bash
# Production зависимость
composer require guzzlehttp/guzzle

# Development зависимость
composer require --dev phpunit/phpunit

# С версией
composer require monolog/monolog:^3.0

# Несколько пакетов
composer require guzzle doctrine/dbal symfony/console
```

### remove - удалить пакет

```bash
composer remove guzzlehttp/guzzle

composer remove --dev phpunit/phpunit
```

### show - информация о пакетах

```bash
# Все установленные пакеты
composer show

# Информация о конкретном пакете
composer show monolog/monolog

# Дерево зависимостей
composer show --tree

# Только устаревшие пакеты
composer outdated

# Только direct зависимости
composer show --direct
```

### why / depends - кто зависит от пакета

```bash
# Кто требует symfony/console
composer why symfony/console

# Или
composer depends symfony/console
```

### validate - проверить composer.json

```bash
composer validate

# С предупреждениями
composer validate --strict
```

### dump-autoload - пересоздать autoloader

```bash
composer dump-autoload

# С оптимизацией
composer dump-autoload -o
```

### search - поиск пакетов

```bash
composer search redis
```

### create-project - создать новый проект

```bash
# Laravel
composer create-project laravel/laravel my-app

# Symfony
composer create-project symfony/skeleton my-app
```

### self-update - обновить Composer

```bash
composer self-update

# Откатиться
composer self-update --rollback
```

---

## 📋 Scripts - автоматизация задач

### Определение scripts

```json
{
    "scripts": {
        "test": "phpunit",
        "test:unit": "phpunit --testsuite=Unit",
        "test:feature": "phpunit --testsuite=Feature",
        "analyse": [
            "phpstan analyse",
            "psalm --show-info=true"
        ],
        "cs:check": "phpcs",
        "cs:fix": "phpcbf",
        "post-install-cmd": [
            "@php artisan clear-compiled",
            "@php artisan package:discover"
        ],
        "pre-autoload-dump": "Google\\Task\\Composer::cleanup"
    }
}
```

### Запуск scripts

```bash
composer test
composer test:unit
composer analyse
```

### События (hooks)

**Стандартные события:**
- `pre-install-cmd` - перед `composer install`
- `post-install-cmd` - после `composer install`
- `pre-update-cmd` - перед `composer update`
- `post-update-cmd` - после `composer update`
- `pre-autoload-dump` - перед созданием autoloader
- `post-autoload-dump` - после создания autoloader
- `post-root-package-install` - после создания composer.json
- `post-create-project-cmd` - после `composer create-project`

```json
{
    "scripts": {
        "post-install-cmd": [
            "@php artisan vendor:publish --tag=laravel-assets --ansi --force"
        ],
        "post-update-cmd": [
            "@php artisan vendor:publish --tag=laravel-assets --ansi --force",
            "@php artisan clear-compiled"
        ],
        "post-autoload-dump": [
            "Illuminate\\Foundation\\ComposerScripts::postAutoloadDump",
            "@php artisan package:discover --ansi"
        ]
    }
}
```

### Script aliases

```json
{
    "scripts": {
        "test": "phpunit",
        "ci": [
            "@test",
            "@analyse",
            "@cs:check"
        ]
    }
}
```

Вызов: `composer ci`

---

## 🌐 Repositories - источники пакетов

### Packagist.org (по умолчанию)

```json
{
    "repositories": [
        {
            "type": "composer",
            "url": "https://packagist.org"
        }
    ]
}
```

### Private VCS repository

```json
{
    "repositories": [
        {
            "type": "vcs",
            "url": "git@github.com:company/private-package.git"
        }
    ],
    "require": {
        "company/private-package": "^1.0"
    }
}
```

### Path repository (local development)

```json
{
    "repositories": [
        {
            "type": "path",
            "url": "../my-local-package",
            "options": {
                "symlink": true
            }
        }
    ],
    "require": {
        "vendor/my-local-package": "dev-main"
    }
}
```

### Private Packagist / Satis

```json
{
    "repositories": [
        {
            "type": "composer",
            "url": "https://packagist.company.com",
            "options": {
                "http": {
                    "header": [
                        "API-TOKEN: your-token"
                    ]
                }
            }
        }
    ]
}
```

### Отключить Packagist

```json
{
    "repositories": [
        {
            "packagist.org": false
        },
        {
            "type": "composer",
            "url": "https://private-packagist.com"
        }
    ]
}
```

---

## 🔒 composer.lock - фиксация версий

### Зачем нужен lock файл?

**composer.json:**
```json
{
    "require": {
        "monolog/monolog": "^3.0"
    }
}
```

**Проблема без lock:**
- Developer 1 устанавливает: `3.0.0` (1 января)
- Developer 2 устанавливает: `3.5.0` (1 июня)
- Production: `3.7.0` (1 сентября)
- **Разные версии = разное поведение!**

**Решение с lock:**
```json
// composer.lock
{
    "packages": [
        {
            "name": "monolog/monolog",
            "version": "3.2.1",
            "source": {
                "type": "git",
                "url": "https://github.com/Seldaek/monolog.git",
                "reference": "abc123"
            }
        }
    ]
}
```

- ✅ Все установят **ровно 3.2.1**
- ✅ Воспроизводимые builds

### Workflow

```bash
# Developer 1 (обновляет зависимости)
composer update
git add composer.lock
git commit -m "Update dependencies"
git push

# Developer 2 (устанавливает те же версии)
git pull
composer install  # использует composer.lock

# Production
git pull
composer install --no-dev --optimize-autoloader
```

**Правило:** `composer.lock` **ВСЕГДА** в git для проектов (library может не включать).

---

## 🚀 Оптимизация для Production

### 1. Install без dev-зависимостей

```bash
composer install --no-dev
```

Исключает: PHPUnit, PHPStan, Psalm, Mockery и т.д.

### 2. Оптимизация autoloader

```bash
composer dump-autoload --optimize

# Или
composer dump-autoload --classmap-authoritative
```

**Разница:**
- `--optimize` - создает classmap для PSR-4, fallback на filesystem
- `--classmap-authoritative` - только classmap, **без fallback** (быстрее, но новые классы не найдет)

### 3. Prefer dist вместо source

```bash
composer install --prefer-dist
```

- `--prefer-dist` - скачивает .zip (быстрее)
- `--prefer-source` - клонирует git repo (для разработки)

### 4. Без scripts

```bash
composer install --no-scripts
```

Пропускает post-install/post-update hooks.

### 5. Использовать platform config

```json
{
    "config": {
        "platform": {
            "php": "8.1.10",
            "ext-redis": "5.3.7"
        }
    }
}
```

Фиксирует версию PHP/extensions для разрешения зависимостей.

### 6. Cache

```bash
# Очистить cache
composer clear-cache

# Путь к cache
composer config cache-dir
```

### Laravel Production Deployment

```bash
# Install
composer install --no-dev --optimize-autoloader

# Cache
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan event:cache

# Opcache должен быть включен
opcache.enable=1
opcache.validate_timestamps=0  # для production
```

---

## 🛡️ Platform Requirements

### Требования к PHP и extensions

```json
{
    "require": {
        "php": "^8.1",
        "ext-mbstring": "*",
        "ext-pdo": "*",
        "ext-json": "*",
        "ext-redis": "^5.3"
    }
}
```

**Проверить platform:**
```bash
composer check-platform-reqs
```

### Игнорировать platform requirements

```bash
# При установке (если на production другая версия PHP)
composer install --ignore-platform-reqs

# Или только определенные
composer install --ignore-platform-req=php
composer install --ignore-platform-req=ext-redis
```

---

## 📦 Создание своего пакета

### 1. Инициализация

```bash
mkdir my-package
cd my-package
composer init
```

**composer.json:**
```json
{
    "name": "vendor/my-package",
    "description": "My awesome package",
    "type": "library",
    "license": "MIT",
    "autoload": {
        "psr-4": {
            "Vendor\\MyPackage\\": "src/"
        }
    },
    "autoload-dev": {
        "psr-4": {
            "Vendor\\MyPackage\\Tests\\": "tests/"
        }
    },
    "require": {
        "php": "^8.1"
    },
    "require-dev": {
        "phpunit/phpunit": "^10.0"
    }
}
```

### 2. Структура пакета

```
my-package/
├── src/
│   └── MyClass.php
├── tests/
│   └── MyClassTest.php
├── composer.json
├── README.md
├── LICENSE
└── .gitignore
```

### 3. Публикация на Packagist

1. Создай репозиторий на GitHub
2. Зарегистрируйся на [packagist.org](https://packagist.org)
3. Submit пакет (указать URL репозитория)
4. Настрой автообновление через webhook

### 4. Versioning

```bash
git tag 1.0.0
git push --tags
```

Packagist автоматически создаст релиз.

### 5. Branch aliases (для dev-main)

```json
{
    "extra": {
        "branch-alias": {
            "dev-main": "1.x-dev"
        }
    }
}
```

Теперь можно:
```bash
composer require vendor/my-package:^1.0
```

И это установит `dev-main`.

---

## 🔧 Config Options

```json
{
    "config": {
        "optimize-autoloader": true,
        "preferred-install": "dist",
        "sort-packages": true,
        "allow-plugins": {
            "pestphp/pest-plugin": true
        },
        "platform": {
            "php": "8.1.10"
        },
        "vendor-dir": "vendor",
        "bin-dir": "vendor/bin",
        "process-timeout": 300,
        "cache-dir": "/path/to/cache",
        "discard-changes": true
    }
}
```

**Основные опции:**
- `optimize-autoloader` - авто-оптимизация при install/update
- `preferred-install` - `dist` или `source`
- `sort-packages` - сортировать require в composer.json
- `allow-plugins` - разрешить плагины (PHP 8.0+)
- `platform` - фиксация версий PHP/extensions
- `process-timeout` - таймаут команд (default 300s)

---

## 🎓 Best Practices

### 1. Всегда коммить composer.lock

```bash
git add composer.json composer.lock
git commit -m "Update dependencies"
```

### 2. Используй `^` для версий

```json
{
    "require": {
        "monolog/monolog": "^3.0"  // ✅
    }
}
```

Не:
```json
{
    "require": {
        "monolog/monolog": "*"  // ❌ опасно!
    }
}
```

### 3. require vs require-dev

```json
{
    "require": {
        "guzzle": "^7.0"  // для работы приложения
    },
    "require-dev": {
        "phpunit": "^10.0"  // только для разработки
    }
}
```

### 4. Проверяй безопасность

```bash
# Встроенный audit (Composer 2.4+)
composer audit

# Или SensioLabs Security Checker
composer require --dev sensiolabs/security-checker
vendor/bin/security-checker security:check
```

### 5. Production deployment

```bash
composer install \
    --no-dev \
    --optimize-autoloader \
    --prefer-dist \
    --no-interaction \
    --no-progress
```

---

## 🎓 Для собеседования: ключевые точки

1. **composer install vs update** - install использует lock, update обновляет зависимости
2. **composer.lock** - фиксирует точные версии для воспроизводимых builds
3. **Semver** - ^3.0 = >=3.0.0 <4.0.0, ~3.4 = >=3.4.0 <4.0.0
4. **PSR-4 autoloading** - namespace к directory маппинг
5. **require vs require-dev** - production vs development зависимости
6. **Оптимизация** - `--optimize-autoloader` для production
7. **Scripts** - автоматизация через post-install-cmd и т.д.
8. **Repositories** - VCS, path, private Packagist
9. **Создание пакета** - composer init, публикация на Packagist
10. **Security** - `composer audit` для проверки уязвимостей

**Главное:** Composer решает dependency hell, обеспечивает autoloading, упрощает управление зависимостями.

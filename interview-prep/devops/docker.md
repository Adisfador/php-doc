# Docker

Платформа для контейнеризации приложений - упаковка приложения со всеми зависимостями в изолированный контейнер.

---

## 🐳 Что такое Docker

**Docker** - платформа для разработки, доставки и запуска приложений в контейнерах.

**Контейнер** - изолированная среда выполнения с приложением и всеми его зависимостями.

### Контейнер vs Виртуальная машина

```
Virtual Machine                    Container
┌─────────────────────┐           ┌─────────────────────┐
│   App A   │  App B  │           │  App A  │  App B    │
├───────────┴─────────┤           ├───────────┴─────────┤
│   Guest OS (Linux)  │           │   Docker Engine     │
├─────────────────────┤           ├─────────────────────┤
│     Hypervisor      │           │     Host OS         │
├─────────────────────┤           └─────────────────────┘
│      Host OS        │
└─────────────────────┘

Size: ~GB                          Size: ~MB
Boot: ~minutes                     Boot: ~seconds
Resources: Heavy                   Resources: Lightweight
Isolation: Full                    Isolation: Process-level
```

**Преимущества контейнеров:**
- **Lightweight** - нет Guest OS, только процессы
- **Fast** - запуск за секунды
- **Portable** - "работает на моей машине" → работает везде
- **Consistent** - одинаковое окружение dev/staging/prod
- **Scalable** - легко масштабировать

**Когда VM лучше:**
- Нужна полная изоляция ядра
- Разные операционные системы (Windows + Linux)
- Legacy приложения

---

## 📦 Docker Архитектура

```
┌─────────────────────────────────────────────┐
│              Docker Client                  │
│         (docker CLI commands)               │
└────────────────┬────────────────────────────┘
                 │ REST API
┌────────────────▼────────────────────────────┐
│            Docker Daemon                    │
│         (dockerd - background)              │
│                                             │
│  ┌─────────────────────────────────────┐   │
│  │         Containers                  │   │
│  │  ┌──────┐  ┌──────┐  ┌──────┐      │   │
│  │  │ PHP  │  │MySQL │  │Nginx │      │   │
│  │  └──────┘  └──────┘  └──────┘      │   │
│  └─────────────────────────────────────┘   │
│                                             │
│  ┌─────────────────────────────────────┐   │
│  │            Images                   │   │
│  │  php:8.2-fpm, mysql:8.0, nginx     │   │
│  └─────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────┐
│          Docker Registry                    │
│        (Docker Hub, GitLab, etc.)           │
└─────────────────────────────────────────────┘
```

**Компоненты:**
- **Docker Client** - CLI интерфейс (`docker run`, `docker build`)
- **Docker Daemon** - фоновый процесс, управляет контейнерами
- **Docker Registry** - хранилище образов (Docker Hub)
- **Images** - read-only шаблоны для контейнеров
- **Containers** - запущенные экземпляры образов

---

## 🖼️ Docker Images

### Что такое Image

**Image** - read-only template с инструкциями для создания контейнера.

**Состоит из слоёв (layers):**
```
┌──────────────────────────┐
│  App code (Laravel)      │  Layer 5 (10 MB)
├──────────────────────────┤
│  Composer dependencies   │  Layer 4 (50 MB)
├──────────────────────────┤
│  PHP extensions          │  Layer 3 (20 MB)
├──────────────────────────┤
│  PHP 8.2                 │  Layer 2 (100 MB)
├──────────────────────────┤
│  Ubuntu base             │  Layer 1 (200 MB)
└──────────────────────────┘

Total: 380 MB
```

**Слои переиспользуются:**
```
Image A (Nginx)           Image B (PHP)
┌───────────────┐        ┌───────────────┐
│ Nginx files   │        │ PHP files     │
├───────────────┤        ├───────────────┤
│ Ubuntu base   │◄───────┤ Ubuntu base   │  (shared layer!)
└───────────────┘        └───────────────┘
```

### Основные команды

```bash
# Список образов
docker images
docker image ls

# Скачать образ
docker pull php:8.2-fpm
docker pull mysql:8.0
docker pull nginx:alpine

# Удалить образ
docker rmi php:8.2-fpm
docker image rm php:8.2-fpm

# Удалить все неиспользуемые образы
docker image prune

# Информация об образе
docker inspect php:8.2-fpm

# История слоёв
docker history php:8.2-fpm

# Теги образа
docker pull php:8.2-fpm        # specific version
docker pull php:latest         # latest
docker pull php:8.2-fpm-alpine # Alpine Linux (smaller)
```

### Создание своего образа

**Два способа:**
1. **Dockerfile** - текстовый файл с инструкциями (recommended)
2. **docker commit** - сохранить изменения контейнера (не recommended)

---

## 📝 Dockerfile

### Базовый Dockerfile для Laravel

```dockerfile
# Базовый образ
FROM php:8.2-fpm

# Метаданные
LABEL maintainer="your-email@example.com"
LABEL version="1.0"

# Рабочая директория
WORKDIR /var/www/html

# Установка системных зависимостей
RUN apt-get update && apt-get install -y \
    git \
    curl \
    libpng-dev \
    libonig-dev \
    libxml2-dev \
    zip \
    unzip \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Установка PHP расширений
RUN docker-php-ext-install \
    pdo_mysql \
    mbstring \
    exif \
    pcntl \
    bcmath \
    gd

# Установка Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Копирование composer files для кеширования слоёв
COPY composer.json composer.lock ./
RUN composer install --no-dev --no-scripts --no-autoloader --prefer-dist

# Копирование приложения
COPY . .

# Генерация autoloader
RUN composer dump-autoload --optimize

# Права доступа
RUN chown -R www-data:www-data /var/www/html \
    && chmod -R 755 /var/www/html/storage

# Expose порт
EXPOSE 9000

# Команда запуска
CMD ["php-fpm"]
```

### Dockerfile инструкции

**FROM** - базовый образ
```dockerfile
FROM php:8.2-fpm
FROM php:8.2-fpm-alpine  # Alpine - меньше размер
FROM ubuntu:22.04
```

**RUN** - выполнить команду при сборке (создаёт новый слой)
```dockerfile
RUN apt-get update && apt-get install -y git

# Multi-line (лучше для кеша)
RUN apt-get update \
    && apt-get install -y \
        git \
        curl \
        vim \
    && apt-get clean
```

**COPY** - копировать файлы из host в image
```dockerfile
COPY . /var/www/html
COPY composer.json composer.lock ./
COPY --chown=www-data:www-data . /var/www/html
```

**ADD** - как COPY, но с дополнительными возможностями
```dockerfile
ADD app.tar.gz /var/www/  # автоматически распаковывает архивы
ADD https://example.com/file.txt /tmp/  # скачивает по URL
```

**WORKDIR** - установить рабочую директорию
```dockerfile
WORKDIR /var/www/html
# Все последующие команды выполняются в этой директории
```

**ENV** - установить environment переменные
```dockerfile
ENV APP_ENV=production
ENV DB_HOST=mysql
ENV DB_PORT=3306
```

**ARG** - build-time переменные
```dockerfile
ARG PHP_VERSION=8.2
FROM php:${PHP_VERSION}-fpm

# При сборке:
# docker build --build-arg PHP_VERSION=8.1 .
```

**EXPOSE** - документирует используемые порты (не публикует!)
```dockerfile
EXPOSE 9000
EXPOSE 80 443
```

**CMD** - команда по умолчанию при запуске контейнера
```dockerfile
CMD ["php-fpm"]
CMD ["php", "artisan", "serve", "--host=0.0.0.0"]

# Shell form (запускается через /bin/sh -c)
CMD php artisan serve
```

**ENTRYPOINT** - основная команда (не переопределяется при docker run)
```dockerfile
ENTRYPOINT ["php", "artisan"]
# docker run image queue:work  → php artisan queue:work
```

**VOLUME** - создать точку монтирования
```dockerfile
VOLUME /var/www/html/storage
```

**USER** - установить пользователя для выполнения команд
```dockerfile
USER www-data
# Команды после этого выполняются от www-data
```

**LABEL** - метаданные
```dockerfile
LABEL version="1.0" \
      description="Laravel application" \
      maintainer="dev@example.com"
```

### Оптимизация Dockerfile

**1. Порядок слоёв - от редко меняющихся к часто:**
```dockerfile
# ❌ Плохо - каждое изменение кода сбрасывает кеш
FROM php:8.2-fpm
COPY . /var/www
RUN composer install

# ✅ Хорошо - composer install кешируется
FROM php:8.2-fpm
COPY composer.json composer.lock ./
RUN composer install
COPY . /var/www
```

**2. Объединять RUN команды:**
```dockerfile
# ❌ Плохо - 3 слоя
RUN apt-get update
RUN apt-get install -y git
RUN apt-get clean

# ✅ Хорошо - 1 слой
RUN apt-get update \
    && apt-get install -y git \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
```

**3. .dockerignore - исключить ненужные файлы:**
```
# .dockerignore
.git
.env
node_modules
vendor
storage/logs/*
storage/framework/cache/*
storage/framework/sessions/*
storage/framework/views/*
tests
.phpunit.result.cache
```

**4. Multi-stage builds:**
```dockerfile
# Stage 1: Build
FROM composer:latest AS builder
WORKDIR /app
COPY composer.json composer.lock ./
RUN composer install --no-dev --optimize-autoloader
COPY . .

# Stage 2: Production
FROM php:8.2-fpm-alpine
WORKDIR /var/www/html
COPY --from=builder /app /var/www/html
RUN chown -R www-data:www-data /var/www/html
CMD ["php-fpm"]
```

**Результат:**
```
Builder stage: 500 MB  (не попадает в финальный образ)
Final image:   150 MB  (только необходимое)
```

---

## 📦 Docker Containers

### Основные команды

```bash
# Запуск контейнера
docker run php:8.2-fpm

# С именем
docker run --name my-php php:8.2-fpm

# В фоне (detached)
docker run -d --name my-php php:8.2-fpm

# С портами (host:container)
docker run -d -p 8080:80 nginx

# С volume (host:container)
docker run -d -v /host/path:/container/path nginx

# С environment variables
docker run -d -e DB_HOST=mysql -e DB_PORT=3306 php:8.2-fpm

# Интерактивный режим + TTY
docker run -it ubuntu bash

# Автоудаление после остановки
docker run --rm ubuntu echo "Hello"

# Список запущенных контейнеров
docker ps

# Все контейнеры (включая остановленные)
docker ps -a

# Остановить контейнер
docker stop my-php

# Запустить остановленный
docker start my-php

# Перезапустить
docker restart my-php

# Удалить контейнер
docker rm my-php

# Удалить запущенный (force)
docker rm -f my-php

# Логи контейнера
docker logs my-php
docker logs -f my-php  # follow (tail)
docker logs --tail 100 my-php

# Выполнить команду в запущенном контейнере
docker exec my-php ls -la
docker exec -it my-php bash  # интерактивный shell

# Статистика ресурсов
docker stats

# Информация о контейнере
docker inspect my-php

# Процессы в контейнере
docker top my-php

# Копировать файлы
docker cp my-php:/var/www/html/file.txt ./
docker cp ./file.txt my-php:/var/www/html/
```

### Пример: Запуск Laravel приложения

```bash
# 1. Build образа
docker build -t my-laravel-app .

# 2. Запуск контейнера
docker run -d \
  --name laravel \
  -p 8000:8000 \
  -v $(pwd):/var/www/html \
  -e APP_ENV=local \
  -e DB_HOST=mysql \
  my-laravel-app \
  php artisan serve --host=0.0.0.0

# 3. Миграции
docker exec laravel php artisan migrate

# 4. Логи
docker logs -f laravel

# 5. Shell в контейнере
docker exec -it laravel bash
```

---

## 🗂️ Docker Volumes

### Типы хранения данных

**1. Volumes (Recommended)**
```bash
# Создать volume
docker volume create my-volume

# Список volumes
docker volume ls

# Использовать volume
docker run -d -v my-volume:/var/www/html/storage php:8.2-fpm

# Информация
docker volume inspect my-volume

# Удалить volume
docker volume rm my-volume

# Удалить неиспользуемые
docker volume prune
```

**Преимущества:**
- Управляются Docker
- Независимы от host filesystem
- Легко backup/restore
- Работают на Windows/Mac/Linux одинаково

**2. Bind Mounts**
```bash
# Монтировать директорию host в контейнер
docker run -d -v /host/path:/container/path nginx
docker run -d -v $(pwd):/var/www/html php:8.2-fpm
```

**Преимущества:**
- Прямой доступ к файлам на host
- Изменения сразу видны в контейнере (live reload)

**Use case:**
- Development - код на host, выполнение в контейнере

**3. tmpfs Mounts (только в памяти)**
```bash
docker run -d --tmpfs /tmp nginx
```

**Use case:**
- Временные файлы, не нужны на диске

### Пример: Laravel с volumes

```bash
# Volume для storage (persistent)
docker volume create laravel-storage

docker run -d \
  --name laravel \
  -v $(pwd):/var/www/html \           # код (bind mount)
  -v laravel-storage:/var/www/html/storage \  # storage (volume)
  php:8.2-fpm
```

---

## 🌐 Docker Networks

### Типы сетей

**1. Bridge (default)**
```bash
# Контейнеры в одной bridge сети могут общаться
docker network create my-network

docker run -d --name mysql --network my-network mysql:8.0
docker run -d --name php --network my-network php:8.2-fpm

# PHP может подключиться к MySQL по имени 'mysql'
DB_HOST=mysql
```

**2. Host**
```bash
# Использует сеть хоста напрямую
docker run -d --network host nginx
# Nginx доступен сразу на localhost:80
```

**3. None**
```bash
# Без сети
docker run -d --network none ubuntu
```

### Команды

```bash
# Список сетей
docker network ls

# Создать сеть
docker network create my-network

# Подключить контейнер к сети
docker network connect my-network my-container

# Отключить
docker network disconnect my-network my-container

# Информация
docker network inspect my-network

# Удалить сеть
docker network rm my-network

# Удалить неиспользуемые
docker network prune
```

---

## 🐙 Docker Compose

### Что такое Docker Compose

**Docker Compose** - инструмент для определения и запуска multi-container приложений.

**docker-compose.yml** - файл конфигурации в YAML формате.

### Пример: Laravel + MySQL + Nginx + Redis

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Nginx
  nginx:
    image: nginx:alpine
    container_name: laravel-nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./:/var/www/html
      - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - php
    networks:
      - laravel

  # PHP-FPM
  php:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: laravel-php
    volumes:
      - ./:/var/www/html
    environment:
      - APP_ENV=local
      - DB_HOST=mysql
      - DB_DATABASE=laravel
      - DB_USERNAME=laravel
      - DB_PASSWORD=secret
      - REDIS_HOST=redis
    depends_on:
      - mysql
      - redis
    networks:
      - laravel

  # MySQL
  mysql:
    image: mysql:8.0
    container_name: laravel-mysql
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: laravel
      MYSQL_USER: laravel
      MYSQL_PASSWORD: secret
    volumes:
      - mysql-data:/var/lib/mysql
    networks:
      - laravel

  # Redis
  redis:
    image: redis:alpine
    container_name: laravel-redis
    ports:
      - "6379:6379"
    networks:
      - laravel

  # Queue Worker
  queue:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: laravel-queue
    command: php artisan queue:work --tries=3
    volumes:
      - ./:/var/www/html
    depends_on:
      - php
      - redis
    networks:
      - laravel

# Volumes
volumes:
  mysql-data:
    driver: local

# Networks
networks:
  laravel:
    driver: bridge
```

### Docker Compose команды

```bash
# Запуск всех сервисов (в фоне)
docker-compose up -d

# Просмотр логов
docker-compose logs -f
docker-compose logs -f php

# Остановка всех сервисов
docker-compose down

# Остановка + удаление volumes
docker-compose down -v

# Пересборка образов
docker-compose build

# Пересборка + запуск
docker-compose up -d --build

# Список запущенных сервисов
docker-compose ps

# Выполнить команду в сервисе
docker-compose exec php php artisan migrate
docker-compose exec php composer install
docker-compose exec mysql mysql -u root -p

# Запустить одноразовую команду (новый контейнер)
docker-compose run --rm php php artisan test

# Масштабирование
docker-compose up -d --scale queue=3

# Restart сервиса
docker-compose restart php

# Просмотр конфигурации
docker-compose config
```

### Laravel с Docker Compose - Workflow

```bash
# 1. Клонировать проект
git clone https://github.com/example/laravel-app.git
cd laravel-app

# 2. Копировать .env
cp .env.example .env

# 3. Запустить контейнеры
docker-compose up -d

# 4. Установить зависимости
docker-compose exec php composer install

# 5. Генерация ключа
docker-compose exec php php artisan key:generate

# 6. Миграции
docker-compose exec php php artisan migrate --seed

# 7. Приложение доступно на http://localhost

# Development workflow:
# - Код редактируется на host (автоматически синхронизируется через volume)
# - Контейнеры выполняют код

# 8. Остановка
docker-compose down
```

---

## 🚀 Laravel Sail

**Laravel Sail** - встроенный Docker-based development environment для Laravel.

### Установка

```bash
# В новом проекте
composer require laravel/sail --dev

# Установка Sail
php artisan sail:install

# Выбрать сервисы: mysql, redis, meilisearch, mailhog, etc.

# Алиас для удобства
alias sail='./vendor/bin/sail'
```

### Использование

```bash
# Запуск
sail up
sail up -d  # detached

# Остановка
sail down

# Артисан команды
sail artisan migrate
sail artisan tinker

# Composer
sail composer install
sail composer require package

# NPM
sail npm install
sail npm run dev

# Тесты
sail test
sail test --filter=UserTest

# Shell в контейнере
sail shell
sail root-shell

# MySQL
sail mysql

# Redis
sail redis

# Логи
sail logs
```

### Sail docker-compose.yml

```yaml
# docker-compose.yml (генерируется Sail)
version: '3'
services:
    laravel.test:
        build:
            context: ./vendor/laravel/sail/runtimes/8.2
            dockerfile: Dockerfile
        image: sail-8.2/app
        ports:
            - '${APP_PORT:-80}:80'
        environment:
            WWWUSER: '${WWWUSER}'
            LARAVEL_SAIL: 1
        volumes:
            - '.:/var/www/html'
        networks:
            - sail
        depends_on:
            - mysql
            - redis
    
    mysql:
        image: 'mysql/mysql-server:8.0'
        ports:
            - '${FORWARD_DB_PORT:-3306}:3306'
        environment:
            MYSQL_ROOT_PASSWORD: '${DB_PASSWORD}'
            MYSQL_DATABASE: '${DB_DATABASE}'
        volumes:
            - 'sail-mysql:/var/lib/mysql'
        networks:
            - sail
    
    redis:
        image: 'redis:alpine'
        ports:
            - '${FORWARD_REDIS_PORT:-6379}:6379'
        volumes:
            - 'sail-redis:/data'
        networks:
            - sail

networks:
    sail:
        driver: bridge

volumes:
    sail-mysql:
        driver: local
    sail-redis:
        driver: local
```

---

## 🎯 Best Practices

### 1. Использовать официальные образы

```dockerfile
# ✅ Хорошо
FROM php:8.2-fpm-alpine

# ❌ Плохо (непроверенный образ)
FROM randomuser/php-custom
```

### 2. Минимизировать размер образа

```dockerfile
# Использовать Alpine
FROM php:8.2-fpm-alpine  # ~50 MB
# vs
FROM php:8.2-fpm         # ~400 MB

# Удалять кеши
RUN apt-get update \
    && apt-get install -y git \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Multi-stage builds
FROM composer AS builder
# ... build
FROM php:8.2-fpm-alpine
COPY --from=builder /app /var/www
```

### 3. Не запускать от root

```dockerfile
# Создать пользователя
RUN addgroup -g 1000 appuser \
    && adduser -D -u 1000 -G appuser appuser

USER appuser
```

### 4. Использовать .dockerignore

```
.git
.env
node_modules
vendor
tests
.phpunit.result.cache
storage/logs/*
```

### 5. Один процесс на контейнер

```
# ❌ Плохо - nginx + php в одном контейнере
# ✅ Хорошо - nginx в одном, php в другом
```

### 6. Health checks

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost/ || exit 1
```

```yaml
# docker-compose.yml
services:
  php:
    healthcheck:
      test: ["CMD", "php", "-v"]
      interval: 30s
      timeout: 3s
      retries: 3
```

### 7. Версионирование образов

```bash
# ❌ Плохо
docker pull php:latest

# ✅ Хорошо - конкретная версия
docker pull php:8.2.15-fpm-alpine
```

### 8. Логирование в stdout/stderr

```dockerfile
# Symfony/Laravel уже пишут в stdout
# Nginx/PHP-FPM логи в stdout/stderr
RUN ln -sf /dev/stdout /var/log/nginx/access.log \
    && ln -sf /dev/stderr /var/log/nginx/error.log
```

---

## 🔧 Troubleshooting

### Проблемы и решения

**1. Контейнер сразу останавливается**
```bash
# Проверить логи
docker logs container-name

# Проблема обычно в CMD/ENTRYPOINT
# Должен быть foreground процесс, не background
```

**2. Нет места на диске**
```bash
# Очистить всё неиспользуемое
docker system prune -a

# Показать использование диска
docker system df
```

**3. Контейнеры не видят друг друга**
```bash
# Проверить сеть
docker network inspect network-name

# Убедиться что контейнеры в одной сети
docker-compose ps
```

**4. Медленная работа на Windows/Mac**
```bash
# Проблема: bind mounts медленные
# Решение: использовать volumes или Docker Desktop настройки
```

**5. Permission denied**
```bash
# Проблема: пользователь в контейнере != пользователь на host
# Решение: установить правильный UID/GID

ARG USER_ID=1000
ARG GROUP_ID=1000
RUN groupadd -g ${GROUP_ID} appuser \
    && useradd -u ${USER_ID} -g appuser appuser

# Build с аргументами
docker build --build-arg USER_ID=$(id -u) --build-arg GROUP_ID=$(id -g) .
```

---

## 🎓 Для собеседования: ключевые точки

1. **Контейнер vs VM** - контейнер = процесс с изоляцией (namespace, cgroups), VM = полная ОС с hypervisor
2. **Dockerfile** - FROM (базовый образ), RUN (команды при сборке), COPY (файлы), CMD (команда запуска), слои кешируются
3. **Образ vs Контейнер** - образ = read-only template (класс), контейнер = запущенный экземпляр (объект)
4. **Volumes** - persistent storage, независимый от lifecycle контейнера, volumes (Docker managed) vs bind mounts (host directory)
5. **Networks** - bridge (default, контейнеры по имени), host (сеть хоста), контейнеры в одной сети общаются по имени сервиса
6. **Docker Compose** - multi-container приложения, docker-compose.yml (services, volumes, networks), `docker-compose up -d`
7. **Multi-stage builds** - уменьшение размера образа, builder stage → production stage, копировать только необходимое
8. **Layer caching** - порядок команд важен, редко меняющиеся первыми (FROM, RUN apt install), часто меняющиеся последними (COPY app)
9. **Laravel Sail** - официальный Docker environment для Laravel, `sail up`, `sail artisan`, `sail composer`
10. **Best practices** - Alpine образы (меньше размер), не root пользователь, один процесс на контейнер, .dockerignore, конкретные версии образов

**Главное:** Понимай разницу контейнер/VM, как работают слои в образах (кеширование), volumes для persistence, networks для связи между контейнерами, Docker Compose для оркестрации multi-container приложений.

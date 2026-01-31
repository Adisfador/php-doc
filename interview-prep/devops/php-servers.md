# CGI и PHP серверы

## Что такое CGI

**CGI (Common Gateway Interface)** - протокол взаимодействия веб-сервера с внешними программами.

### Как работает CGI

```
1. HTTP Request → Web Server (Nginx/Apache)
2. Web Server запускает CGI программу (новый процесс)
3. Web Server передает данные через:
   - Environment variables (QUERY_STRING, REQUEST_METHOD)
   - stdin (POST data)
4. CGI программа обрабатывает запрос
5. CGI программа выводит результат в stdout
6. Web Server отправляет ответ клиенту
7. Процесс завершается
```

### Проблемы классического CGI

❌ **Медленно:**
- Новый процесс для каждого запроса
- Загрузка интерпретатора PHP каждый раз
- Парсинг php.ini каждый раз

❌ **Ресурсоемко:**
- Много памяти (процесс на запрос)
- Высокая CPU нагрузка (fork/exec)

---

## FastCGI

**FastCGI** - улучшенная версия CGI с persistent процессами.

### Архитектура FastCGI

```
┌─────────────────────────────────────────┐
│          Web Server (Nginx)             │
└────────────┬────────────────────────────┘
             │ FastCGI Protocol
             │ (через TCP socket или Unix socket)
             ↓
┌─────────────────────────────────────────┐
│      FastCGI Process Manager            │
│  ┌─────────┬─────────┬─────────────┐    │
│  │ Worker1 │ Worker2 │   Worker3   │    │
│  │ (idle)  │ (busy)  │   (busy)    │    │
│  └─────────┴─────────┴─────────────┘    │
└─────────────────────────────────────────┘
```

### Преимущества FastCGI

✅ **Быстро:**
- Процессы живут между запросами
- PHP уже загружен
- OpCache работает эффективно

✅ **Масштабируемо:**
- Пул воркеров
- Независимо от веб-сервера
- Может быть на другой машине

### Конфигурация Nginx + FastCGI

```nginx
server {
    listen 80;
    server_name example.com;
    root /var/www/html;
    index index.php;

    location ~ \.php$ {
        # Передача запросов PHP-FPM
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;  # Unix socket
        # или
        # fastcgi_pass 127.0.0.1:9000;  # TCP socket
        
        fastcgi_index index.php;
        
        # FastCGI параметры
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
        
        # Таймауты
        fastcgi_read_timeout 300;
        fastcgi_send_timeout 300;
        
        # Буферы
        fastcgi_buffers 16 16k;
        fastcgi_buffer_size 32k;
    }
}
```

---

## PHP-FPM (FastCGI Process Manager)

**PHP-FPM** - это FastCGI реализация для PHP с расширенными возможностями управления процессами.

### Архитектура PHP-FPM

```
┌──────────────────────────────────────────┐
│         Master Process                   │
│  - Читает конфигурацию                   │
│  - Управляет worker'ами                  │
│  - Обрабатывает сигналы (reload)         │
└────────────┬─────────────────────────────┘
             │
    ┌────────┼────────┬─────────┐
    ↓        ↓        ↓         ↓
┌─────────┐ ┌─────────┐ ┌─────────┐
│ Worker1 │ │ Worker2 │ │ Worker3 │
│ PHP     │ │ PHP     │ │ PHP     │
│ процесс │ │ процесс │ │ процесс │
└─────────┘ └─────────┘ └─────────┘
```

### Process Management Modes

#### 1. Static (статический пул)

```ini
; /etc/php/8.2/fpm/pool.d/www.conf
pm = static
pm.max_children = 50
```

- Фиксированное количество воркеров
- Всегда `pm.max_children` процессов
- **Использовать:** когда нагрузка стабильная и предсказуемая

**Плюсы:**
- Предсказуемое потребление памяти
- Нет overhead на создание процессов

**Минусы:**
- Много памяти даже при низкой нагрузке

#### 2. Dynamic (динамический пул)

```ini
pm = dynamic
pm.max_children = 50        # Максимум процессов
pm.start_servers = 5        # При старте
pm.min_spare_servers = 5    # Минимум idle процессов
pm.max_spare_servers = 10   # Максимум idle процессов
```

- Количество процессов меняется
- Создаются/убиваются по мере необходимости
- **Использовать:** большинство случаев

**Пример работы:**
```
Start: 5 процессов
Нагрузка растет → создается до max_children (50)
Нагрузка падает → убиваются до min_spare_servers (5)
```

#### 3. Ondemand (по требованию)

```ini
pm = ondemand
pm.max_children = 50
pm.process_idle_timeout = 10s  # Убить процесс после 10s idle
```

- Процессы создаются при запросе
- Убиваются после idle timeout
- **Использовать:** редкие запросы, экономия памяти

**Плюсы:**
- Минимум памяти при отсутствии нагрузки

**Минусы:**
- Overhead на создание процесса для каждого burst'а

### Важные настройки

```ini
; === Пул процессов ===
[www]
user = www-data
group = www-data

; Слушать на Unix socket (быстрее) или TCP
listen = /run/php/php8.2-fpm.sock
; listen = 127.0.0.1:9000

; Права на socket
listen.owner = www-data
listen.group = www-data
listen.mode = 0660

; === Process Management ===
pm = dynamic
pm.max_children = 50
pm.start_servers = 5
pm.min_spare_servers = 5
pm.max_spare_servers = 10
pm.max_requests = 500  # Перезапуск worker'а после N запросов (против memory leaks)

; === Таймауты ===
request_terminate_timeout = 30s  # Убить запрос через 30s
request_slowlog_timeout = 5s     # Логировать медленные запросы

; === Логирование ===
slowlog = /var/log/php-fpm-slow.log
php_admin_value[error_log] = /var/log/php-fpm-error.log
php_admin_flag[log_errors] = on

; === Лимиты памяти ===
php_admin_value[memory_limit] = 256M

; === Status page ===
pm.status_path = /fpm-status
ping.path = /fpm-ping
```

### Расчет pm.max_children

**Формула:**
```
pm.max_children = (Доступная RAM) / (RAM на процесс PHP)
```

**Пример:**
- Сервер с 4GB RAM
- 1GB для ОС/MySQL/других сервисов
- 3GB доступно для PHP
- Средний PHP процесс: 50MB

```
pm.max_children = 3000 MB / 50 MB = 60 процессов
```

**Проверка памяти процесса:**
```bash
# Средняя память на процесс
ps aux | grep php-fpm | awk '{sum+=$6} END {print sum/NR/1024 " MB"}'
```

### Мониторинг PHP-FPM

#### Status Page

```nginx
location ~ ^/(fpm-status|fpm-ping)$ {
    access_log off;
    allow 127.0.0.1;
    deny all;
    fastcgi_pass unix:/run/php/php8.2-fpm.sock;
    include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
}
```

**Вывод /fpm-status:**
```
pool:                 www
process manager:      dynamic
start time:           01/Jan/2024:12:00:00 +0000
start since:          3600
accepted conn:        12543
listen queue:         0
max listen queue:     5
listen queue len:     128
idle processes:       8
active processes:     2
total processes:      10
max active processes: 15
max children reached: 0
slow requests:        3
```

#### CLI команды

```bash
# Статус пулов
systemctl status php8.2-fpm

# Список процессов
ps aux | grep php-fpm

# Reload конфигурации (graceful)
systemctl reload php8.2-fpm

# Restart
systemctl restart php8.2-fpm

# Проверка конфигурации
php-fpm8.2 -t
```

---

## mod_php (Apache Module)

**mod_php** - PHP встроен в Apache как модуль.

### Как работает

```
┌─────────────────────────────────────┐
│         Apache Process              │
│  ┌───────────────────────────────┐  │
│  │      mod_php module           │  │
│  │  (PHP интерпретатор внутри)   │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
```

### Преимущества mod_php

✅ Простая конфигурация
✅ Нет сетевого overhead

### Недостатки mod_php

❌ **Много памяти:**
- Каждый Apache процесс включает PHP
- Даже для статических файлов!

❌ **Безопасность:**
- Все скрипты выполняются от одного пользователя (www-data)

❌ **MPM prefork only:**
- Apache должен использовать prefork MPM (медленный)
- Не работает с event/worker MPM

### Конфигурация Apache + mod_php

```apache
# Загрузка модуля
LoadModule php_module modules/libphp.so

# Обработка PHP файлов
<FilesMatch \.php$>
    SetHandler application/x-httpd-php
</FilesMatch>

# PHP настройки в .htaccess
<IfModule mod_php.c>
    php_value upload_max_filesize 100M
    php_value post_max_size 100M
    php_flag display_errors Off
</IfModule>
```

---

## Сравнение подходов

| Критерий | mod_php | PHP-FPM | CGI |
|----------|---------|---------|-----|
| **Производительность** | Средняя | Высокая | Низкая |
| **Память** | Много | Оптимально | Много |
| **Безопасность** | Низкая | Высокая | Средняя |
| **Изоляция** | Нет | Пулы | Процессы |
| **Масштабируемость** | Ограничена | Отличная | Плохая |
| **Веб-сервер** | Apache | Nginx/Apache | Любой |
| **Use case** | Legacy | Production | Устарело |

---

## Nginx vs Apache

### Nginx

**Архитектура:**
- Event-driven (асинхронный)
- Один master + несколько worker'ов
- Non-blocking I/O

**Преимущества:**
- ✅ Низкое потребление памяти
- ✅ Много одновременных соединений (C10K problem)
- ✅ Отличен для статики
- ✅ Reverse proxy / Load balancer

**Недостатки:**
- ❌ Нет встроенной обработки PHP (нужен PHP-FPM)
- ❌ Конфигурация сложнее для новичков
- ❌ Динамические модули ограничены

**Когда использовать:**
- High traffic сайты
- Микросервисы
- Reverse proxy
- Modern stack

### Apache

**Архитектура:**
- Process/thread based
- MPM (Multi-Processing Module): prefork, worker, event

**Преимущества:**
- ✅ .htaccess (конфигурация на уровне директории)
- ✅ Много модулей
- ✅ mod_rewrite очень мощный
- ✅ Встроенная поддержка PHP (mod_php)

**Недостатки:**
- ❌ Больше памяти
- ❌ Меньше connections per server
- ❌ .htaccess замедляет (проверка на каждый запрос)

**Когда использовать:**
- Shared hosting
- Legacy приложения
- Нужны .htaccess

---

## Контейнеризация vs Виртуализация

### Виртуализация (VM)

```
┌─────────────────────────────────────┐
│          Host OS (Linux)            │
├─────────────────────────────────────┤
│         Hypervisor (KVM)            │
├──────────┬──────────┬───────────────┤
│   VM1    │   VM2    │     VM3       │
│ Guest OS │ Guest OS │   Guest OS    │
│  Ubuntu  │  CentOS  │   Windows     │
│  App A   │  App B   │    App C      │
└──────────┴──────────┴───────────────┘
```

**Характеристики:**
- Полная изоляция (отдельное ядро)
- Тяжелые (гигабайты, минуты на старт)
- Сильная безопасность

### Контейнеризация (Docker)

```
┌─────────────────────────────────────┐
│          Host OS (Linux)            │
├─────────────────────────────────────┤
│       Docker Engine                 │
├──────────┬──────────┬───────────────┤
│Container1│Container2│  Container3   │
│  App A   │  App B   │    App C      │
│ (libs)   │ (libs)   │   (libs)      │
└──────────┴──────────┴───────────────┘
         Общее ядро хоста
```

**Характеристики:**
- Shared kernel
- Легкие (мегабайты, секунды на старт)
- Процессная изоляция

### Сравнение

| Критерий | VM | Container |
|----------|----|-----------| 
| **Размер** | GB | MB |
| **Старт** | Минуты | Секунды |
| **Изоляция** | Полная | Процессная |
| **Производительность** | Overhead | Почти native |
| **Портативность** | Средняя | Отличная |
| **Use case** | Разные ОС, сильная изоляция | Микросервисы, CI/CD |

---

## Docker для PHP

### Dockerfile пример

```dockerfile
FROM php:8.2-fpm

# Установка расширений
RUN apt-get update && apt-get install -y \
    libpq-dev \
    libzip-dev \
    && docker-php-ext-install pdo pdo_pgsql zip opcache

# Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# PHP конфигурация
RUN mv "$PHP_INI_DIR/php.ini-production" "$PHP_INI_DIR/php.ini"

# Рабочая директория
WORKDIR /var/www/html

# Копирование приложения
COPY . .

# Установка зависимостей
RUN composer install --no-dev --optimize-autoloader

# Права
RUN chown -R www-data:www-data /var/www/html

EXPOSE 9000
CMD ["php-fpm"]
```

### docker-compose.yml

```yaml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./:/var/www/html
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - php

  php:
    build: .
    volumes:
      - ./:/var/www/html
    environment:
      - DB_HOST=postgres
      - DB_DATABASE=laravel
      - DB_USERNAME=laravel
      - DB_PASSWORD=secret

  postgres:
    image: postgres:15
    environment:
      - POSTGRES_DB=laravel
      - POSTGRES_USER=laravel
      - POSTGRES_PASSWORD=secret
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

---

## Ключевые вопросы для интервью

- В чем разница между CGI и FastCGI?
- Как работает PHP-FPM?
- Объясните разницу между pm=static, dynamic, ondemand
- Как рассчитать pm.max_children?
- В чем преимущества PHP-FPM перед mod_php?
- Nginx vs Apache: когда что использовать?
- В чем разница между контейнеризацией и виртуализацией?
- Как Docker использует ресурсы хост-системы?
- Что такое Unix socket vs TCP socket для PHP-FPM?
- Как мониторить PHP-FPM?

# Веб-серверы: Nginx и Apache

Два основных веб-сервера для PHP приложений - архитектура, отличия, конфигурация.

---

## 🌐 Nginx vs Apache

### Архитектурные отличия

**Apache - Process/Thread-based:**
```
Request → New Process/Thread
         ├─ Process 1 (Request 1)
         ├─ Process 2 (Request 2)
         ├─ Process 3 (Request 3)
         └─ ...

Prefork MPM: 1 process = 1 request
Worker MPM:  1 thread = 1 request
Event MPM:   Non-blocking (similar to Nginx)
```

**Nginx - Event-driven, Asynchronous:**
```
Master Process
  └─ Worker Processes (fixed number)
      ├─ Worker 1 → handles 1000s requests
      ├─ Worker 2 → handles 1000s requests
      └─ Worker 3 → handles 1000s requests

Event loop: non-blocking I/O
```

### Сравнение

| Feature | Nginx | Apache |
|---------|-------|--------|
| **Архитектура** | Event-driven | Process/Thread |
| **Concurrency** | 10,000+ connections | 100-300 connections |
| **RAM usage** | Low (~2-4 MB/worker) | High (~10-20 MB/process) |
| **Static files** | ⚡ Excellent | Good |
| **Dynamic content** | Via FastCGI | Built-in (mod_php) |
| **Configuration** | Simple, centralized | .htaccess (per-directory) |
| **Modules** | Compiled-in | Dynamic loading |
| **URL rewriting** | rewrite module | mod_rewrite (.htaccess) |
| **Market share** | ~34% | ~31% |
| **Performance** | High load | Low/medium load |

### Когда использовать

**Nginx:**
- ✅ High traffic (10,000+ concurrent)
- ✅ Статические файлы (assets, images)
- ✅ Reverse proxy
- ✅ Load balancing
- ✅ Низкое потребление RAM

**Apache:**
- ✅ Shared hosting (.htaccess)
- ✅ Динамическая конфигурация
- ✅ Legacy applications
- ✅ mod_php (встроенная поддержка PHP)

**Nginx + Apache (лучшее из двух):**
```
Internet → Nginx (reverse proxy, static)
              ↓
           Apache (dynamic PHP via mod_php)
```

---

## 🔷 Nginx

### Архитектура Nginx

```
┌─────────────────────────────────────┐
│        Master Process               │
│   (root, reads config, manages)     │
└────────┬────────────────────────────┘
         │
    ┌────┴────┬────────┬────────┐
    ▼         ▼        ▼        ▼
┌────────┐┌────────┐┌────────┐┌────────┐
│Worker 1││Worker 2││Worker 3││Worker 4│
│  (non- ││  (non- ││  (non- ││  (non- │
│ blocking││blocking││blocking││blocking│
│   I/O) ││  I/O)  ││  I/O)  ││  I/O)  │
└────────┘└────────┘└────────┘└────────┘

Workers = CPU cores (обычно)
```

**Процесс обработки запроса:**
```
1. Client connects
2. Worker принимает connection (event loop)
3. Если static file → читает и отдаёт
4. Если PHP → передаёт в PHP-FPM (FastCGI)
5. PHP-FPM обрабатывает
6. Nginx получает ответ
7. Nginx отдаёт клиенту
8. Worker готов к следующему запросу (не блокируется!)
```

### Базовая конфигурация

```nginx
# /etc/nginx/nginx.conf
user www-data;
worker_processes auto;  # = CPU cores
pid /run/nginx.pid;

events {
    worker_connections 1024;  # max connections per worker
    use epoll;  # efficient event model (Linux)
}

http {
    # Basic settings
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    
    # MIME types
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    # Logging
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;
    
    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml text/javascript 
               application/json application/javascript application/xml+rss;
    
    # Virtual hosts
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

### Server Block (Virtual Host)

```nginx
# /etc/nginx/sites-available/example.com
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;
    
    # Redirect to HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name example.com www.example.com;
    
    root /var/www/example.com/public;
    index index.php index.html;
    
    # SSL
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    
    # Security headers
    add_header Strict-Transport-Security "max-age=31536000" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    
    # Logging
    access_log /var/log/nginx/example.com-access.log;
    error_log /var/log/nginx/example.com-error.log;
    
    # Index fallback
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }
    
    # PHP-FPM
    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
        
        # Security
        fastcgi_hide_header X-Powered-By;
        fastcgi_read_timeout 300;
    }
    
    # Deny access to hidden files
    location ~ /\. {
        deny all;
    }
    
    # Static files caching
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2|ttf|eot)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
    
    # Deny access to sensitive files
    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

### Laravel конфигурация

```nginx
# /etc/nginx/sites-available/laravel.conf
server {
    listen 80;
    server_name laravel.test;
    root /var/www/laravel/public;
    
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";
    
    index index.php;
    
    charset utf-8;
    
    # Laravel routes
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }
    
    # PHP-FPM
    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }
    
    # Deny access to .htaccess
    location ~ /\.ht {
        deny all;
    }
    
    # Deny access to storage and vendor
    location ~ ^/(storage|vendor) {
        deny all;
        return 404;
    }
}
```

### Location директивы

**Приоритет:**
```nginx
# 1. Exact match (highest priority)
location = /exact/path {
    # Только /exact/path
}

# 2. Regex case-sensitive
location ~ \.php$ {
    # Заканчивается на .php
}

# 3. Regex case-insensitive
location ~* \.(jpg|png|gif)$ {
    # .JPG, .jpg, .PNG, .png
}

# 4. Prefix match (longest wins)
location ^~ /images/ {
    # Начинается с /images/
    # Останавливает regex проверку
}

# 5. Prefix match
location /api/ {
    # Начинается с /api/
}

# 6. Default
location / {
    # Всё остальное
}
```

**Примеры:**
```nginx
# API endpoint
location /api/ {
    proxy_pass http://backend:8000;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
}

# WebSocket
location /ws {
    proxy_pass http://websocket:6001;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";
}

# Static files (long cache)
location ~* \.(css|js|jpg|png|gif|ico)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
    access_log off;
}

# Disable access to git files
location ~ /\.git {
    deny all;
    return 404;
}
```

### Reverse Proxy

```nginx
upstream backend {
    server 127.0.0.1:8000;
    server 127.0.0.1:8001;
    server 127.0.0.1:8002;
}

server {
    listen 80;
    server_name example.com;
    
    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Buffering
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 24 4k;
        
        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

### Load Balancing

```nginx
# Round-robin (default)
upstream backend {
    server backend1.example.com;
    server backend2.example.com;
    server backend3.example.com;
}

# Least connections
upstream backend {
    least_conn;
    server backend1.example.com;
    server backend2.example.com;
}

# IP hash (sticky sessions)
upstream backend {
    ip_hash;
    server backend1.example.com;
    server backend2.example.com;
}

# Weighted
upstream backend {
    server backend1.example.com weight=3;  # 3x more requests
    server backend2.example.com weight=1;
}

# With health checks
upstream backend {
    server backend1.example.com max_fails=3 fail_timeout=30s;
    server backend2.example.com max_fails=3 fail_timeout=30s;
    server backend3.example.com backup;  # backup server
}
```

### Rate Limiting

```nginx
# Define rate limit zone (10 MB, 10 req/sec per IP)
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

server {
    location /api/ {
        limit_req zone=api_limit burst=20 nodelay;
        # burst=20: allow 20 extra requests
        # nodelay: don't delay requests
        
        proxy_pass http://backend;
    }
}
```

### Nginx команды

```bash
# Проверка конфигурации
sudo nginx -t

# Перезагрузка конфигурации (без downtime)
sudo nginx -s reload

# Остановка
sudo nginx -s stop      # быстрая
sudo nginx -s quit      # graceful

# Запуск
sudo systemctl start nginx

# Рестарт
sudo systemctl restart nginx

# Статус
sudo systemctl status nginx

# Логи
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log

# Версия
nginx -v
nginx -V  # с опциями компиляции
```

---

## 🔶 Apache

### Архитектура Apache

**MPM (Multi-Processing Modules):**

**1. Prefork MPM (traditional):**
```
Master Process
  ├─ Child Process 1 (1 request)
  ├─ Child Process 2 (1 request)
  ├─ Child Process 3 (1 request)
  └─ ...

Каждый процесс = отдельный request
Нет разделения памяти
Безопасно для non-thread-safe библиотек
```

**2. Worker MPM:**
```
Master Process
  ├─ Child Process 1
  │   ├─ Thread 1 (request 1)
  │   ├─ Thread 2 (request 2)
  │   └─ Thread 3 (request 3)
  ├─ Child Process 2
  │   ├─ Thread 1
  │   └─ ...
```

**3. Event MPM (recommended):**
```
Similar to Worker but with:
- Dedicated threads для keep-alive connections
- Non-blocking I/O для static files
- Лучше performance
```

### Базовая конфигурация

```apache
# /etc/apache2/apache2.conf (Debian/Ubuntu)
# /etc/httpd/conf/httpd.conf (CentOS/RHEL)

# ServerRoot
ServerRoot "/etc/apache2"

# Listen
Listen 80

# User/Group
User www-data
Group www-data

# Server admin
ServerAdmin admin@example.com

# DocumentRoot (default)
DocumentRoot "/var/www/html"

# Directory permissions
<Directory />
    Options FollowSymLinks
    AllowOverride None
    Require all denied
</Directory>

<Directory /var/www/>
    Options Indexes FollowSymLinks
    AllowOverride All
    Require all granted
</Directory>

# Logging
ErrorLog ${APACHE_LOG_DIR}/error.log
CustomLog ${APACHE_LOG_DIR}/access.log combined

# Include
IncludeOptional mods-enabled/*.load
IncludeOptional mods-enabled/*.conf
IncludeOptional conf-enabled/*.conf
IncludeOptional sites-enabled/*.conf
```

### Virtual Host

```apache
# /etc/apache2/sites-available/example.com.conf

<VirtualHost *:80>
    ServerName example.com
    ServerAlias www.example.com
    ServerAdmin admin@example.com
    
    DocumentRoot /var/www/example.com/public
    
    <Directory /var/www/example.com/public>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
    
    # Logging
    ErrorLog ${APACHE_LOG_DIR}/example.com-error.log
    CustomLog ${APACHE_LOG_DIR}/example.com-access.log combined
</VirtualHost>

<VirtualHost *:443>
    ServerName example.com
    ServerAlias www.example.com
    
    DocumentRoot /var/www/example.com/public
    
    # SSL
    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/example.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/example.com/privkey.pem
    
    # SSL protocols
    SSLProtocol all -SSLv2 -SSLv3 -TLSv1 -TLSv1.1
    SSLCipherSuite HIGH:!aNULL:!MD5
    SSLHonorCipherOrder on
    
    <Directory /var/www/example.com/public>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
    
    ErrorLog ${APACHE_LOG_DIR}/example.com-ssl-error.log
    CustomLog ${APACHE_LOG_DIR}/example.com-ssl-access.log combined
</VirtualHost>
```

### Laravel .htaccess

```apache
# /var/www/laravel/public/.htaccess

<IfModule mod_rewrite.c>
    <IfModule mod_negotiation.c>
        Options -MultiViews -Indexes
    </IfModule>

    RewriteEngine On
    
    # Handle Authorization Header
    RewriteCond %{HTTP:Authorization} .
    RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]
    
    # Redirect Trailing Slashes If Not A Folder...
    RewriteCond %{REQUEST_FILENAME} !-d
    RewriteCond %{REQUEST_URI} (.+)/$
    RewriteRule ^ %1 [L,R=301]
    
    # Send Requests To Front Controller...
    RewriteCond %{REQUEST_FILENAME} !-d
    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteRule ^ index.php [L]
</IfModule>

# Security
<FilesMatch "^\.">
    Require all denied
</FilesMatch>

# Disable directory browsing
Options -Indexes

# PHP settings (если используется mod_php)
<IfModule mod_php7.c>
    php_value upload_max_filesize 20M
    php_value post_max_size 20M
    php_value max_execution_time 300
    php_value max_input_time 300
</IfModule>
```

### mod_rewrite примеры

```apache
# Redirect HTTP to HTTPS
RewriteEngine On
RewriteCond %{HTTPS} off
RewriteRule ^(.*)$ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]

# Redirect www to non-www
RewriteCond %{HTTP_HOST} ^www\.(.+)$ [NC]
RewriteRule ^(.*)$ https://%1/$1 [R=301,L]

# Remove trailing slash
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule ^(.*)/$ /$1 [L,R=301]

# Force lowercase URLs
RewriteMap lowercase int:tolower
RewriteCond %{REQUEST_URI} [A-Z]
RewriteRule ^(.*)$ ${lowercase:$1} [R=301,L]

# Block access to files
RewriteRule ^(\.env|composer\.json|composer\.lock)$ - [F,L]

# Maintenance mode
RewriteCond %{REQUEST_URI} !^/maintenance\.html$
RewriteCond %{REMOTE_ADDR} !^123\.456\.789\.0$
RewriteRule ^.*$ /maintenance.html [R=503,L]
```

### Apache modules

```bash
# Включить модуль
sudo a2enmod rewrite
sudo a2enmod ssl
sudo a2enmod headers
sudo a2enmod expires

# Отключить модуль
sudo a2dismod php7.4
sudo a2dismod mpm_prefork

# Список включенных модулей
apache2ctl -M

# Перезагрузка после изменения модулей
sudo systemctl reload apache2
```

**Важные модули для Laravel:**
```bash
sudo a2enmod rewrite     # URL rewriting
sudo a2enmod headers     # HTTP headers
sudo a2enmod expires     # Expiration headers
sudo a2enmod deflate     # Compression
sudo a2enmod ssl         # HTTPS
sudo a2enmod proxy       # Reverse proxy
sudo a2enmod proxy_fcgi  # PHP-FPM
```

### Apache + PHP-FPM (рекомендуется)

```apache
# Отключить mod_php, использовать PHP-FPM
sudo a2dismod php8.2
sudo a2dismod mpm_prefork
sudo a2enmod mpm_event
sudo a2enmod proxy_fcgi setenvif

# /etc/apache2/sites-available/example.com.conf
<VirtualHost *:80>
    ServerName example.com
    DocumentRoot /var/www/example.com/public
    
    <Directory /var/www/example.com/public>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
    
    # PHP-FPM via proxy
    <FilesMatch \.php$>
        SetHandler "proxy:unix:/var/run/php/php8.2-fpm.sock|fcgi://localhost"
    </FilesMatch>
    
    # Or via proxy_fcgi
    ProxyPassMatch ^/(.*\.php(/.*)?)$ unix:/var/run/php/php8.2-fpm.sock|fcgi://localhost/var/www/example.com/public/
</VirtualHost>
```

### Apache команды

```bash
# Проверка конфигурации
sudo apache2ctl configtest
sudo apachectl -t

# Рестарт
sudo systemctl restart apache2

# Graceful reload (без разрыва соединений)
sudo systemctl reload apache2
sudo apache2ctl graceful

# Статус
sudo systemctl status apache2

# Включить/отключить сайт
sudo a2ensite example.com.conf
sudo a2dissite example.com.conf

# Логи
sudo tail -f /var/log/apache2/access.log
sudo tail -f /var/log/apache2/error.log

# Версия и модули
apache2 -v
apache2ctl -M  # loaded modules
apache2ctl -V  # version and compile settings
```

---

## 🔧 PHP-FPM (FastCGI Process Manager)

### Что такое PHP-FPM

**PHP-FPM** - альтернативная реализация PHP FastCGI с расширенными возможностями.

**Преимущества:**
- Отдельные процессы для PHP (изоляция от веб-сервера)
- Pool конфигурации (разные настройки для разных сайтов)
- Adaptive process spawning
- Graceful restart
- Slow log (медленные запросы)
- Status page

### Архитектура

```
Nginx/Apache
      │
      │ FastCGI Protocol (port 9000 or socket)
      ▼
PHP-FPM Master
      ├─ Pool "www"
      │   ├─ Worker 1 (idle)
      │   ├─ Worker 2 (busy - executing PHP)
      │   ├─ Worker 3 (busy)
      │   └─ ...
      └─ Pool "admin"
          ├─ Worker 1
          └─ ...
```

### Конфигурация пула

```ini
; /etc/php/8.2/fpm/pool.d/www.conf

[www]
; User/Group
user = www-data
group = www-data

; Listen (socket or TCP)
listen = /var/run/php/php8.2-fpm.sock
; или
; listen = 127.0.0.1:9000

; Listen permissions
listen.owner = www-data
listen.group = www-data
listen.mode = 0660

; Process manager (dynamic, static, ondemand)
pm = dynamic

; Max children (max workers)
pm.max_children = 50

; Start servers (при запуске)
pm.start_servers = 5

; Min spare servers (idle workers minimum)
pm.min_spare_servers = 5

; Max spare servers (idle workers maximum)
pm.max_spare_servers = 35

; Max requests per worker (memory leak prevention)
pm.max_requests = 500

; Status page
pm.status_path = /php-fpm-status

; Ping page
ping.path = /php-fpm-ping

; Slow log
slowlog = /var/log/php-fpm/www-slow.log
request_slowlog_timeout = 5s

; Timeouts
request_terminate_timeout = 300s

; PHP ini values
php_admin_value[error_log] = /var/log/php-fpm/www-error.log
php_admin_flag[log_errors] = on
php_value[upload_max_filesize] = 20M
php_value[post_max_size] = 20M
php_value[max_execution_time] = 300
php_value[max_input_time] = 300
php_value[memory_limit] = 256M
```

### Process Manager типы

**1. static:**
```ini
pm = static
pm.max_children = 10

Всегда 10 workers (фиксированное число)
✅ Предсказуемое использование памяти
❌ Не адаптируется к нагрузке
Use: Production с постоянной нагрузкой
```

**2. dynamic (recommended):**
```ini
pm = dynamic
pm.max_children = 50
pm.start_servers = 5
pm.min_spare_servers = 5
pm.max_spare_servers = 35

Динамически создаёт/убивает workers
✅ Адаптируется к нагрузке
✅ Экономит память при низкой нагрузке
Use: Production с переменной нагрузкой
```

**3. ondemand:**
```ini
pm = ondemand
pm.max_children = 50
pm.process_idle_timeout = 10s

Создаёт workers только при запросах
Убивает idle workers через 10s
✅ Минимальное использование памяти
❌ Latency при создании процесса
Use: Development, низконагруженные сайты
```

### Расчёт max_children

```bash
# Формула:
pm.max_children = (Available RAM - OS - Other) / PHP process RAM

# Пример:
# Server RAM: 4 GB
# OS + Other: 1 GB
# Available: 3 GB = 3072 MB
# PHP process: ~60 MB (зависит от приложения)

pm.max_children = 3072 / 60 ≈ 50

# Проверка реального использования памяти:
ps -ylC php-fpm8.2 --sort:rss
# RSS column = memory per process
```

### Multiple pools (разные сайты)

```ini
; /etc/php/8.2/fpm/pool.d/site1.conf
[site1]
user = site1
group = site1
listen = /var/run/php/site1.sock
pm = dynamic
pm.max_children = 20
php_value[upload_max_filesize] = 50M

; /etc/php/8.2/fpm/pool.d/site2.conf
[site2]
user = site2
group = site2
listen = /var/run/php/site2.sock
pm = dynamic
pm.max_children = 10
php_value[upload_max_filesize] = 10M
```

**Nginx конфигурация:**
```nginx
# Site 1
server {
    server_name site1.com;
    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/site1.sock;
        # ...
    }
}

# Site 2
server {
    server_name site2.com;
    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/site2.sock;
        # ...
    }
}
```

### PHP-FPM команды

```bash
# Рестарт
sudo systemctl restart php8.2-fpm

# Graceful reload (без разрыва соединений)
sudo systemctl reload php8.2-fpm

# Статус
sudo systemctl status php8.2-fpm

# Проверка конфигурации
sudo php-fpm8.2 -t

# Логи
sudo tail -f /var/log/php8.2-fpm.log

# Количество workers (для pool www)
ps aux | grep 'php-fpm: pool www' | wc -l

# Status page (если настроен)
# Nginx:
location ~ ^/(php-fpm-status|php-fpm-ping)$ {
    fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    include fastcgi_params;
}

curl http://localhost/php-fpm-status
curl http://localhost/php-fpm-ping
```

---

## 🎓 Для собеседования: ключевые точки

1. **Nginx vs Apache** - Nginx event-driven (10k+ connections, low RAM) vs Apache process/thread-based (.htaccess, mod_php)
2. **Nginx архитектура** - Master process + Worker processes (async, non-blocking I/O), один worker = тысячи connections
3. **Apache MPM** - Prefork (1 process = 1 request), Worker (threads), Event (non-blocking, recommended)
4. **Reverse Proxy** - Nginx перед Apache/App server, Nginx обрабатывает static, проксирует dynamic в backend
5. **Load Balancing** - upstream директива в Nginx, методы (round-robin, least_conn, ip_hash), health checks (max_fails)
6. **PHP-FPM** - отдельные процессы для PHP, pools с разными настройками, process manager (static/dynamic/ondemand)
7. **pm.max_children** - расчёт: (Available RAM) / (PHP process RAM), важно не превысить память сервера
8. **.htaccess** - Apache per-directory конфигурация, mod_rewrite для URL rewriting, AllowOverride All в Virtual Host
9. **FastCGI** - протокол между веб-сервером и PHP-FPM, TCP (127.0.0.1:9000) или Unix socket (faster)
10. **Nginx location** - приоритет: exact (=) > regex (~) > prefix (^~) > prefix, try_files для Laravel routing

**Главное:** Понимай архитектурные отличия Nginx (event-driven) vs Apache (process-based), когда использовать каждый, Nginx + PHP-FPM setup (FastCGI), load balancing для масштабирования, process manager типы в PHP-FPM и расчёт max_children.

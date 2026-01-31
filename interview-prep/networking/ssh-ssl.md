# SSH и SSL/TLS

Криптографические протоколы для безопасного удалённого доступа и передачи данных.

---

## 🔐 SSH (Secure Shell)

### Что такое SSH

**SSH** - криптографический сетевой протокол для безопасного удалённого управления и передачи файлов.

**Использование:**
- Удалённое управление серверами
- Передача файлов (SCP, SFTP)
- Туннелирование (port forwarding)
- Git операции

**Порт:** 22 (по умолчанию)

### SSH vs Telnet

| Feature | SSH | Telnet |
|---------|-----|--------|
| Encryption | ✅ Encrypted | ❌ Plaintext |
| Authentication | Keys + Password | Password only |
| Port | 22 | 23 |
| Security | ✅ Secure | ❌ Insecure |
| Use today | ✅ Standard | ❌ Deprecated |

---

## 🔑 SSH Authentication

### Password Authentication

**Простая но менее безопасная.**

```bash
ssh user@example.com
# Password: *******
```

**Проблемы:**
- Brute force атаки
- Перехват пароля
- Сложно автоматизировать

### Public Key Authentication (Recommended)

**Использует пару ключей: публичный + приватный.**

```
Client                          Server
  |                               |
  | Private Key                   | Public Key
  | (id_rsa)                      | (~/.ssh/authorized_keys)
  |                               |
  |------- Challenge ------------>|
  |                               | Encrypts with Public Key
  |<--- Encrypted Challenge ------|
  | Decrypts with Private Key     |
  |------ Proof of Identity ----->|
  |                               |
  |<======= Authenticated ========|
```

---

## 🔧 SSH Key Generation

### Генерация ключей

**RSA (2048-4096 bits):**
```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
# Generates:
#   ~/.ssh/id_rsa      (private key)
#   ~/.ssh/id_rsa.pub  (public key)
```

**Ed25519 (Recommended, faster and more secure):**
```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
# Generates:
#   ~/.ssh/id_ed25519
#   ~/.ssh/id_ed25519.pub
```

**With passphrase (extra security):**
```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
# Enter passphrase: *******
# Confirm passphrase: *******
```

### Копирование публичного ключа на сервер

**Способ 1: ssh-copy-id (easiest):**
```bash
ssh-copy-id user@example.com
# Автоматически добавляет ключ в ~/.ssh/authorized_keys
```

**Способ 2: Вручную:**
```bash
# На локальной машине
cat ~/.ssh/id_ed25519.pub

# Скопировать вывод, затем на сервере:
mkdir -p ~/.ssh
chmod 700 ~/.ssh
echo "публичный_ключ" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

**Способ 3: Одной командой:**
```bash
cat ~/.ssh/id_ed25519.pub | ssh user@example.com "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

### Проверка подключения

```bash
ssh -T git@github.com
# Hi username! You've successfully authenticated...

ssh user@example.com
# Should connect without password
```

---

## 🛠️ SSH Config

### ~/.ssh/config

**Удобные алиасы для серверов.**

```bash
# ~/.ssh/config

Host production
    HostName prod.example.com
    User deployer
    Port 22
    IdentityFile ~/.ssh/id_ed25519_production
    
Host staging
    HostName staging.example.com
    User deployer
    Port 2222
    IdentityFile ~/.ssh/id_ed25519_staging
    
Host github
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_github
    
# Wildcard для всех серверов example.com
Host *.example.com
    User admin
    Port 22
    ForwardAgent yes
    
# Defaults для всех
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
    Compression yes
```

**Использование:**
```bash
# Вместо
ssh deployer@prod.example.com -i ~/.ssh/id_ed25519_production

# Просто
ssh production
```

---

## 🚇 SSH Tunneling (Port Forwarding)

### Local Port Forwarding

**Перенаправить локальный порт на удалённый.**

```bash
ssh -L local_port:destination:destination_port user@ssh_server

# Example: Доступ к MySQL на удалённом сервере
ssh -L 3307:localhost:3306 user@db-server.com

# Теперь localhost:3307 → db-server.com:3306
mysql -h 127.0.0.1 -P 3307 -u root -p
```

**Use cases:**
- Доступ к БД на продакшене
- Доступ к админкам (закрыты файрволлом)
- Обход файрвола

**Схема:**
```
Your Computer              SSH Server               Database
    |                          |                        |
    |--- SSH Tunnel ---------->|                        |
    |                          |--- MySQL ------------->|
    | localhost:3307           |    remote:3306         |
```

### Remote Port Forwarding

**Перенаправить удалённый порт на локальный.**

```bash
ssh -R remote_port:localhost:local_port user@ssh_server

# Example: Показать локальный сервер (localhost:8000) публично
ssh -R 8080:localhost:8000 user@public-server.com

# Теперь public-server.com:8080 → localhost:8000
```

**Use cases:**
- Демонстрация локальной разработки клиенту
- Webhook testing (GitHub webhooks → localhost)
- Обход NAT

### Dynamic Port Forwarding (SOCKS Proxy)

**SSH как SOCKS прокси.**

```bash
ssh -D 1080 user@ssh-server.com

# Настроить браузер на SOCKS proxy localhost:1080
# Весь трафик идёт через ssh-server.com
```

**Use cases:**
- Обход geo-blocking
- Безопасный интернет через публичный WiFi
- Обход корпоративного файрвола

---

## 🔐 SSH Agent

### ssh-agent

**Хранит приватные ключи в памяти, чтобы не вводить passphrase каждый раз.**

```bash
# Запуск агента
eval "$(ssh-agent -s)"

# Добавить ключ
ssh-add ~/.ssh/id_ed25519
# Enter passphrase: *******

# Список ключей
ssh-add -l

# Удалить все ключи
ssh-add -D

# Удалить агента
ssh-agent -k
```

**macOS - автозапуск:**
```bash
# ~/.ssh/config
Host *
  AddKeysToAgent yes
  UseKeychain yes
  IdentityFile ~/.ssh/id_ed25519
```

### Agent Forwarding

**Использовать локальный SSH agent на удалённом сервере.**

```bash
ssh -A user@server.com
# or in ~/.ssh/config:
# ForwardAgent yes
```

**Use case:**
```
Your Computer → Server A → Server B → Git

# Без forwarding: нужно копировать ключи на Server A
# С forwarding: использует ваш локальный ключ
```

**⚠️ Security risk:**
- Админ на Server A может использовать ваш ключ
- Используй только на доверенных серверах

---

## 🛡️ SSH Security Best Practices

### Отключить password authentication

```bash
# /etc/ssh/sshd_config
PasswordAuthentication no
PubkeyAuthentication yes
ChallengeResponseAuthentication no
```

```bash
sudo systemctl restart sshd
```

### Отключить root login

```bash
# /etc/ssh/sshd_config
PermitRootLogin no
```

**Вместо root используй sudo:**
```bash
ssh user@server.com
sudo command
```

### Сменить порт (security through obscurity)

```bash
# /etc/ssh/sshd_config
Port 2222
```

```bash
ssh -p 2222 user@server.com
```

### Ограничить пользователей

```bash
# /etc/ssh/sshd_config
AllowUsers deployer admin
# or
AllowGroups ssh-users
```

### Fail2ban

**Блокировать IP после нескольких неудачных попыток.**

```bash
sudo apt install fail2ban

# /etc/fail2ban/jail.local
[sshd]
enabled = true
port = 22
logpath = /var/log/auth.log
maxretry = 3
bantime = 3600
findtime = 600
```

---

## 🚀 Laravel Deployment с SSH

### Laravel Forge

**Автоматическое управление SSH ключами.**

```bash
# Forge автоматически:
# 1. Генерирует SSH ключ для сервера
# 2. Добавляет публичный ключ в GitHub/GitLab/Bitbucket
# 3. Настраивает deployment через git pull
```

### Laravel Envoyer

**Zero-downtime deployment.**

```bash
# Envoyer использует SSH для:
# 1. Git clone в новую директорию
# 2. Composer install
# 3. php artisan migrate
# 4. Атомарная замена симлинка
# 5. php artisan queue:restart
```

**Deployment script example:**
```bash
cd /home/forge/example.com/releases/{{ release }}

# Composer
composer install --no-interaction --prefer-dist --optimize-autoloader

# Environment
ln -nfs /home/forge/example.com/.env .env

# Optimize
php artisan config:cache
php artisan route:cache
php artisan view:cache

# Migrate
php artisan migrate --force

# Symlink
ln -nfs /home/forge/example.com/releases/{{ release }} /home/forge/example.com/current

# Reload
php artisan queue:restart
```

### Manual Deployment Script

```bash
#!/bin/bash
# deploy.sh

SERVER="user@example.com"
REMOTE_PATH="/var/www/html"

# SSH и выполнение команд
ssh $SERVER << 'EOF'
cd $REMOTE_PATH
git pull origin main
composer install --no-dev --optimize-autoloader
php artisan migrate --force
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan queue:restart
sudo systemctl reload php8.2-fpm
EOF

echo "Deployment complete!"
```

```bash
chmod +x deploy.sh
./deploy.sh
```

### SCP (Secure Copy)

```bash
# Копировать файл на сервер
scp local_file.txt user@server.com:/remote/path/

# Копировать директорию
scp -r local_folder user@server.com:/remote/path/

# Копировать с сервера
scp user@server.com:/remote/file.txt ./local_path/

# С SSH config alias
scp local_file.txt production:/var/www/html/
```

### SFTP (SSH File Transfer Protocol)

```bash
sftp user@server.com

# Commands:
put local_file.txt          # Upload
get remote_file.txt         # Download
ls                          # List remote
lls                         # List local
cd /remote/path            # Change remote dir
lcd /local/path            # Change local dir
mkdir dirname              # Create remote dir
rm filename                # Delete remote file
bye                        # Exit
```

---

## 🔒 SSL/TLS

### Что такое SSL/TLS

**SSL (Secure Sockets Layer)** - устаревший, заменён на **TLS (Transport Layer Security)**.

**TLS** - криптографический протокол для безопасной передачи данных через интернет.

**Версии:**
- SSL 1.0, 2.0, 3.0 - ❌ Deprecated (уязвимы)
- TLS 1.0, 1.1 - ❌ Deprecated
- TLS 1.2 - ✅ Secure
- TLS 1.3 - ✅ Latest (faster, more secure)

**Использование:**
- HTTPS (HTTP + TLS)
- SMTP, IMAP (email)
- VPN
- WebSockets (WSS)

---

## 🤝 TLS Handshake

### TLS 1.2 Handshake

```
Client                                  Server
  |                                       |
  |------- ClientHello ----------------->|
  |   - TLS version                       |
  |   - Cipher suites                     |
  |   - Random bytes                      |
  |                                       |
  |<------ ServerHello -----------------|
  |   - Selected cipher                   |
  |   - Certificate (public key)          |
  |   - Server random bytes               |
  |                                       |
  |--- Certificate Verify -------------->|
  |   (verify server certificate)         |
  |                                       |
  |--- ClientKeyExchange ---------------->|
  |   (pre-master secret encrypted        |
  |    with server's public key)          |
  |                                       |
  | Both derive session keys from:        |
  | - Client random                       |
  | - Server random                       |
  | - Pre-master secret                   |
  |                                       |
  |--- ChangeCipherSpec ---------------->|
  |--- Finished (encrypted) ------------>|
  |                                       |
  |<-- ChangeCipherSpec -----------------|
  |<-- Finished (encrypted) -------------|
  |                                       |
  |<====== Encrypted Application Data ===|
```

### TLS 1.3 Handshake (Faster)

**Улучшения TLS 1.3:**
- **1-RTT** вместо 2-RTT (быстрее)
- **0-RTT** для повторных соединений
- Меньше cipher suites (только безопасные)
- Forward secrecy по умолчанию

```
Client                          Server
  |                               |
  |--- ClientHello -------------->|
  |   + KeyShare                  |
  |   (публичный ключ сразу)      |
  |                               |
  |<-- ServerHello ---------------|
  |    + KeyShare                 |
  |    + Certificate              |
  |    + Finished                 |
  | (всё в одном round-trip!)     |
  |                               |
  |--- Finished ----------------->|
  |                               |
  |<== Encrypted Data ===========>|
```

**1-RTT vs 2-RTT:**
```
TLS 1.2:  ~100ms (2 round trips)
TLS 1.3:  ~50ms  (1 round trip)
```

---

## 🎫 SSL/TLS Certificates

### Certificate Components

**X.509 Certificate содержит:**
```
- Subject (CN: example.com)
- Issuer (CA: Let's Encrypt)
- Public Key
- Validity (Not Before / Not After)
- Serial Number
- Signature Algorithm
- Extensions (SAN - Subject Alternative Names)
```

### Certificate Chain

```
Root CA (Self-signed, trusted by browsers)
    ↓
Intermediate CA (Signed by Root)
    ↓
Server Certificate (Signed by Intermediate)
    → example.com
```

**Браузер проверяет:**
1. Server Certificate подписан Intermediate CA
2. Intermediate CA подписан Root CA
3. Root CA в списке доверенных (browser trust store)
4. Сертификат не истёк
5. Домен совпадает с CN или SAN

### Certificate Types

**1. Domain Validation (DV) - базовый:**
```
- Проверка: только владение доменом
- Стоимость: $0-50/year (Let's Encrypt бесплатно)
- Время: минуты
- Use: большинство сайтов
```

**2. Organization Validation (OV):**
```
- Проверка: домен + организация
- Стоимость: $50-200/year
- Время: дни
- Use: корпоративные сайты
```

**3. Extended Validation (EV):**
```
- Проверка: расширенная проверка организации
- Стоимость: $200-1000/year
- Время: недели
- Use: банки, e-commerce (зелёная строка в старых браузерах)
```

**4. Wildcard Certificate:**
```
*.example.com
- Покрывает все поддомены
- НЕ покрывает example.com (нужен SAN)
```

**5. Multi-Domain (SAN):**
```
Subject Alternative Names:
- example.com
- www.example.com
- api.example.com
- *.example.com
```

---

## 🆓 Let's Encrypt

### Получение бесплатного сертификата

**Certbot (ACME client):**
```bash
# Установка (Ubuntu/Debian)
sudo apt update
sudo apt install certbot python3-certbot-nginx

# Для Nginx
sudo certbot --nginx -d example.com -d www.example.com

# Для Apache
sudo certbot --apache -d example.com -d www.example.com

# Manual (webroot)
sudo certbot certonly --webroot -w /var/www/html -d example.com
```

**Автоматическое обновление:**
```bash
# Let's Encrypt сертификаты действуют 90 дней
# Certbot настраивает автообновление через cron/systemd

# Проверка
sudo certbot renew --dry-run

# Принудительное обновление
sudo certbot renew

# Cron (автоматически)
0 0 * * * certbot renew --quiet --post-hook "systemctl reload nginx"
```

### ACME Protocol

**Automatic Certificate Management Environment:**

```
1. Certbot → Let's Encrypt: Запрос сертификата для example.com
2. Let's Encrypt → Certbot: Challenge (HTTP-01 или DNS-01)
3. Certbot: Создаёт файл в .well-known/acme-challenge/
4. Let's Encrypt: Проверяет http://example.com/.well-known/acme-challenge/token
5. Let's Encrypt → Certbot: Сертификат (если challenge успешен)
```

**Challenges:**
- **HTTP-01** - файл на веб-сервере (порт 80)
- **DNS-01** - TXT record в DNS (для wildcard)
- **TLS-ALPN-01** - TLS extension (порт 443)

---

## ⚙️ Nginx HTTPS Configuration

### Базовая конфигурация

```nginx
server {
    listen 80;
    server_name example.com www.example.com;
    
    # Редирект на HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name example.com www.example.com;
    
    # SSL Certificate
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    
    # SSL Configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';
    ssl_prefer_server_ciphers off;
    
    # HSTS (Force HTTPS)
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    
    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/letsencrypt/live/example.com/chain.pem;
    
    # Session Cache
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    
    root /var/www/html;
    index index.php index.html;
    
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }
    
    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

### Security Headers

```nginx
# Security headers
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header Referrer-Policy "no-referrer-when-downgrade" always;
add_header Content-Security-Policy "default-src 'self' https: data: 'unsafe-inline' 'unsafe-eval';" always;

# HSTS
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
```

---

## 🔐 Laravel HTTPS Configuration

### Force HTTPS

**AppServiceProvider:**
```php
// app/Providers/AppServiceProvider.php
use Illuminate\Support\Facades\URL;

public function boot()
{
    if ($this->app->environment('production')) {
        URL::forceScheme('https');
    }
}
```

**Middleware:**
```php
// app/Http/Middleware/ForceHttps.php
namespace App\Http\Middleware;

use Closure;

class ForceHttps
{
    public function handle($request, Closure $next)
    {
        if (!$request->secure() && app()->environment('production')) {
            return redirect()->secure($request->getRequestUri(), 301);
        }
        
        return $next($request);
    }
}

// app/Http/Kernel.php
protected $middleware = [
    \App\Http\Middleware\ForceHttps::class,
];
```

**.env:**
```env
APP_URL=https://example.com
SESSION_SECURE_COOKIE=true
```

### Trust Proxies (за Load Balancer)

```php
// app/Http/Middleware/TrustProxies.php
protected $proxies = '*';  // CloudFlare, AWS ELB, etc.

protected $headers = Request::HEADER_X_FORWARDED_ALL;
```

---

## 🎓 Для собеседования: ключевые точки

### SSH
1. **Public Key Authentication** - пара ключей (приватный + публичный), безопаснее пароля
2. **ssh-keygen** - RSA vs Ed25519 (Ed25519 рекомендуется - быстрее и безопаснее)
3. **authorized_keys** - файл с публичными ключами на сервере (~/.ssh/authorized_keys)
4. **SSH Tunneling** - Local forwarding (доступ к БД), Remote forwarding (показать localhost публично), Dynamic (SOCKS proxy)
5. **ssh-agent** - хранит ключи в памяти, не нужно вводить passphrase каждый раз
6. **Security** - отключить password auth, отключить root login, сменить порт, fail2ban
7. **Config** - ~/.ssh/config для алиасов и настроек

### SSL/TLS
1. **TLS Handshake** - ClientHello → ServerHello (certificate) → Key Exchange → Encrypted data
2. **TLS 1.3** - быстрее (1-RTT вместо 2-RTT), 0-RTT для повторных соединений
3. **Certificate Chain** - Root CA → Intermediate CA → Server Certificate
4. **Let's Encrypt** - бесплатные DV сертификаты, автообновление через certbot, ACME protocol (HTTP-01/DNS-01 challenges)
5. **HSTS** - Strict-Transport-Security заставляет браузер использовать только HTTPS
6. **Certificate Types** - DV (domain validation), OV (organization), EV (extended), Wildcard (*.example.com), SAN (multi-domain)
7. **Symmetric vs Asymmetric** - TLS handshake использует asymmetric (RSA/ECDSA) для обмена ключами, затем symmetric (AES) для данных (быстрее)
8. **Forward Secrecy** - даже если приватный ключ скомпрометирован, старые сессии защищены (ephemeral keys)

**Главное:** Понимай SSH key-based auth (безопаснее пароля), TLS handshake (как клиент и сервер договариваются о шифровании), certificate chain (как браузер проверяет сертификат), Let's Encrypt для бесплатных сертификатов.

# DNS (Domain Name System)

Система доменных имён - распределённая база данных для преобразования доменных имён в IP-адреса.

---

## 🌐 Что такое DNS

**DNS** - это "телефонная книга интернета", преобразует человекочитаемые имена в IP-адреса.

```
example.com → 93.184.216.34
```

### Зачем нужен DNS

**Проблема без DNS:**
```bash
# Пришлось бы запоминать
http://93.184.216.34/page
http://172.217.16.206/search
http://151.101.1.140/news
```

**С DNS:**
```bash
http://example.com/page
http://google.com/search
http://reddit.com/news
```

**Преимущества:**
- **Human-friendly** - легко запомнить имя вместо IP
- **Flexibility** - можно менять IP без изменения домена
- **Load Balancing** - один домен → несколько IP
- **Redundancy** - распределённая система

---

## 🔍 Что происходит при вводе URL в браузер

### Полный цикл DNS resolution

```
User вводит: https://example.com

1. Browser Cache
   ├─ Проверяет кеш браузера
   └─ НЕТ → дальше

2. OS Cache
   ├─ Проверяет кеш ОС (hosts file)
   └─ НЕТ → дальше

3. Router Cache
   ├─ Проверяет кеш роутера
   └─ НЕТ → дальше

4. ISP DNS Resolver (Recursive Resolver)
   ├─ Проверяет свой кеш
   └─ НЕТ → начинает рекурсивный поиск

5. Root Name Server (13 корневых серверов)
   ├─ Запрос: где найти .com?
   └─ Ответ: спроси TLD сервер 192.5.6.30

6. TLD Name Server (.com сервер)
   ├─ Запрос: где найти example.com?
   └─ Ответ: спроси Authoritative NS 93.184.216.34

7. Authoritative Name Server (example.com NS)
   ├─ Запрос: IP для example.com?
   └─ Ответ: 93.184.216.34

8. ISP Resolver
   ├─ Кеширует результат (TTL)
   └─ Возвращает IP браузеру

9. Browser
   └─ Устанавливает TCP соединение с 93.184.216.34
```

### Детальная схема

```
Browser                 Recursive Resolver       Root Server    TLD Server    Authoritative NS
   |                           |                      |              |                |
   |---(1) example.com?------->|                      |              |                |
   |                           |---(2) .com?--------->|              |                |
   |                           |<--(3) ask TLD--------|              |                |
   |                           |                      |              |                |
   |                           |---(4) example.com?----------------->|                |
   |                           |<--(5) ask Auth NS------------------|                |
   |                           |                      |              |                |
   |                           |---(6) example.com?------------------------------>    |
   |                           |<--(7) 93.184.216.34---------------------------------|
   |                           |                      |              |                |
   |<--(8) 93.184.216.34------|                      |              |                |
   |                           |                      |              |                |
   |===(9) TCP to 93.184.216.34========================================>
```

### Пример с реальными данными

```bash
# 1. Browser cache
chrome://net-internals/#dns

# 2. OS cache (Windows)
ipconfig /displaydns

# 3. Hosts file (приоритет над DNS)
# Windows: C:\Windows\System32\drivers\etc\hosts
# Linux: /etc/hosts
127.0.0.1  localhost
93.184.216.34  example.com  # принудительное разрешение

# 4. ISP Resolver (обычно 8.8.8.8 Google или провайдера)
# 5-7. Рекурсивный поиск через Root → TLD → Authoritative
```

---

## 📋 DNS Record Types

### A Record (Address)

**IPv4 адрес домена.**

```
example.com.  3600  IN  A  93.184.216.34
```

**Формат:**
```
<domain>  <TTL>  <class>  <type>  <value>
```

**PHP (Laravel):**
```php
// Получить A record
$ip = gethostbyname('example.com');
// "93.184.216.34"

// Все A records
$records = dns_get_record('example.com', DNS_A);
/*
[
    [
        'host' => 'example.com',
        'type' => 'A',
        'ip' => '93.184.216.34',
        'ttl' => 3600,
    ]
]
*/
```

### AAAA Record

**IPv6 адрес.**

```
example.com.  3600  IN  AAAA  2606:2800:220:1:248:1893:25c8:1946
```

```php
$records = dns_get_record('example.com', DNS_AAAA);
/*
[
    [
        'host' => 'example.com',
        'type' => 'AAAA',
        'ipv6' => '2606:2800:220:1:248:1893:25c8:1946',
        'ttl' => 3600,
    ]
]
*/
```

### CNAME Record (Canonical Name)

**Алиас (псевдоним) для другого домена.**

```
www.example.com.  3600  IN  CNAME  example.com.
blog.example.com. 3600  IN  CNAME  hosting.provider.com.
```

**Use cases:**
- `www.example.com` → `example.com`
- `blog.example.com` → `medium.com` (если блог на Medium)
- CDN: `cdn.example.com` → `d111111abcdef8.cloudfront.net`

**Важно:**
- CNAME не может сосуществовать с другими записями для того же имени
- Root domain (`example.com`) не может быть CNAME (только поддомены)

```php
$records = dns_get_record('www.example.com', DNS_CNAME);
/*
[
    [
        'host' => 'www.example.com',
        'type' => 'CNAME',
        'target' => 'example.com',
        'ttl' => 3600,
    ]
]
*/
```

### MX Record (Mail Exchange)

**Почтовые серверы для домена.**

```
example.com.  3600  IN  MX  10 mail1.example.com.
example.com.  3600  IN  MX  20 mail2.example.com.
```

**Priority (10, 20):**
- Меньшее число = выше приоритет
- `10` - первичный сервер
- `20` - резервный сервер

```php
$records = dns_get_record('example.com', DNS_MX);
/*
[
    [
        'host' => 'example.com',
        'type' => 'MX',
        'pri' => 10,
        'target' => 'mail1.example.com',
        'ttl' => 3600,
    ],
    [
        'host' => 'example.com',
        'type' => 'MX',
        'pri' => 20,
        'target' => 'mail2.example.com',
        'ttl' => 3600,
    ]
]
*/
```

### TXT Record

**Произвольный текст. Используется для верификации и настройки.**

**SPF (Sender Policy Framework) - защита от спуфинга:**
```
example.com.  3600  IN  TXT  "v=spf1 include:_spf.google.com ~all"
```

**DKIM (DomainKeys Identified Mail) - подпись писем:**
```
default._domainkey.example.com.  3600  IN  TXT  "v=DKIM1; k=rsa; p=MIGfMA0GCSqGSIb3..."
```

**DMARC (Domain-based Message Authentication):**
```
_dmarc.example.com.  3600  IN  TXT  "v=DMARC1; p=quarantine; rua=mailto:dmarc@example.com"
```

**Верификация домена (Google, Facebook, etc):**
```
example.com.  3600  IN  TXT  "google-site-verification=abc123xyz"
```

```php
$records = dns_get_record('example.com', DNS_TXT);
/*
[
    [
        'host' => 'example.com',
        'type' => 'TXT',
        'txt' => 'v=spf1 include:_spf.google.com ~all',
        'ttl' => 3600,
    ]
]
*/
```

### NS Record (Name Server)

**Authoritative DNS серверы для домена.**

```
example.com.  3600  IN  NS  ns1.example.com.
example.com.  3600  IN  NS  ns2.example.com.
```

```php
$records = dns_get_record('example.com', DNS_NS);
/*
[
    [
        'host' => 'example.com',
        'type' => 'NS',
        'target' => 'ns1.example.com',
        'ttl' => 3600,
    ],
    [
        'host' => 'example.com',
        'type' => 'NS',
        'target' => 'ns2.example.com',
        'ttl' => 3600,
    ]
]
*/
```

### SOA Record (Start of Authority)

**Информация о зоне DNS.**

```
example.com.  3600  IN  SOA  ns1.example.com. admin.example.com. (
    2024012801  ; Serial (YYYYMMDDNN)
    7200        ; Refresh (2 hours)
    3600        ; Retry (1 hour)
    1209600     ; Expire (2 weeks)
    3600        ; Minimum TTL (1 hour)
)
```

**Поля:**
- **Primary NS** - главный DNS сервер
- **Email** - email администратора (admin.example.com = admin@example.com)
- **Serial** - версия зоны (увеличивается при изменении)
- **Refresh** - как часто slave проверяет изменения
- **Retry** - повтор при ошибке
- **Expire** - когда slave перестаёт отвечать если master недоступен
- **Minimum TTL** - TTL для negative caching (NXDOMAIN)

```php
$records = dns_get_record('example.com', DNS_SOA);
/*
[
    [
        'host' => 'example.com',
        'type' => 'SOA',
        'mname' => 'ns1.example.com',
        'rname' => 'admin.example.com',
        'serial' => 2024012801,
        'refresh' => 7200,
        'retry' => 3600,
        'expire' => 1209600,
        'minimum-ttl' => 3600,
        'ttl' => 3600,
    ]
]
*/
```

### PTR Record (Pointer)

**Reverse DNS - IP → Domain.**

```
34.216.184.93.in-addr.arpa.  3600  IN  PTR  example.com.
```

**Use cases:**
- Верификация почтовых серверов (anti-spam)
- Логирование (reverse lookup)

```php
// Reverse DNS lookup
$domain = gethostbyaddr('93.184.216.34');
// "example.com"
```

### SRV Record (Service)

**Информация о сервисах.**

```
_ldap._tcp.example.com.  3600  IN  SRV  10 60 389 ldap.example.com.
_xmpp._tcp.example.com.  3600  IN  SRV  10 60 5222 xmpp.example.com.
```

**Формат:**
```
_service._protocol.domain  TTL  IN  SRV  priority weight port target
```

**Fields:**
- **Priority** - приоритет (меньше = выше)
- **Weight** - вес для load balancing
- **Port** - порт сервиса
- **Target** - сервер

```php
$records = dns_get_record('_ldap._tcp.example.com', DNS_SRV);
/*
[
    [
        'host' => '_ldap._tcp.example.com',
        'type' => 'SRV',
        'pri' => 10,
        'weight' => 60,
        'port' => 389,
        'target' => 'ldap.example.com',
        'ttl' => 3600,
    ]
]
*/
```

### CAA Record (Certification Authority Authorization)

**Указывает, какие CA могут выдавать сертификаты для домена.**

```
example.com.  3600  IN  CAA  0 issue "letsencrypt.org"
example.com.  3600  IN  CAA  0 issuewild ";"
example.com.  3600  IN  CAA  0 iodef "mailto:security@example.com"
```

**Защита от неавторизованных сертификатов.**

---

## ⏱️ TTL (Time To Live)

### Что такое TTL

**TTL** - время (в секундах), в течение которого DNS запись кешируется.

```
example.com.  3600  IN  A  93.184.216.34
              ↑
              TTL = 3600 секунд = 1 час
```

### Как работает TTL

```
1. Resolver запрашивает example.com
2. Получает ответ с TTL=3600
3. Кеширует на 1 час
4. Последующие запросы в течение 1 часа → из кеша
5. Через 1 час → новый запрос к Authoritative NS
```

### Выбор TTL

**Короткий TTL (300 сек = 5 мин):**
- ✅ Быстрые изменения DNS
- ✅ Миграция серверов
- ✅ A/B testing с DNS
- ❌ Больше нагрузки на DNS серверы
- ❌ Медленнее для пользователей

**Длинный TTL (86400 сек = 24 часа):**
- ✅ Меньше нагрузки на DNS
- ✅ Быстрее для пользователей (кеш)
- ❌ Медленная propagation при изменении
- ❌ Долгий downtime при сбое

**Best practices:**
```
Обычно:         3600-86400 (1-24 часа)
Перед миграцией: 300 (5 мин) - понизить за сутки до миграции
Статические:    86400 (24 часа)
Критичные:      600-1800 (10-30 мин)
```

### Пример стратегии миграции

```
День -1: Понизить TTL до 300 секунд

example.com.  300  IN  A  OLD_IP

День 0: Изменить IP

example.com.  300  IN  A  NEW_IP

(через 5 мин все видят новый IP)

День +1: Повысить TTL обратно

example.com.  86400  IN  A  NEW_IP
```

---

## 🔄 DNS Propagation

### Что такое DNS Propagation

**Распространение DNS изменений по всему интернету.**

**Факторы:**
1. **TTL** - основной фактор
2. **Количество кешей** - браузер, OS, router, ISP, recursive resolvers
3. **Geography** - разные ISP обновляются по-разному

### Временные рамки

```
TTL=300:    5-30 минут
TTL=3600:   1-6 часов
TTL=86400:  24-48 часов
```

**Полная propagation:** обычно 24-48 часов (даже с низким TTL из-за игнорирования TTL некоторыми resolvers).

### Проверка propagation

**Online tools:**
```
https://www.whatsmydns.net/
https://dnschecker.org/
```

**CLI:**
```bash
# dig (Linux/macOS)
dig example.com

# nslookup (Windows/Linux)
nslookup example.com

# host
host example.com

# С указанием DNS сервера
dig @8.8.8.8 example.com        # Google DNS
dig @1.1.1.1 example.com        # Cloudflare DNS
nslookup example.com 8.8.8.8
```

### Очистка кеша

**Browser:**
```
Chrome: chrome://net-internals/#dns → Clear host cache
Firefox: about:networking#dns → Clear DNS Cache
```

**OS:**
```bash
# Windows
ipconfig /flushdns

# macOS
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder

# Linux
sudo systemd-resolve --flush-caches
# or
sudo /etc/init.d/nscd restart
```

---

## 🔍 DNS Query Types

### Recursive Query

**Resolver делает всю работу за клиента.**

```
Client                  Recursive Resolver
  |                           |
  |----(1) example.com?------>|
  |                           |--- Root NS
  |                           |--- TLD NS
  |                           |--- Auth NS
  |                           |
  |<---(2) 93.184.216.34------|
```

**Клиент получает финальный ответ.**

### Iterative Query

**Resolver возвращает referral (ссылку) на следующий сервер.**

```
Client                  Iterative Resolver
  |                           |
  |----(1) example.com?------>|
  |<---(2) ask TLD NS---------|
  |
  |-----(3) .com?------------> TLD NS
  |<---(4) ask Auth NS--------|
  |
  |-----(5) example.com?-----> Auth NS
  |<---(6) 93.184.216.34------|
```

**Клиент сам делает последовательные запросы.**

**На практике:**
- Клиенты (браузеры) используют **recursive** queries к ISP resolver
- ISP resolver использует **iterative** queries к Root/TLD/Auth серверам

---

## 🚀 DNS Optimization

### DNS Prefetching

**Браузер заранее резолвит DNS для ссылок на странице.**

```html
<!-- Hint браузеру заранее резолвить DNS -->
<link rel="dns-prefetch" href="//example.com">
<link rel="dns-prefetch" href="//cdn.example.com">
<link rel="dns-prefetch" href="//analytics.google.com">
```

**Laravel:**
```php
// В layout
@foreach(['cdn.example.com', 'fonts.googleapis.com'] as $domain)
    <link rel="dns-prefetch" href="//{{ $domain }}">
@endforeach
```

### DNS Preconnect

**Не только DNS, но и TCP + TLS handshake.**

```html
<!-- DNS + TCP + TLS -->
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://cdn.example.com">
```

**Use cases:**
- Critical third-party resources (шрифты, CDN)
- API endpoints

### Multiple DNS Servers

**Redundancy для надёжности.**

```
Primary:   8.8.8.8 (Google)
Secondary: 1.1.1.1 (Cloudflare)
Tertiary:  208.67.222.222 (OpenDNS)
```

**Laravel (Queue, Cache):**
```php
// config/database.php - Redis
'redis' => [
    'client' => 'predis',
    'default' => [
        'host' => env('REDIS_HOST', 'redis.example.com'),
        // DNS resolution происходит при подключении
    ],
],

// Для критичных сервисов используй IP напрямую (минуя DNS):
'redis' => [
    'default' => [
        'host' => env('REDIS_HOST', '10.0.1.5'),  // IP вместо домена
    ],
],
```

---

## 🌍 DNS и CDN

### Geographic DNS (GeoDNS)

**Разные IP в зависимости от местоположения клиента.**

```
example.com для пользователей из:

USA:    A  34.208.123.45  (AWS us-east-1)
Europe: A  52.29.144.67   (AWS eu-west-1)
Asia:   A  13.228.67.89   (AWS ap-southeast-1)
```

**Route53 (AWS) Geolocation routing:**
```json
{
  "Type": "A",
  "SetIdentifier": "US",
  "GeoLocation": {
    "ContinentCode": "NA"
  },
  "ResourceRecords": [{"Value": "34.208.123.45"}]
}
```

**Cloudflare автоматически:**
- Возвращает ближайший IP из их сети
- Load balancing + failover

### CDN с DNS

```
www.example.com  →  CNAME  →  abc123.cloudfront.net

                              ↓
                         CloudFront GeoDNS
                              ↓
          ┌───────────────────┼───────────────────┐
          ↓                   ↓                   ↓
    Edge US-East        Edge EU-West       Edge Asia-Pacific
    54.230.1.45         54.230.50.78       54.230.100.23
```

---

## 🔒 DNS Security

### DNSSEC (DNS Security Extensions)

**Цифровые подписи для проверки подлинности DNS ответов.**

**Проблема без DNSSEC:**
- DNS spoofing / DNS poisoning
- Man-in-the-middle attacks
- Cache poisoning

**Как работает:**
```
1. Domain owner подписывает DNS records приватным ключом
2. Публичный ключ публикуется в DNS (DNSKEY record)
3. Resolver проверяет подпись публичным ключом
4. Chain of trust: Root → TLD → Domain
```

**Проверка DNSSEC:**
```bash
dig +dnssec example.com

# В ответе должны быть RRSIG records (подписи)
```

### DNS over HTTPS (DoH)

**DNS запросы через HTTPS - конфиденциальность.**

**Проблема:**
- Обычные DNS запросы - plaintext
- ISP видит все DNS запросы

**DoH:**
```
Browser → HTTPS (encrypted) → DoH Server (Cloudflare, Google)
                           ↓
                       DNS Response
```

**Cloudflare DoH:**
```
https://1.1.1.1/dns-query
https://cloudflare-dns.com/dns-query
```

**Google DoH:**
```
https://dns.google/dns-query
```

**Firefox:**
```
Settings → Privacy → DNS over HTTPS → Enable
```

### DNS over TLS (DoT)

**Аналог DoH, но через TLS на порту 853.**

```bash
# systemd-resolved (Linux)
sudo systemctl edit systemd-resolved

[Resolve]
DNS=1.1.1.1
DNSOverTLS=yes
```

---

## 🛠️ DNS Debugging

### dig (Domain Information Groper)

```bash
# Базовый запрос
dig example.com

# Короткий ответ
dig example.com +short
# 93.184.216.34

# Конкретный тип записи
dig example.com A
dig example.com AAAA
dig example.com MX
dig example.com NS
dig example.com TXT

# Указать DNS сервер
dig @8.8.8.8 example.com
dig @1.1.1.1 example.com

# Trace - показать весь путь резолюции
dig example.com +trace

# Все записи
dig example.com ANY

# Reverse DNS
dig -x 93.184.216.34

# DNSSEC
dig example.com +dnssec
```

**Пример вывода:**
```
; <<>> DiG 9.16.1 <<>> example.com
;; QUESTION SECTION:
;example.com.               IN  A

;; ANSWER SECTION:
example.com.        3600    IN  A   93.184.216.34

;; Query time: 15 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)
;; WHEN: Tue Jan 28 10:30:00 UTC 2026
;; MSG SIZE  rcvd: 56
```

### nslookup

```bash
# Windows/Linux/macOS
nslookup example.com

# С указанием сервера
nslookup example.com 8.8.8.8

# Конкретный тип
nslookup -type=MX example.com
nslookup -type=NS example.com
nslookup -type=TXT example.com

# Reverse
nslookup 93.184.216.34

# Interactive mode
nslookup
> server 8.8.8.8
> set type=A
> example.com
```

### host

```bash
# Простой запрос
host example.com

# Все записи
host -a example.com

# Конкретный тип
host -t MX example.com
host -t NS example.com

# Reverse
host 93.184.216.34

# С сервером
host example.com 8.8.8.8
```

### whois

```bash
# Информация о домене
whois example.com

# Registrar, регистратор
# Даты регистрации и истечения
# Name servers
# Контакты
```

### PHP Examples

```php
// Базовое разрешение
$ip = gethostbyname('example.com');
// "93.184.216.34"

// Все IP (если несколько A records)
$ips = gethostbynamel('example.com');
// ["93.184.216.34"]

// Reverse DNS
$domain = gethostbyaddr('93.184.216.34');
// "example.com"

// Полная информация
$records = dns_get_record('example.com', DNS_ALL);

// Конкретный тип
$a_records = dns_get_record('example.com', DNS_A);
$mx_records = dns_get_record('example.com', DNS_MX);
$txt_records = dns_get_record('example.com', DNS_TXT);
$ns_records = dns_get_record('example.com', DNS_NS);

// Проверка существования домена
if (checkdnsrr('example.com', 'A')) {
    echo "Domain exists";
}

// Проверка MX records (для email валидации)
if (checkdnsrr('example.com', 'MX')) {
    echo "Domain has mail servers";
}
```

**Laravel - валидация email с DNS проверкой:**
```php
// Email validation с проверкой MX records
$request->validate([
    'email' => [
        'required',
        'email:rfc,dns',  // проверяет DNS MX records
    ],
]);

// Custom validator
Validator::extend('email_dns', function ($attribute, $value, $parameters, $validator) {
    $domain = substr(strrchr($value, "@"), 1);
    return checkdnsrr($domain, 'MX') || checkdnsrr($domain, 'A');
});

$request->validate([
    'email' => 'required|email|email_dns',
]);
```

---

## 🎓 Для собеседования: ключевые точки

1. **DNS Resolution Process** - Browser cache → OS cache → ISP resolver → Root NS → TLD NS → Authoritative NS
2. **Record Types** - A (IPv4), AAAA (IPv6), CNAME (alias), MX (mail), TXT (verification/SPF/DKIM), NS (nameservers)
3. **TTL** - Time to Live, время кеширования, влияет на propagation (короткий TTL = быстрые изменения, длинный = меньше нагрузки)
4. **Query Types** - Recursive (resolver делает всё) vs Iterative (клиент сам идёт по цепочке)
5. **DNS Propagation** - распространение изменений, зависит от TTL, обычно 24-48 часов для полной propagation
6. **Security** - DNSSEC (подписи), DoH/DoT (шифрование запросов), защита от spoofing/poisoning
7. **CNAME** - алиас, не может сосуществовать с другими записями, root domain не может быть CNAME
8. **MX Priority** - меньшее число = выше приоритет (10 перед 20)
9. **Debugging** - dig, nslookup, host, whois, dns_get_record() в PHP
10. **Optimization** - dns-prefetch, preconnect, GeoDNS для CDN, низкий TTL перед миграцией

**Главное:** Понимай полный цикл DNS resolution от браузера до authoritative nameserver, знай основные типы записей и их применение, понимай влияние TTL на изменения.

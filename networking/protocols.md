# Протоколы TCP/IP и модель OSI

## Модель OSI (7 уровней)

```
┌─────────────────────────────────────┐
│ 7. Application (Прикладной)         │  HTTP, FTP, SMTP, DNS
├─────────────────────────────────────┤
│ 6. Presentation (Представления)     │  SSL/TLS, шифрование, кодировки
├─────────────────────────────────────┤
│ 5. Session (Сеансовый)              │  Сессии, аутентификация
├─────────────────────────────────────┤
│ 4. Transport (Транспортный)         │  TCP, UDP
├─────────────────────────────────────┤
│ 3. Network (Сетевой)                │  IP, ICMP, маршрутизация
├─────────────────────────────────────┤
│ 2. Data Link (Канальный)            │  Ethernet, MAC адреса
├─────────────────────────────────────┤
│ 1. Physical (Физический)            │  Кабели, WiFi, биты
└─────────────────────────────────────┘
```

## Модель TCP/IP (4 уровня)

```
┌─────────────────────────────────────┐
│ Application                         │  HTTP, FTP, SMTP, DNS
│ (Application + Presentation +       │
│  Session из OSI)                    │
├─────────────────────────────────────┤
│ Transport                           │  TCP, UDP
├─────────────────────────────────────┤
│ Internet                            │  IP, ICMP, ARP
├─────────────────────────────────────┤
│ Network Access                      │  Ethernet, WiFi
│ (Data Link + Physical из OSI)       │
└─────────────────────────────────────┘
```

---

## TCP (Transmission Control Protocol)

### Характеристики

✅ **Надежный:**
- Гарантия доставки
- Порядок пакетов
- Контроль ошибок

✅ **Connection-oriented:**
- Установка соединения (handshake)
- Поддержание состояния
- Закрытие соединения

❌ **Медленнее UDP:**
- Overhead на установку соединения
- Подтверждения (ACK)
- Повторная отправка

### TCP Header

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Acknowledgment Number                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Offset| Res |     Flags       |            Window             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Checksum            |         Urgent Pointer        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**Ключевые поля:**
- **Source/Destination Port** - порты отправителя/получателя
- **Sequence Number** - номер первого байта в сегменте
- **Acknowledgment Number** - следующий ожидаемый байт
- **Flags** - SYN, ACK, FIN, RST, PSH, URG
- **Window** - размер окна приема (flow control)
- **Checksum** - контрольная сумма

### Three-Way Handshake (установка соединения)

```
Client                          Server
  |                                |
  |  1. SYN (seq=x)               |
  |------------------------------>|
  |                                |
  |  2. SYN-ACK (seq=y, ack=x+1)  |
  |<------------------------------|
  |                                |
  |  3. ACK (seq=x+1, ack=y+1)    |
  |------------------------------>|
  |                                |
  |  Connection established        |
```

**Шаги:**
1. **Client → Server:** SYN (synchronize), seq=random
2. **Server → Client:** SYN-ACK, seq=random, ack=client_seq+1
3. **Client → Server:** ACK, ack=server_seq+1

### Four-Way Handshake (закрытие соединения)

```
Client                          Server
  |                                |
  |  1. FIN (seq=x)               |
  |------------------------------>|
  |                                |
  |  2. ACK (ack=x+1)             |
  |<------------------------------|
  |                                |
  |  3. FIN (seq=y)               |
  |<------------------------------|
  |                                |
  |  4. ACK (ack=y+1)             |
  |------------------------------>|
  |                                |
  |  Connection closed             |
```

### TCP Flow Control (Window Size)

```
Sender                          Receiver
  |                                |
  | Data (1000 bytes)              |
  |------------------------------>|
  |                                | Buffer: 5000 bytes free
  | ACK (window=5000)              |
  |<------------------------------|
  |                                |
  | Data (5000 bytes)              |
  |------------------------------>|
  |                                | Buffer: 0 bytes free
  | ACK (window=0)                 |
  |<------------------------------|
  |                                |
  | ... waits ...                  | Application reads data
  |                                | Buffer: 3000 bytes free
  | ACK (window=3000)              |
  |<------------------------------|
```

### TCP Congestion Control

**Алгоритмы:**
- **Slow Start** - экспоненциальный рост окна
- **Congestion Avoidance** - линейный рост
- **Fast Retransmit** - быстрая повторная отправка
- **Fast Recovery** - быстрое восстановление

---

## UDP (User Datagram Protocol)

### Характеристики

✅ **Быстрый:**
- Нет установки соединения
- Минимальный overhead
- Нет подтверждений

✅ **Connectionless:**
- Без состояния
- Независимые датаграммы

❌ **Ненадежный:**
- Нет гарантии доставки
- Порядок не гарантируется
- Нет контроля ошибок

### UDP Header (8 bytes)

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            Length             |           Checksum            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### TCP vs UDP

| Критерий | TCP | UDP |
|----------|-----|-----|
| **Надежность** | Гарантия доставки | Без гарантий |
| **Порядок** | Гарантируется | Не гарантируется |
| **Соединение** | Connection-oriented | Connectionless |
| **Скорость** | Медленнее | Быстрее |
| **Overhead** | Больше (20+ bytes) | Меньше (8 bytes) |
| **Use case** | HTTP, HTTPS, FTP, SSH | DNS, VoIP, Streaming, Gaming |

### Когда использовать UDP

✅ **UDP подходит для:**
- **Streaming** (video, audio) - потеря пакетов допустима
- **Gaming** - низкая задержка важнее надежности
- **DNS** - простой запрос-ответ
- **VoIP** - real-time коммуникация
- **IoT** - маломощные устройства

---

## IP (Internet Protocol)

### IPv4 Address

**Формат:** `xxx.xxx.xxx.xxx` (4 байта, 32 бита)

**Пример:** `192.168.1.1`

**Диапазон:** 0.0.0.0 - 255.255.255.255

**Всего адресов:** 2³² = ~4.3 миллиарда

### Классы адресов (устаревшая система)

| Класс | Диапазон | Маска по умолчанию | Использование |
|-------|----------|-------------------|---------------|
| A | 1.0.0.0 - 126.255.255.255 | /8 (255.0.0.0) | Большие сети |
| B | 128.0.0.0 - 191.255.255.255 | /16 (255.255.0.0) | Средние сети |
| C | 192.0.0.0 - 223.255.255.255 | /24 (255.255.255.0) | Малые сети |
| D | 224.0.0.0 - 239.255.255.255 | - | Multicast |
| E | 240.0.0.0 - 255.255.255.255 | - | Зарезервировано |

### Частные IP адреса (RFC 1918)

```
10.0.0.0 - 10.255.255.255     (10/8)
172.16.0.0 - 172.31.255.255   (172.16/12)
192.168.0.0 - 192.168.255.255 (192.168/16)
```

Используются во внутренних сетях (не маршрутизируются в интернете).

### CIDR (Classless Inter-Domain Routing)

**Формат:** `192.168.1.0/24`

`/24` = маска `255.255.255.0` = первые 24 бита для сети

**Примеры:**
```
192.168.1.0/24
  Сеть: 192.168.1.0
  Маска: 255.255.255.0
  Хосты: 192.168.1.1 - 192.168.1.254 (254 хоста)
  Broadcast: 192.168.1.255

10.0.0.0/8
  Сеть: 10.0.0.0
  Маска: 255.0.0.0
  Хосты: 10.0.0.1 - 10.255.255.254 (~16 млн хостов)
```

### IPv6 Address

**Формат:** 8 групп по 4 hex цифры (128 бит)

**Пример:** `2001:0db8:85a3:0000:0000:8a2e:0370:7334`

**Сокращения:**
```
2001:0db8:85a3:0000:0000:8a2e:0370:7334
↓ Убираем ведущие нули
2001:db8:85a3:0:0:8a2e:370:7334
↓ Заменяем последовательность нулей на ::
2001:db8:85a3::8a2e:370:7334
```

**Всего адресов:** 2¹²⁸ = ~340 ундециллионов

---

## DNS (Domain Name System)

### Иерархия DNS

```
                .  (root)
                |
        .com    .org    .ru
          |       |       |
    google.com  wiki.org  ya.ru
      |
  www.google.com
```

### Типы DNS записей

| Тип | Описание | Пример |
|-----|----------|--------|
| **A** | IPv4 адрес | `example.com → 93.184.216.34` |
| **AAAA** | IPv6 адрес | `example.com → 2606:2800:220:1:...` |
| **CNAME** | Каноническое имя (alias) | `www.example.com → example.com` |
| **MX** | Mail server | `example.com → mail.example.com` |
| **TXT** | Текстовая информация | SPF, DKIM, verification |
| **NS** | Name server | `example.com → ns1.example.com` |
| **SOA** | Start of Authority | Информация о зоне |
| **PTR** | Reverse DNS | `34.216.184.93.in-addr.arpa → example.com` |

### Процесс DNS резолвинга

```
1. Пользователь: www.example.com
   ↓
2. Browser cache → OS cache → Router cache
   ↓ (если нет в кеше)
3. Recursive DNS Resolver (ISP)
   ↓
4. Root Name Server (.) → возвращает .com NS
   ↓
5. TLD Name Server (.com) → возвращает example.com NS
   ↓
6. Authoritative Name Server (example.com) → возвращает IP
   ↓
7. IP адрес возвращается браузеру
```

### TTL (Time To Live)

```
example.com.  3600  IN  A  93.184.216.34
               ↑
              TTL (seconds)
```

- Время кеширования записи
- После истечения TTL нужен новый запрос

### DNS tools

```bash
# Lookup
nslookup example.com

# Dig (детальная информация)
dig example.com
dig example.com MX
dig @8.8.8.8 example.com  # Используя конкретный DNS сервер

# Host
host example.com

# Reverse lookup
dig -x 93.184.216.34
```

---

## HTTP(S)

### HTTP Methods

| Метод | Назначение | Idempotent | Safe |
|-------|-----------|------------|------|
| **GET** | Получение ресурса | ✅ | ✅ |
| **POST** | Создание ресурса | ❌ | ❌ |
| **PUT** | Обновление ресурса (полное) | ✅ | ❌ |
| **PATCH** | Частичное обновление | ❌ | ❌ |
| **DELETE** | Удаление ресурса | ✅ | ❌ |
| **HEAD** | Как GET, но без body | ✅ | ✅ |
| **OPTIONS** | Доступные методы | ✅ | ✅ |

### HTTP Status Codes

**1xx - Informational:**
- `100 Continue`
- `101 Switching Protocols`

**2xx - Success:**
- `200 OK`
- `201 Created`
- `204 No Content`

**3xx - Redirection:**
- `301 Moved Permanently`
- `302 Found` (temporary redirect)
- `304 Not Modified`
- `307 Temporary Redirect`
- `308 Permanent Redirect`

**4xx - Client Error:**
- `400 Bad Request`
- `401 Unauthorized` (authentication required)
- `403 Forbidden` (authenticated but no access)
- `404 Not Found`
- `422 Unprocessable Entity` (validation error)
- `429 Too Many Requests`

**5xx - Server Error:**
- `500 Internal Server Error`
- `502 Bad Gateway`
- `503 Service Unavailable`
- `504 Gateway Timeout`

### HTTP Headers

**Request Headers:**
```http
GET /api/users HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0...
Accept: application/json
Authorization: Bearer <token>
Content-Type: application/json
Cookie: session_id=abc123
```

**Response Headers:**
```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 1234
Cache-Control: max-age=3600
Set-Cookie: session_id=abc123; HttpOnly; Secure
```

### HTTPS (HTTP over SSL/TLS)

**Процесс:**
1. **TCP Handshake** (3-way)
2. **TLS Handshake:**
   - Client Hello (supported ciphers)
   - Server Hello (chosen cipher)
   - Certificate (server's public key)
   - Key Exchange
   - Finished
3. **Encrypted HTTP communication**

---

## Что происходит при вводе URL

**Пример:** `https://www.example.com/page`

### Полный процесс:

1. **Парсинг URL:**
   - Протокол: HTTPS
   - Домен: www.example.com
   - Путь: /page

2. **DNS резолвинг:**
   - Проверка кешей
   - Запрос к DNS серверам
   - Получение IP: `93.184.216.34`

3. **TCP соединение:**
   - Three-way handshake с сервером на порту 443

4. **TLS Handshake:**
   - Обмен ключами
   - Проверка сертификата
   - Установка зашифрованного канала

5. **HTTP запрос:**
   ```http
   GET /page HTTP/1.1
   Host: www.example.com
   ```

6. **Обработка на сервере:**
   - Nginx/Apache получает запрос
   - Передает в PHP-FPM/приложение
   - Генерируется ответ

7. **HTTP ответ:**
   ```http
   HTTP/1.1 200 OK
   Content-Type: text/html
   
   <html>...</html>
   ```

8. **Rendering:**
   - Парсинг HTML
   - Загрузка ресурсов (CSS, JS, images)
   - Построение DOM
   - Отрисовка страницы

---

## Ключевые вопросы для интервью

- Объясните разницу между TCP и UDP
- Как работает TCP three-way handshake?
- Что происходит при вводе URL в браузер?
- Как работает DNS резолвинг?
- В чем разница между HTTP и HTTPS?
- Что такое TLS handshake?
- Объясните разницу между 301 и 302 редиректами
- Что такое idempotent метод?
- Чем отличается PUT от PATCH?
- Какие уровни модели OSI вы знаете?


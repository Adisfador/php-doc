# Физическое хранение данных (Deep Dive)

## Файловая структура PostgreSQL

### Организация файлов

```
$PGDATA/
├── base/              # Базы данных
│   ├── 1/            # template1
│   ├── 13445/        # Ваша БД
│   │   ├── 16384     # Файл таблицы
│   │   ├── 16384.1   # Дополнительный сегмент (если > 1GB)
│   │   └── 16384_fsm # Free Space Map
│   │   └── 16384_vm  # Visibility Map
├── global/           # Общие объекты (роли, базы)
├── pg_wal/          # Write-Ahead Log
├── pg_xact/         # Transaction commit status
└── pg_multixact/    # Multitransaction status
```

### Размер файлов
- По умолчанию макс. размер файла таблицы: **1 GB**
- При превышении создается новый сегмент (.1, .2 и т.д.)
- Настраивается при компиляции (`--with-segsize`)

---

## Страницы (Pages)

### Структура страницы (8 KB по умолчанию)

```
┌─────────────────────────────────────────┐  ← 0 байт
│      Page Header (24 bytes)             │
├─────────────────────────────────────────┤  ← 24
│      Item Pointers (Array)              │  Растут вниз →
│      (4 bytes each)                     │
├─────────────────────────────────────────┤
│                                         │
│      Free Space                         │
│                                         │
├─────────────────────────────────────────┤
│      ← Special Space (для индексов)     │  Опционально
├─────────────────────────────────────────┤
│      ← Tuples (строки)                  │  Растут вверх ←
│      (переменный размер)                │
└─────────────────────────────────────────┘  ← 8192 байта
```

### Page Header (24 bytes)

| Поле | Размер | Описание |
|------|--------|----------|
| `pd_lsn` | 8 bytes | Log Sequence Number (для WAL) |
| `pd_checksum` | 2 bytes | Контрольная сумма страницы |
| `pd_flags` | 2 bytes | Флаги (has_free_lines, has_tuples и т.д.) |
| `pd_lower` | 2 bytes | Смещение конца массива указателей |
| `pd_upper` | 2 bytes | Смещение начала свободного места |
| `pd_special` | 2 bytes | Смещение специального пространства |
| `pd_pagesize_version` | 2 bytes | Размер страницы и версия |
| `pd_prune_xid` | 4 bytes | Oldest unpruned XMAX |

### Item Pointer (Line Pointer) - 4 bytes

```
┌──────────────┬──────────────┬──────────┐
│   lp_off     │   lp_flags   │  lp_len  │
│  (15 bits)   │   (2 bits)   │ (15 bits)│
└──────────────┴──────────────┴──────────┘
```

- `lp_off` - смещение tuple от начала страницы
- `lp_flags` - статус (unused, normal, redirect, dead)
- `lp_len` - длина tuple

---

## Tuple (строка данных)

### Структура Tuple

```
┌─────────────────────────────────────────┐
│      Tuple Header (~23 bytes)           │
├─────────────────────────────────────────┤
│      Null Bitmap (опционально)          │
├─────────────────────────────────────────┤
│      OID (опционально, 4 bytes)         │
├─────────────────────────────────────────┤
│      User Data                          │
│      (столбцы таблицы)                  │
└─────────────────────────────────────────┘
```

### Tuple Header (HeapTupleHeaderData)

| Поле | Размер | Описание |
|------|--------|----------|
| `t_xmin` | 4 bytes | ID транзакции, вставившей tuple |
| `t_xmax` | 4 bytes | ID транзакции, удалившей/обновившей tuple |
| `t_cid` | 4 bytes | Command ID (внутри транзакции) |
| `t_ctid` | 6 bytes | Физический адрес tuple (block + offset) или новой версии |
| `t_infomask2` | 2 bytes | Битовые флаги (число колонок, heap-only tuple) |
| `t_infomask` | 2 bytes | Битовые флаги (XMIN valid, XMAX valid, null values) |
| `t_hoff` | 1 byte | Смещение начала user data |

### Null Bitmap
- По 1 биту на каждый столбец
- Присутствует только если есть NULL значения
- Округляется до байта

---

## TOAST (The Oversized-Attribute Storage Technique)

### Зачем нужен TOAST
- Строка не должна превышать ~2KB (чтобы помещалось несколько строк на страницу 8KB)
- Большие значения (TEXT, BYTEA, JSON) выносятся в отдельную таблицу

### Стратегии TOAST

| Стратегия | Описание |
|-----------|----------|
| **PLAIN** | Без TOAST, без сжатия (для типов фиксированной длины) |
| **EXTENDED** | Сжатие + TOAST (по умолчанию для TEXT, JSON и т.д.) |
| **EXTERNAL** | Только TOAST, без сжатия (для уже сжатых данных) |
| **MAIN** | Сжатие, TOAST только в крайнем случае |

```sql
ALTER TABLE mytable 
ALTER COLUMN data SET STORAGE EXTERNAL;
```

### TOAST таблица

```
pg_toast.pg_toast_<oid>
├── chunk_id    (OID основной строки)
├── chunk_seq   (Номер чанка)
└── chunk_data  (До ~2KB данных)
```

**Пример:**
```sql
CREATE TABLE articles (
    id INT,
    content TEXT  -- Если > 2KB, будет TOAST
);

-- При вставке большого текста:
INSERT INTO articles VALUES (1, repeat('A', 10000));

-- PostgreSQL создаст записи в pg_toast.pg_toast_<oid>:
-- chunk_id=123, chunk_seq=0, chunk_data='AAA...' (2KB)
-- chunk_id=123, chunk_seq=1, chunk_data='AAA...' (2KB)
-- ...
```

---

## Free Space Map (FSM)

### Назначение
- Отслеживает свободное место на каждой странице
- Ускоряет поиск страницы для INSERT
- Файл `<relfilenode>_fsm`

### Структура
- Дерево с 3 уровнями (для таблиц < 1GB)
- Каждый узел хранит макс. свободное место в дочерних страницах
- Точность: 1/256 от размера страницы (~32 байта)

```sql
-- Проверить FSM
SELECT * FROM pg_freespace('mytable');
```

---

## Visibility Map (VM)

### Назначение
- Отмечает страницы, где все tuple видимы для всех транзакций
- Оптимизация VACUUM (пропускает такие страницы)
- Index-Only Scans (можно не читать саму таблицу)
- Файл `<relfilenode>_vm`

### Структура
- Bitmap: 2 бита на страницу
  - Бит 1: all-visible (все tuple видимы)
  - Бит 2: all-frozen (все tuple заморожены)

```sql
-- Проверить VM
SELECT * FROM pg_visibility_map('mytable');
```

---

## Write-Ahead Log (WAL)

### Принцип работы
1. Изменения сначала записываются в WAL
2. Затем применяются к data files (асинхронно)
3. При сбое: replay WAL для восстановления

### Структура WAL записи

```
┌────────────────────────────────────┐
│  WAL Record Header                 │
│  - total length                    │
│  - transaction ID                  │
│  - resource manager ID             │
│  - record type                     │
├────────────────────────────────────┤
│  Data (изменения страницы)         │
│  - block number                    │
│  - offset                          │
│  - old/new tuple data              │
└────────────────────────────────────┘
```

### WAL файлы
- Размер: **16 MB** (по умолчанию)
- Имя: `000000010000000000000001`
  - Timeline ID / Log ID / Segment ID
- После заполнения архивируются или удаляются

### LSN (Log Sequence Number)
- Уникальный адрес в WAL: `<файл>/<смещение>`
- Формат: `0/15D7A40` (WAL file / offset)

### Checkpoints
- Периодическое сохранение всех грязных страниц на диск
- Минимизирует объем WAL для replay
- Настраивается через `checkpoint_timeout`, `max_wal_size`

```sql
-- Принудительный checkpoint
CHECKPOINT;
```

---

## Индексы (физическое устройство)

### B-Tree индекс

#### Структура страницы B-Tree

```
┌─────────────────────────────────────────┐
│      Meta Page (root, уровень дерева)   │  Страница 0
├─────────────────────────────────────────┤
│      Root Page                          │  Страница 1
│  ┌──────┬──────┬──────┐                 │
│  │ <=10 │ <=20 │ <=30 │ (указатели)    │
│  └──────┴──────┴──────┘                 │
├─────────────────────────────────────────┤
│      Internal Page                      │  Страница 2
│  ┌───┬───┬───┬───┐                      │
│  │<=5│<=8│...│     │                    │
│  └───┴───┴───┴───┘                      │
├─────────────────────────────────────────┤
│      Leaf Page                          │  Страница 3
│  ┌─────────────────────────────────┐    │
│  │ Key=1, TID=(page=10, offset=1) │    │  TID = Tuple ID
│  │ Key=2, TID=(page=10, offset=2) │    │
│  │ ...                             │    │
│  └─────────────────────────────────┘    │
└─────────────────────────────────────────┘
```

**Leaf pages содержат:**
- Indexed key value
- TID (Tuple Identifier): блок + смещение в heap таблице

**Поиск по B-Tree:**
1. Читаем meta page → находим root
2. Читаем root → бинарный поиск → переход на internal page
3. Читаем internal → переход на leaf page
4. Читаем leaf → находим TID
5. Читаем heap page по TID → получаем данные

### Hash индекс

- Страницы разбиты на buckets
- Hash function определяет bucket
- Каждый bucket - linked list страниц
- **Не поддерживает:** ORDER BY, range scans
- **Поддерживает:** только равенство `=`

### GiST (Generalized Search Tree)

- Универсальный индекс для custom типов
- Используется для: геометрия (PostGIS), full-text search, ltree
- Сбалансированное дерево с предикатами

### GIN (Generalized Inverted Index)

- Для составных значений: массивы, JSONB, full-text
- Структура: posting tree + posting lists

```
┌────────────────────────────────┐
│  Entry Tree (B-Tree)           │
│  ┌───────┬───────┬───────┐     │
│  │ word1 │ word2 │ word3 │     │
│  └───┬───┴───┬───┴───┬───┘     │
│      │       │       │         │
├──────┼───────┼───────┼─────────┤
│  Posting Lists                 │
│  word1 → [TID1, TID5, TID9]    │
│  word2 → [TID2, TID3]          │
│  word3 → [TID1, TID8, TID10]   │
└────────────────────────────────┘
```

### BRIN (Block Range Index)

- Минимальная/максимальная граница для диапазона страниц
- Очень маленький размер индекса
- Эффективен для отсортированных данных

```
┌────────────────────────────────┐
│  Pages 0-999:   min=1, max=100 │
│  Pages 1000-1999: min=101, max=200│
│  Pages 2000-2999: min=201, max=300│
└────────────────────────────────┘
```

---

## Процесс INSERT

1. **Выделение места:**
   - Проверка FSM для поиска страницы со свободным местом
   - Если нет места → выделение новой страницы

2. **Запись tuple:**
   - Формирование tuple (header + null bitmap + data)
   - TOAST для больших значений
   - Запись в heap page

3. **Обновление индексов:**
   - Вставка в каждый индекс (key → TID)

4. **WAL запись:**
   - Запись изменений в WAL
   - fsync (если синхронный commit)

5. **Обновление метаданных:**
   - FSM (уменьшение свободного места)
   - VM (сброс флага all-visible)

---

## Процесс UPDATE

### В PostgreSQL UPDATE = DELETE + INSERT

1. **Старая версия:**
   - Установка `xmax` = текущий XID
   - Tuple становится "dead" после коммита

2. **Новая версия:**
   - Создание нового tuple с `xmin` = текущий XID
   - `t_ctid` старой версии → новая версия

3. **HOT Update (Heap-Only Tuple):**
   - Оптимизация: если новая версия помещается на ту же страницу
   - И обновленные столбцы не индексированы
   - Индексы не обновляются!

```sql
-- HOT update возможен
UPDATE users SET last_login = NOW() WHERE id = 1;

-- HOT update НЕвозможен (обновляем индексированный столбец)
UPDATE users SET email = 'new@email.com' WHERE id = 1;
```

---

## Bloat и VACUUM

### Причины Bloat
- Мертвые tuple после UPDATE/DELETE
- Неиспользуемое пространство
- TOAST bloat

### VACUUM процесс

1. **Сканирование таблицы:**
   - Чтение каждой страницы
   - Проверка visibility каждого tuple

2. **Удаление мертвых tuple:**
   - Помечаются как unused
   - Обновление FSM

3. **Обновление индексов:**
   - Удаление ссылок на мертвые tuple

4. **Truncate:**
   - Освобождение пустых страниц в конце файла

### VACUUM FULL

- Полная перестройка таблицы
- Создание нового файла без bloat
- **Блокирует** таблицу (ACCESS EXCLUSIVE)
- Освобождает место в ОС

```sql
-- Обычный VACUUM
VACUUM mytable;

-- С анализом статистики
VACUUM ANALYZE mytable;

-- Полная очистка (долгая операция!)
VACUUM FULL mytable;
```

### Autovacuum

```sql
-- Настройки autovacuum
ALTER TABLE mytable SET (
    autovacuum_vacuum_scale_factor = 0.1,  -- 10% мертвых строк
    autovacuum_vacuum_threshold = 50       -- Минимум 50 мертвых строк
);
```

---

## Ключевые вопросы для интервью

- Из каких частей состоит страница в PostgreSQL?
- Как хранятся строки больше 2KB?
- Зачем нужны FSM и VM?
- Чем отличается VACUUM от VACUUM FULL?
- Что такое HOT update и когда он возможен?
- Как работает WAL и зачем нужны checkpoints?
- Как B-Tree индекс связан с heap таблицей?
- Что происходит при UPDATE на физическом уровне?
- Почему UPDATE в PostgreSQL создает новую версию строки?

# Транзакции и изоляция (Deep Dive)

## ACID принципы

### Atomicity (Атомарность)
- Транзакция либо выполняется полностью, либо не выполняется вообще
- Write-Ahead Logging (WAL) в PostgreSQL
- Undo/Redo logs
- Rollback mechanisms

### Consistency (Согласованность)
- Целостность данных после транзакции
- Constraint checking
- Triggers и их роль
- Deferred constraints

### Isolation (Изолированность)
- Одновременные транзакции не влияют друг на друга
- Уровни изоляции (см. ниже)

### Durability (Долговечность)
- Подтвержденные транзакции сохраняются даже при сбое
- fsync и sync commit
- WAL archiving
- Checkpoints

---

## Проблемы параллельных транзакций (аномалии)

### 1. Dirty Read (Грязное чтение)
- Чтение незафиксированных данных другой транзакции
```sql
-- TX1
UPDATE accounts SET balance = 1000 WHERE id = 1;
-- TX2 читает balance = 1000 (некоммитнутое значение)
-- TX1
ROLLBACK; -- TX2 прочитала "грязные" данные
```

### 2. Non-Repeatable Read (Неповторяющееся чтение)
- Повторное чтение возвращает другие данные
```sql
-- TX1
SELECT balance FROM accounts WHERE id = 1; -- 500
-- TX2
UPDATE accounts SET balance = 1000 WHERE id = 1;
COMMIT;
-- TX1
SELECT balance FROM accounts WHERE id = 1; -- 1000 (изменилось!)
```

### 3. Phantom Read (Фантомное чтение)
- Появление/исчезновение строк при повторном запросе
```sql
-- TX1
SELECT COUNT(*) FROM accounts WHERE balance > 500; -- 10 строк
-- TX2
INSERT INTO accounts (balance) VALUES (1000);
COMMIT;
-- TX1
SELECT COUNT(*) FROM accounts WHERE balance > 500; -- 11 строк
```

### 4. Lost Update (Потерянное обновление)
- Одна транзакция перезаписывает изменения другой
```sql
-- TX1: читает balance = 500
-- TX2: читает balance = 500
-- TX1: пишет balance = 500 + 100 = 600
-- TX2: пишет balance = 500 + 50 = 550 (потеряли +100!)
```

### 5. Write Skew (Аномалия записи)
- Две транзакции читают одни данные и обновляют разные строки, нарушая бизнес-логику
```sql
-- Правило: минимум 1 врач на дежурстве
-- TX1: SELECT COUNT(*) WHERE on_duty = true; -- 2
-- TX2: SELECT COUNT(*) WHERE on_duty = true; -- 2
-- TX1: UPDATE doctors SET on_duty = false WHERE id = 1; COMMIT;
-- TX2: UPDATE doctors SET on_duty = false WHERE id = 2; COMMIT;
-- Результат: 0 врачей на дежурстве!
```

### 6. Serialization Anomaly
- Результат параллельного выполнения невозможен ни при какой последовательности транзакций

---

## Уровни изоляции

| Уровень | Dirty Read | Non-Repeatable Read | Phantom Read | Lost Update | Write Skew | Serialization Anomaly |
|---------|------------|---------------------|--------------|-------------|------------|-----------------------|
| **Read Uncommitted** | ✅ Возможен | ✅ Возможен | ✅ Возможен | ✅ Возможен | ✅ Возможен | ✅ Возможен |
| **Read Committed** | ❌ Невозможен | ✅ Возможен | ✅ Возможен | ✅ Возможен | ✅ Возможен | ✅ Возможен |
| **Repeatable Read** | ❌ Невозможен | ❌ Невозможен | ⚠️ Зависит* | ⚠️ Зависит** | ✅ Возможен | ✅ Возможен |
| **Serializable** | ❌ Невозможен | ❌ Невозможен | ❌ Невозможен | ❌ Невозможен | ❌ Невозможен | ❌ Невозможен |

*PostgreSQL: Phantom Read невозможен на RR (snapshot isolation). MySQL/InnoDB: обычные SELECT читают снимок, locking reads защищены next-key locks, но после собственного UPDATE чужие вставленные строки могут «появиться»  
**PostgreSQL: Lost Update предотвращается на RR (ошибка 40001 `could not serialize access`). MySQL/InnoDB: НЕ предотвращается при схеме `SELECT → вычисление в приложении → UPDATE` (UPDATE делает current read и перезаписывает). Защита: `SELECT ... FOR UPDATE`, атомарный `SET x = x + 1`, version-колонка, SERIALIZABLE

### Read Uncommitted
```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
```
- Минимальная изоляция
- Практически не используется
- В PostgreSQL эквивалентен Read Committed

### Read Committed (по умолчанию в PostgreSQL; в MySQL InnoDB по умолчанию Repeatable Read)
```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
```
- Видны только зафиксированные данные
- Каждый запрос видит свой снимок данных
- Нет грязного чтения

### Repeatable Read
```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```
- Транзакция видит снимок данных на момент первого запроса
- Повторные чтения возвращают те же данные
- В PostgreSQL: основан на MVCC, может быть serialization failure

**RR: PostgreSQL vs MySQL/InnoDB**

| | PostgreSQL | MySQL/InnoDB |
|---|---|---|
| По умолчанию | нет (Read Committed) | да |
| Снимок | на первый запрос транзакции | на первый consistent read |
| `UPDATE`/`DELETE` | работают по снимку; при конфликте с чужим коммитом — `40001` | current read: видят последнюю закоммиченную версию, блокируют строки |
| Lost update | предотвращается (ошибка + retry) | **возможен** (`SELECT` → `UPDATE` значением из приложения) |
| Фантомы | нет (snapshot) | snapshot-чтения без фантомов; locking reads защищены next-key/gap locks |
| Gap locks | нет | есть |

```sql
-- InnoDB, REPEATABLE READ: lost update
-- T1: SELECT balance FROM accounts WHERE id=1;   -- 100
-- T2: SELECT balance FROM accounts WHERE id=1;   -- 100
-- T1: UPDATE accounts SET balance = 150 WHERE id=1; COMMIT;
-- T2: UPDATE accounts SET balance = 120 WHERE id=1; COMMIT;  -- ждал lock, потом затёр
-- Итог 120, прибавка T1 потеряна. В PostgreSQL T2 получила бы 40001.
```

Защита в InnoDB: `SELECT ... FOR UPDATE`, `SET balance = balance + 50`, version-колонка, SERIALIZABLE (+ retry при deadlock).

### Serializable
```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```
- Полная изоляция
- Serialization failures при конфликтах
- SSI (Serializable Snapshot Isolation) в PostgreSQL
- Требует retry логики в приложении

### Производительность уровней изоляции

Сам снимок (snapshot / read view) стоит дёшево, поэтому **RC и RR по скорости отличаются мало**. Реальные расходы дают локи, долгие транзакции и откаты с повторами.

| | Read Committed | Repeatable Read | Serializable |
|---|---|---|---|
| **PostgreSQL** | базовая скорость | почти такая же; риск bloat от долгих транзакций, откаты `40001` | накладные расходы SSI + откаты и retry |
| **InnoDB** | меньше всего локов | больше локов (gap/next-key): ожидания и дедлоки | каждый `SELECT` ставит shared-лок, читатели блокируют писателей |

**PostgreSQL:**
- RC: снимок на каждую команду, между командами не держится
- RR: один снимок на всю транзакцию → пока он жив, `VACUUM` не может чистить мёртвые версии строк, которые снимок мог бы увидеть → bloat и замедление. Поэтому долгие RR-транзакции вредны
- `40001` на RR/Serializable: транзакция отработала впустую и выполняется заново
- Serializable (SSI): чтения не блокируют, но СУБД ведёт SIREAD-метки (память + CPU). При их укрупнении (строка → страница → таблица) растёт число ложных откатов. Накладные расходы заметны, но при редких конфликтах терпимы

**InnoDB:**
- RC: gap-локов почти нет, лишние локи на непопавшие под условие строки снимаются сразу после команды → меньше ожиданий и дедлоков (поэтому RC часто выбирают для write-heavy нагрузки)
- RR: `UPDATE`/`FOR UPDATE` держат next-key/gap-локи на весь просмотренный диапазон до конца транзакции → вставки в диапазон ждут
- Долгая RR-транзакция держит read view → purge не чистит undo log → растёт `history list length`, чтения старых версий замедляются
- Serializable: самый сильный спад пропускной способности при конкуренции

**От чего зависит итог (не от «частоты изменений» вообще):**
1. **Конкуренция** — сколько транзакций одновременно трогают одни и те же строки. Мало → разницы почти нет, в PG даже Serializable недорог
2. **Длина транзакции** — чем длиннее, тем дольше держатся локи (InnoDB) или снимок (PG)
3. **Доля записи** — read-heavy нагрузка мало страдает на любом уровне, write-heavy на InnoDB выигрывает от RC

**Практика:** держи транзакции короткими, не делай внутри них внешние вызовы, повышай уровень точечно на нужную транзакцию (`BEGIN ISOLATION LEVEL ...`), а не глобально; на RR/Serializable в PG retry обязателен.

---

## Блокировки (Locks)

### Типы блокировок

#### 1. Pessimistic Locking (Пессимистичные блокировки)
Блокируем данные до завершения транзакции

**Shared Lock (S-lock) - разделяемая блокировка**
```sql
SELECT * FROM accounts WHERE id = 1 FOR SHARE;
-- Другие могут читать, но не могут изменять
```

**Exclusive Lock (X-lock) - эксклюзивная блокировка**
```sql
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
-- Никто не может читать с блокировкой или изменять
```

**FOR UPDATE варианты:**
```sql
-- Ждать освобождения блокировки
SELECT ... FOR UPDATE;

-- Пропустить заблокированные строки
SELECT ... FOR UPDATE SKIP LOCKED;

-- Вернуть ошибку если заблокировано
SELECT ... FOR UPDATE NOWAIT;
```

**Table-level locks**
```sql
LOCK TABLE accounts IN ACCESS EXCLUSIVE MODE;
```

Режимы блокировок таблиц (от слабых к сильным). В скобках — команды, которые **сами берут** этот режим на таблицу (это не список того, что разрешено):
- ACCESS SHARE (SELECT)
- ROW SHARE (SELECT FOR UPDATE / FOR SHARE)
- ROW EXCLUSIVE (INSERT, UPDATE, DELETE)
- SHARE UPDATE EXCLUSIVE (VACUUM без FULL, ANALYZE, CREATE INDEX CONCURRENTLY, некоторые ALTER TABLE)
- SHARE (CREATE INDEX без CONCURRENTLY)
- SHARE ROW EXCLUSIVE (CREATE TRIGGER, некоторые ALTER TABLE, например ADD FOREIGN KEY)
- EXCLUSIVE (REFRESH MATERIALIZED VIEW CONCURRENTLY)
- ACCESS EXCLUSIVE (DROP TABLE, TRUNCATE, VACUUM FULL, CLUSTER, REINDEX, большинство ALTER TABLE; режим по умолчанию у `LOCK TABLE`)

**Таблица конфликтов** — какие режимы не могут одновременно держаться на одной таблице разными транзакциями (второй запрос ждёт):

| Режим | Конфликтует с |
|---|---|
| ACCESS SHARE | ACCESS EXCLUSIVE |
| ROW SHARE | EXCLUSIVE, ACCESS EXCLUSIVE |
| ROW EXCLUSIVE | SHARE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE |
| SHARE UPDATE EXCLUSIVE | SHARE UPDATE EXCLUSIVE, SHARE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE |
| SHARE | ROW EXCLUSIVE, SHARE UPDATE EXCLUSIVE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE |
| SHARE ROW EXCLUSIVE | ROW EXCLUSIVE, SHARE UPDATE EXCLUSIVE, SHARE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE |
| EXCLUSIVE | все, кроме ACCESS SHARE |
| ACCESS EXCLUSIVE | все |

Следствия:
- `SELECT` (ACCESS SHARE) блокируется только `ACCESS EXCLUSIVE`
- Несколько `UPDATE`/`INSERT` (ROW EXCLUSIVE) на одной таблице не конфликтуют между собой на уровне таблицы (конфликты решаются на уровне строк)
- `CREATE INDEX` без `CONCURRENTLY` (SHARE) ждёт все текущие записи и блокирует новые
- `ALTER TABLE`/`TRUNCATE` на живой таблице (ACCESS EXCLUSIVE) встают в очередь за всеми запросами, а все новые запросы встают за ним — таблица «зависает»

#### 2. Optimistic Locking (Оптимистичные блокировки)
Проверяем конфликты при сохранении, не блокируем чтение

```sql
-- Добавляем версию
CREATE TABLE accounts (
    id INT PRIMARY KEY,
    balance DECIMAL,
    version INT DEFAULT 0
);

-- При обновлении
UPDATE accounts 
SET balance = 1000, version = version + 1
WHERE id = 1 AND version = 5;

-- Если affected rows = 0, значит версия изменилась (конфликт)
```

**Laravel реализация:**
```php
class Account extends Model
{
    protected $casts = [
        'version' => 'integer',
    ];
    
    public function incrementBalance($amount)
    {
        $currentVersion = $this->version;
        
        $affected = DB::table('accounts')
            ->where('id', $this->id)
            ->where('version', $currentVersion)
            ->update([
                'balance' => DB::raw('balance + ' . (float) $amount), // приводим тип: иначе SQL-инъекция
                'version' => $currentVersion + 1
            ]);
            
        if ($affected === 0) {
            throw new OptimisticLockException();
        }
    }
}
```

### Deadlocks (Взаимоблокировки)

**Причина:**
```sql
-- TX1
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- TX2
BEGIN;
UPDATE accounts SET balance = balance - 50 WHERE id = 2;
-- TX1
UPDATE accounts SET balance = balance + 100 WHERE id = 2; -- ЖДЕТ TX2
-- TX2
UPDATE accounts SET balance = balance + 50 WHERE id = 1;  -- ЖДЕТ TX1
-- DEADLOCK!
```

**Решения:**
- Упорядоченный доступ к ресурсам (всегда блокировать в порядке возрастания ID)
- Таймауты блокировок
- Deadlock detection и автоматический rollback в СУБД
- Retry логика в приложении

---

## MVCC (Multi-Version Concurrency Control)

### Как работает в PostgreSQL

**Каждая строка имеет:**
- `xmin` - ID транзакции, создавшей версию
- `xmax` - ID транзакции, удалившей/обновившей версию
- tuple visibility rules

**Пример:**
```sql
CREATE TABLE users (id INT, name TEXT);
INSERT INTO users VALUES (1, 'Alice'); -- xmin=100, xmax=0

-- TX 101
UPDATE users SET name = 'Bob' WHERE id = 1;
-- Старая версия: xmin=100, xmax=101
-- Новая версия: xmin=101, xmax=0

-- TX 102 (параллельно TX 101)
SELECT * FROM users WHERE id = 1;
-- Видит 'Alice', потому что TX 101 еще не закоммитилась
```

### Снимок (snapshot) в PostgreSQL: кто виден

Снимок — это не копия данных, а маленькая структура про состояние всей БД в момент взятия:

- `xmin` — id **самой старой ещё активной** транзакции (все id меньше уже завершены)
- `xmax` — следующий ещё не выданный id (всё `>= xmax` стартовало после снимка)
- `xip` — список транзакций, **активных** в момент снимка

Id выдаются по порядку старта, а завершаются в любом порядке. Долгая транзакция «держит» `xmin`, поэтому между `xmin` и `xmax` лежат и активные, и уже завершённые (commit/abort) транзакции.

**Правило видимости изменений транзакции Tn:**

1. `Tn >= xmax` → не видна (стартовала после снимка)
2. `Tn` в `xip` → не видна (шла в момент снимка, даже если потом закоммитится)
3. иначе (`Tn` завершилась до снимка) → смотрим статус в `pg_xact`: **commit → видна, abort → не видна**
4. собственные изменения транзакция видит всегда

**Пример:** в момент снимка T100 идёт, T101 commit, T102 идёт, T103 abort, T104 commit, T105 не стартовала → `xmin=100`, `xmax=105`, `xip={100,102}`. Видны T101 и T104. T103 завершилась, но откатилась. T100 и T102 в `xip`. T105 после `xmax`.

**Видимость версии строки:** `xmin` строки (создатель) виден снимку, а `xmax` строки (удалитель/заменивший) пуст или невиден снимку.

**Когда берётся снимок:**
- Read Committed — новый на каждую команду
- Repeatable Read — один на транзакцию, на первой команде, читающей/пишущей данные (`BEGIN`, `SET`, `SAVEPOINT` не считаются). В InnoDB — на первом обычном `SELECT` (consistent read), `UPDATE`/`FOR UPDATE` снимок не создают
- Снимок относится ко всей БД, а не к конкретным строкам: `UPDATE` сначала находит строки по правилам видимости снимка, потом создаёт новые версии со своим xid
- xid выдаётся лениво, на первой записи; у read-only транзакции xid может не быть, а снимок есть

**Преимущества MVCC:**
- Читатели не блокируют писателей
- Писатели не блокируют читателей
- Consistent reads

**Проблемы MVCC:**
- Bloat (мертвые версии строк)
- Необходимость VACUUM

---

## Практические паттерны

### 1. Безопасное обновление счета
```php
DB::transaction(function () {
    $account = Account::where('id', 1)
        ->lockForUpdate()
        ->first();
    
    if ($account->balance < 100) {
        throw new InsufficientFundsException();
    }
    
    $account->decrement('balance', 100);
});
```

### 2. Обработка Serialization Failure
```php
$maxRetries = 3;
$attempt = 0;

while ($attempt < $maxRetries) {
    try {
        DB::transaction(function () {
            // Ваш код
        }, attempts: 1); // Laravel не будет автоматически ретраить
        
        break; // Успех
    } catch (\Illuminate\Database\QueryException $e) {
        if ($e->getCode() === '40001') { // serialization_failure
            $attempt++;
            if ($attempt >= $maxRetries) {
                throw $e;
            }
            usleep(100000 * 2 ** $attempt); // exponential backoff
        } else {
            throw $e;
        }
    }
}
```

### 3. Distributed Locks с Redis
```php
$lock = Cache::lock('transfer:' . $accountId, 10);

if ($lock->get()) {
    try {
        // Критическая секция
        $this->processTransfer($accountId);
    } finally {
        $lock->release();
    }
} else {
    // Не удалось получить блокировку
    throw new ConcurrentAccessException();
}
```

---

## Savepoints (Точки сохранения)

```sql
BEGIN;
    INSERT INTO accounts (id, balance) VALUES (1, 1000);
    
    SAVEPOINT sp1;
    UPDATE accounts SET balance = 500 WHERE id = 1;
    
    SAVEPOINT sp2;
    DELETE FROM accounts WHERE id = 1;
    
    ROLLBACK TO sp2; -- Откатываем только DELETE
    -- UPDATE остается
    
    ROLLBACK TO sp1; -- Откатываем UPDATE
    -- Остается только INSERT
COMMIT;
```

**В Laravel:**
```php
DB::transaction(function () {
    DB::table('accounts')->insert(['id' => 1, 'balance' => 1000]);
    
    DB::beginTransaction(); // Savepoint
    DB::table('accounts')->where('id', 1)->update(['balance' => 500]);
    DB::rollBack(); // Rollback to savepoint
    
    // INSERT остался, UPDATE откатился
});
```

---

## PostgreSQL специфика

### Transaction ID wraparound
- PostgreSQL использует 32-bit transaction IDs
- После 2 миллиардов транзакций нужен VACUUM
- Autovacuum предотвращает wraparound

### Prepared Transactions (2PC)
```sql
BEGIN;
-- операции
PREPARE TRANSACTION 'trans_id';

-- Позже
COMMIT PREPARED 'trans_id';
-- или
ROLLBACK PREPARED 'trans_id';
```

---

## Ключевые вопросы для интервью

- Чем отличается Repeatable Read в PostgreSQL от MySQL?
- Как MVCC предотвращает блокировки?
- Когда использовать оптимистичные, а когда пессимистичные блокировки?
- Что такое Write Skew и как от него защититься?
- Почему Serializable может быть медленным?
- Как работает deadlock detection?
- Зачем нужны savepoints?
- Что такое transaction ID wraparound в PostgreSQL?

---

## 🎓 Для собеседования: ключевые точки

1. **ACID** - Atomicity (WAL), Consistency (constraints), Isolation (levels), Durability (fsync)
2. **Уровни изоляции** - Read Uncommitted < Read Committed < Repeatable Read < Serializable
3. **Аномалии** - Dirty Read, Non-Repeatable Read, Phantom Read, Lost Update, Write Skew
4. **REPEATABLE READ различия** - PostgreSQL: snapshot isolation, при конфликте записи `40001` (lost update невозможен). MySQL/InnoDB (уровень по умолчанию): UPDATE/FOR UPDATE делают current read, lost update возможен без FOR UPDATE/атомарного апдейта/версии; фантомы закрыты snapshot'ом + next-key/gap locks
5. **MVCC** - PostgreSQL: tuple versioning, MySQL: undo logs. Читатели не блокируют писателей
6. **Pessimistic Locking** - FOR UPDATE/SHARE, блокируем сразу. SKIP LOCKED для очередей
7. **Optimistic Locking** - version field, проверяем при сохранении. Для read-heavy
8. **Deadlock** - упорядочивай доступ к ресурсам (всегда по возрастанию ID), timeout, retry
9. **Serializable** - полная изоляция, serialization failure → retry логика обязательна
10. **Write Skew** - две транзакции читают одно, пишут разное, нарушая бизнес-логику. Защита: Serializable или explicit locking
11. **Savepoints** - частичный rollback (ROLLBACK TO sp1), полезно для nested transactions
12. **Transaction ID wraparound** - PostgreSQL 32-bit TX ID → VACUUM prevent wraparound

**Главное:** Понимай trade-off между isolation level и performance. Используй минимально необходимый уровень. Retry логика обязательна для Serializable.

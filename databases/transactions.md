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
- Деferred constraints

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
| **Repeatable Read** | ❌ Невозможен | ❌ Невозможен | ⚠️ Зависит* | ❌ Невозможен** | ✅ Возможен | ✅ Возможен |
| **Serializable** | ❌ Невозможен | ❌ Невозможен | ❌ Невозможен | ❌ Невозможен | ❌ Невозможен | ❌ Невозможен |

*В PostgreSQL Phantom Read невозможен даже на RR благодаря MVCC  
**В PostgreSQL Lost Update предотвращается на RR

### Read Uncommitted
```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
```
- Минимальная изоляция
- Практически не используется
- В PostgreSQL эквивалентен Read Committed

### Read Committed (по умолчанию в PostgreSQL/MySQL)
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

### Serializable
```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```
- Полная изоляция
- Serialization failures при конфликтах
- SSI (Serializable Snapshot Isolation) в PostgreSQL
- Требует retry логики в приложении

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

Режимы блокировок таблиц (от слабых к сильным):
- ACCESS SHARE (SELECT)
- ROW SHARE (SELECT FOR UPDATE)
- ROW EXCLUSIVE (INSERT, UPDATE, DELETE)
- SHARE UPDATE EXCLUSIVE (VACUUM, некоторые ALTER TABLE)
- SHARE (CREATE INDEX)
- SHARE ROW EXCLUSIVE
- EXCLUSIVE
- ACCESS EXCLUSIVE (DROP, TRUNCATE, VACUUM FULL)

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
                'balance' => DB::raw('balance + ' . $amount),
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
            usleep(100000 * $attempt); // exponential backoff
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
4. **REPEATABLE READ различия** - PostgreSQL предотвращает phantom reads (MVCC), MySQL нет (next-key locks)
5. **MVCC** - PostgreSQL: tuple versioning, MySQL: undo logs. Читатели не блокируют писателей
6. **Pessimistic Locking** - FOR UPDATE/SHARE, блокируем сразу. SKIP LOCKED для очередей
7. **Optimistic Locking** - version field, проверяем при сохранении. Для read-heavy
8. **Deadlock** - упорядочивай доступ к ресурсам (всегда по возрастанию ID), timeout, retry
9. **Serializable** - полная изоляция, serialization failure → retry логика обязательна
10. **Write Skew** - две транзакции читают одно, пишут разное, нарушая бизнес-логику. Защита: Serializable или explicit locking
11. **Savepoints** - частичный rollback (ROLLBACK TO sp1), полезно для nested transactions
12. **Transaction ID wraparound** - PostgreSQL 32-bit TX ID → VACUUM prevent wraparound

**Главное:** Понимай trade-off между isolation level и performance. Используй минимально необходимый уровень. Retry логика обязательна для Serializable.

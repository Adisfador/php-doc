# Нормализация баз данных

Нормальные формы (1NF - 6NF), денормализация, практические примеры для собеседований Middle/Senior.

---

## 📚 Что такое нормализация?

**Нормализация** — процесс организации данных в БД для уменьшения избыточности и зависимости данных.

**Цели нормализации:**
- 🔹 Устранение дублирования данных
- 🔹 Обеспечение целостности данных
- 🔹 Упрощение структуры таблиц
- 🔹 Облегчение поддержки (но не масштабирования: JOIN между шардами усложняют горизонтальное масштабирование)

**Всего существует 6 нормальных форм:**
- 1NF, 2NF, 3NF — основные (используются в 95% случаев)
- BCNF (3.5NF) — усиленная версия 3NF
- 4NF, 5NF, 6NF — редко применяются на практике

---

## 1️⃣ Первая нормальная форма (1NF)

**Определение:** Таблица находится в 1NF, если:
- Все атрибуты атомарны (неделимы)
- Нет повторяющихся групп данных
- Каждая запись уникальна (есть первичный ключ)

### ❌ Нарушение 1NF

```sql
-- Телефоны хранятся в одном поле через запятую
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    phones VARCHAR(255)  -- '123-456, 789-012, 345-678'
);
```

**Проблемы:**
- Невозможно эффективно искать по телефону
- Сложно добавить/удалить отдельный телефон
- Нарушена атомарность данных

### ✅ Соответствие 1NF

```sql
-- Разделяем на две таблицы
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE user_phones (
    id INT PRIMARY KEY,
    user_id INT,
    phone VARCHAR(20),
    FOREIGN KEY (user_id) REFERENCES users(id)
);
```

**Преимущества:**
- Каждый телефон в отдельной строке
- Простой поиск: `WHERE phone = '123-456'`
- Легко добавлять/удалять телефоны

### Пример с продуктами

**До 1NF:**
```sql
CREATE TABLE orders (
    id INT,
    customer VARCHAR(100),
    products VARCHAR(500)  -- 'Laptop, Mouse, Keyboard'
);
```

**После 1NF:**
```sql
CREATE TABLE orders (
    id INT PRIMARY KEY,
    customer_id INT
);

CREATE TABLE order_items (
    id INT PRIMARY KEY,
    order_id INT,
    product_id INT,
    quantity INT,
    FOREIGN KEY (order_id) REFERENCES orders(id)
);
```

---

## 2️⃣ Вторая нормальная форма (2NF)

**Определение:** Таблица в 2NF, если:
- Находится в 1NF
- Все неключевые атрибуты **полностью зависят** от первичного ключа

**Применимо для составных ключей:** Если первичный ключ состоит из нескольких полей, каждый атрибут должен зависеть от **всего ключа**, а не от его части.

### ❌ Нарушение 2NF

```sql
-- Первичный ключ: (student_id, course_id)
CREATE TABLE student_courses (
    student_id INT,
    course_id INT,
    student_name VARCHAR(100),  -- зависит только от student_id
    student_age INT,             -- зависит только от student_id
    course_name VARCHAR(100),    -- зависит только от course_id
    grade VARCHAR(2),            -- зависит от (student_id, course_id)
    PRIMARY KEY (student_id, course_id)
);
```

**Проблема:**
- `student_name` зависит только от `student_id` (частичная зависимость)
- `course_name` зависит только от `course_id` (частичная зависимость)
- Дублирование: имя студента повторяется в каждой записи

### ✅ Соответствие 2NF

```sql
-- Разделяем на три таблицы
CREATE TABLE students (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    age INT
);

CREATE TABLE courses (
    id INT PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE enrollments (
    student_id INT,
    course_id INT,
    grade VARCHAR(2),
    PRIMARY KEY (student_id, course_id),
    FOREIGN KEY (student_id) REFERENCES students(id),
    FOREIGN KEY (course_id) REFERENCES courses(id)
);
```

**Преимущества:**
- Нет дублирования данных
- Изменение имени студента — в одном месте
- Каждая таблица отвечает за одну сущность

---

## 3️⃣ Третья нормальная форма (3NF)

**Определение:** Таблица в 3NF, если:
- Находится в 2NF
- Нет **транзитивных зависимостей** (неключевые атрибуты не зависят от других неключевых)

**Транзитивная зависимость:** A → B → C (C зависит от A через B)

### ❌ Нарушение 3NF

```sql
CREATE TABLE employees (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    department_id INT,
    department_name VARCHAR(100),  -- зависит от department_id, а не от id
    department_location VARCHAR(100)  -- транзитивная зависимость
);
```

**Проблема:**
- `department_name` зависит от `department_id`, а не напрямую от `id`
- Транзитивная цепь: id → department_id → department_name
- При изменении названия отдела нужно обновлять все строки сотрудников

### ✅ Соответствие 3NF

```sql
CREATE TABLE departments (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    location VARCHAR(100)
);

CREATE TABLE employees (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    department_id INT,
    FOREIGN KEY (department_id) REFERENCES departments(id)
);
```

**Преимущества:**
- Название отдела в одном месте
- Изменение отдела не требует обновления множества записей
- Нет транзитивных зависимостей

### Практический пример

**До 3NF:**
```sql
CREATE TABLE orders (
    id INT PRIMARY KEY,
    customer_id INT,
    customer_name VARCHAR(100),
    customer_city VARCHAR(100),
    customer_zip VARCHAR(10),
    city_from_zip VARCHAR(100)  -- зависит от zip, а не от id
);
```

**После 3NF:**
```sql
CREATE TABLE customers (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    zip_code VARCHAR(10)
);

CREATE TABLE zip_codes (
    code VARCHAR(10) PRIMARY KEY,
    city VARCHAR(100)
);

CREATE TABLE orders (
    id INT PRIMARY KEY,
    customer_id INT,
    FOREIGN KEY (customer_id) REFERENCES customers(id)
);
```

---

## 🔺 Нормальная форма Бойса-Кодда (BCNF / 3.5NF)

**Определение:** Таблица в BCNF, если:
- Находится в 3NF
- Для каждой функциональной зависимости X → Y, X является **суперключом**

**Простыми словами:** Более строгая версия 3NF. Устраняет аномалии, которые могут возникнуть при наличии нескольких кандидатов в первичные ключи.

### ❌ Нарушение BCNF (но в 3NF)

```sql
-- Таблица расписания преподавателей
CREATE TABLE schedule (
    student_id INT,
    subject VARCHAR(100),
    teacher VARCHAR(100),
    -- Правила:
    -- 1. Один студент изучает предмет у одного преподавателя
    -- 2. Один преподаватель ведёт только один предмет
    -- Первичный ключ: (student_id, subject)
    -- Но: teacher → subject (преподаватель определяет предмет)
    PRIMARY KEY (student_id, subject)
);
```

**Проблема:**
- `teacher → subject` (преподаватель ведёт только один предмет)
- Но `teacher` не является суперключом
- Аномалия: нельзя добавить преподавателя без студента

### ✅ Соответствие BCNF

```sql
CREATE TABLE teachers (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    subject VARCHAR(100)
);

CREATE TABLE student_schedule (
    student_id INT,
    teacher_id INT,
    PRIMARY KEY (student_id, teacher_id),
    FOREIGN KEY (teacher_id) REFERENCES teachers(id)
);
```

**На практике:** Различие между 3NF и BCNF возникает редко, в большинстве случаев достаточно 3NF. Минус BCNF: декомпозиция может потерять зависимость `(student_id, subject) → teacher` - её уже нельзя проверить без JOIN.

---

## 📋 Высшие нормальные формы (4NF, 5NF, 6NF)

### 4️⃣ Четвёртая нормальная форма (4NF)

**Определение:** Таблица в 4NF, если:
- Находится в BCNF
- Нет **многозначных зависимостей**

**Многозначная зависимость:** Один атрибут определяет множество значений другого независимо от третьего.

**Пример нарушения:**
```sql
-- Сотрудник может владеть несколькими навыками и языками независимо друг от друга
CREATE TABLE employee_skills_languages (
    employee_id INT,
    skill VARCHAR(100),
    language VARCHAR(100)
    -- Если у сотрудника 3 навыка и 2 языка = 6 строк (избыточность)
);
```

**Решение 4NF:**
```sql
CREATE TABLE employee_skills (
    employee_id INT,
    skill VARCHAR(100),
    PRIMARY KEY (employee_id, skill)
);

CREATE TABLE employee_languages (
    employee_id INT,
    language VARCHAR(100),
    PRIMARY KEY (employee_id, language)
);
```

**На практике:** Используется редко, в основном в сложных аналитических системах.

---

### 5️⃣ Пятая нормальная форма (5NF / PJNF)

**Определение:** Таблица в 5NF, если:
- Находится в 4NF
- Нет **зависимостей соединения** (join dependencies)

**Простыми словами:** Таблица не может быть разбита на меньшие таблицы без потери информации при соединении.

**На практике:** Крайне редко применяется. Используется в специализированных хранилищах данных.

---

### 6️⃣ Шестая нормальная форма (6NF)

**Определение:** Таблица в 6NF, если декомпозирована до предела — каждая таблица содержит первичный ключ и не более одного атрибута.

**Применение:**
- Временные базы данных (temporal databases)
- Системы хранения истории изменений
- Специализированные аналитические платформы

**На практике:** Практически не используется в традиционных OLTP-системах из-за чрезмерной сложности.

---

## 🔄 Денормализация

**Денормализация** — сознательное нарушение нормальных форм для улучшения производительности чтения.

### Когда применять?

✅ **Используйте денормализацию:**
- OLAP системы (аналитика, отчёты)
- Кеширование часто используемых данных
- Избежание дорогих JOIN
- Read-heavy нагрузка (много чтения, мало записи)

❌ **Избегайте денормализации:**
- OLTP системы (транзакции)
- Write-heavy нагрузка
- Частые обновления данных
- Критичная консистентность

### Примеры денормализации

```sql
-- Нормализованная версия (3NF)
CREATE TABLE posts (
    id INT PRIMARY KEY,
    title VARCHAR(255),
    user_id INT
);

CREATE TABLE comments (
    id INT PRIMARY KEY,
    post_id INT,
    content TEXT
);

-- Денормализованная версия (кеш для производительности)
CREATE TABLE posts (
    id INT PRIMARY KEY,
    title VARCHAR(255),
    user_id INT,
    comments_count INT  -- дублирование для быстрого доступа
);
```

**Компромисс:**
- ✅ Быстрее: `SELECT comments_count FROM posts WHERE id = 1`
- ❌ Нужно обновлять счётчик при добавлении/удалении комментариев

### Денормализованные поля в Laravel

```php
// Вместо JOIN каждый раз
$posts = Post::with('user', 'comments')->get();

// Создаём денормализованные поля
Schema::table('posts', function (Blueprint $table) {
    $table->string('author_name');  // дублируем из users
    $table->integer('comments_count')->default(0);
});

// Обновляем через события
class PostObserver {
    public function creating(Post $post) {
        // creating (до INSERT), а не created: иначе лишний UPDATE
        $post->author_name = $post->user->name;
    }
}

class CommentObserver {
    public function created(Comment $comment) {
        $comment->post()->increment('comments_count');
    }
    public function deleted(Comment $comment) {
        $comment->post()->decrement('comments_count');
    }
}
```

**Важно:** копию нужно синхронизировать при любом изменении источника. Если пользователь сменит имя, `author_name` в старых постах устареет, поэтому нужен ещё `UserObserver::updated` (или фоновая задача).

### Материализованные представления (PostgreSQL)

Альтернатива ручным полям: БД хранит результат запроса физически, обновляется командой.
```sql
CREATE MATERIALIZED VIEW post_stats AS
SELECT p.id, p.title, COUNT(c.id) AS comments_count
FROM posts p LEFT JOIN comments c ON c.post_id = p.id
GROUP BY p.id, p.title;

CREATE UNIQUE INDEX ON post_stats (id);
REFRESH MATERIALIZED VIEW CONCURRENTLY post_stats;  -- без блокировки чтения
```
Данные устаревают до очередного REFRESH. В MySQL материализованных представлений нет.

### Стратегии денормализации

**1. Счётчики и агрегаты**
```sql
-- Вместо COUNT(*) каждый раз
ALTER TABLE users ADD COLUMN posts_count INT DEFAULT 0;
ALTER TABLE categories ADD COLUMN products_count INT DEFAULT 0;
```

**2. Кеширование вычислений**
```sql
-- Вместо SUM каждый раз
ALTER TABLE orders ADD COLUMN total_amount DECIMAL(10,2);
-- Обновляется триггером при изменении order_items
```

**3. Фиксация исторических данных (snapshot)**
```sql
-- Фиксируем цену на момент заказа
CREATE TABLE order_items (
    id INT PRIMARY KEY,
    product_id INT,
    price DECIMAL(10,2),  -- цена, по которой реально продали
    quantity INT
);
```
Это **не денормализация в строгом смысле**: `order_items.price` - отдельный факт (цена сделки), а не копия `products.price`. Он не должен меняться вслед за каталогом, поэтому избыточности нет. Настоящая денормализация - когда копия обязана совпадать с оригиналом (`author_name`, `comments_count`).

---

## 🎯 Сравнительная таблица

| Форма | Требование | Применение | Как часто нужна |
|-------|-----------|------------|-----------------|
| **1NF** | Атомарные значения, первичный ключ | Все OLTP системы | всегда |
| **2NF** | Нет частичной зависимости | Составные ключи | всегда |
| **3NF** | Нет транзитивных зависимостей | Стандарт для OLTP | обычная цель |
| **BCNF** | Каждая зависимость от суперключа | Сложные схемы | редко |
| **4NF** | Нет многозначных зависимостей | Аналитические системы | редко |
| **5NF** | Нет зависимостей соединения | Специализированные DWH | почти никогда |
| **6NF** | Максимальная декомпозиция | Temporal databases | почти никогда |

---

## 🎓 Ключевые точки для собеседования

### Вопрос 1: Что такое нормализация и зачем она нужна?

**Ответ:**
```
Нормализация — процесс организации данных для устранения избыточности 
и обеспечения целостности. Основные цели:

1. Устранение дублирования данных
2. Упрощение обновлений (один источник истины)
3. Обеспечение консистентности
4. Упрощение структуры таблиц

На практике в 95% случаев достаточно 3NF.
```

### Вопрос 2: Объясните разницу между 2NF и 3NF

**Ответ:**
```
2NF: устраняет частичную зависимость от составного ключа
   Пример: (student_id, course_id) → student_name ❌
   student_id → student_name ✅

3NF: устраняет транзитивные зависимости
   Пример: employee_id → department_id → department_name ❌
   Нужны отдельные таблицы для employees и departments ✅
```

### Вопрос 3: Когда использовать денормализацию?

**Ответ:**
```
Используйте денормализацию для:
✅ OLAP / аналитика (read-heavy)
✅ Кеширование счётчиков (posts_count, likes_count)
✅ Избежание дорогих JOIN

(Цена на момент заказа - не денормализация, а отдельный факт: см. стратегию 3)

Избегайте для:
❌ OLTP транзакции (write-heavy)
❌ Часто обновляемые данные
❌ Критичная консистентность
```

### Вопрос 4: Приведите пример нарушения 1NF

**Ответ:**
```sql
-- ❌ Нарушение 1NF: неатомарные данные
CREATE TABLE users (
    id INT,
    name VARCHAR(100),
    phones VARCHAR(255)  -- '111-222, 333-444, 555-666'
);

-- ✅ Соответствие 1NF
CREATE TABLE users (id INT, name VARCHAR(100));
CREATE TABLE user_phones (
    id INT,
    user_id INT,
    phone VARCHAR(20)
);
```

### Вопрос 5: Что такое BCNF и чем отличается от 3NF?

**Ответ:**
```
BCNF (3.5NF) — усиленная версия 3NF:
- В 3NF: неключевые атрибуты не зависят от других неключевых
- В BCNF: КАЖДАЯ зависимость должна быть от суперключа

Различие проявляется при наличии нескольких потенциальных ключей.

На практике различие встречается редко. В большинстве систем достаточно 3NF.
Минус BCNF: декомпозиция может потерять зависимость (в примере выше
(student_id, subject) → teacher нельзя проверить без JOIN), поэтому
на практике часто останавливаются на 3NF.
```

### Вопрос 6: Как нормализация влияет на производительность?

**Ответ:**
```
Плюсы нормализации:
✅ Быстрее INSERT/UPDATE/DELETE (меньше дублей)
✅ Меньше размер БД (нет дублирования)
✅ Проще поддержка консистентности

Минусы нормализации:
❌ Медленнее SELECT (нужны JOIN)
❌ Сложнее запросы при большой глубине связей

Решение:
- OLTP: нормализация (3NF) + индексы
- OLAP: денормализация + материализованные представления
- Гибрид: 3NF + кеш счётчики
```

### Вопрос 7: Как хранить историю цен в нормализованной БД?

**Ответ:**
```sql
-- Вариант 1: Таблица истории (temporal, с периодом действия)
CREATE TABLE products (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    current_price DECIMAL(10,2)
);

CREATE TABLE price_history (
    id INT PRIMARY KEY,
    product_id INT,
    price DECIMAL(10,2),
    valid_from TIMESTAMP,
    valid_to TIMESTAMP
);

-- Вариант 2: Фиксация цены в заказе (snapshot)
CREATE TABLE order_items (
    id INT PRIMARY KEY,
    product_id INT,
    price DECIMAL(10,2),  -- цена на момент заказа
    quantity INT
);

-- Это не дубль products.price, а отдельный факт (цена сделки), избыточности нет
```

---

## 📚 Практические рекомендации

### Для разработчика

1. **Начинайте с 3NF** для OLTP систем
2. **Используйте денормализацию** только при доказанной необходимости
3. **Измеряйте производительность** перед оптимизацией
4. **Индексы** часто решают проблему без денормализации

### Для архитектора

1. **OLTP → 3NF** (транзакции, консистентность)
2. **OLAP → денормализация** (аналитика, скорость чтения)
3. **Hybrid → CQRS** (разные модели для чтения и записи)

### Золотое правило

> "Нормализуйте пока не упретесь в производительность, 
> потом денормализуйте только то, что действительно тормозит"

---

## 🔗 Связанные разделы

- [Индексы](indexes.md) — ускорение без денормализации
- [Оптимизация](optimization.md) — профилирование запросов
- [Транзакции](transactions.md) — консистентность при обновлениях
- [SQL синтаксис](sql-syntax.md) — практические примеры запросов

# SQL Синтаксис

Полный разбор SQL: SELECT, JOIN, GROUP BY, HAVING, подзапросы, оконные функции, триггеры, процедуры.

---

## 📝 SELECT - Базовый синтаксис

### Простой SELECT

```sql
-- Все колонки
SELECT * FROM users;

-- Конкретные колонки
SELECT id, name, email FROM users;

-- Алиасы
SELECT 
    id AS user_id,
    name AS full_name,
    email AS contact_email
FROM users;

-- DISTINCT - уникальные значения
SELECT DISTINCT city FROM users;
SELECT DISTINCT country, city FROM users;  -- комбинация уникальна
```

### WHERE - Фильтрация

```sql
-- Операторы сравнения
SELECT * FROM users WHERE age > 18;
SELECT * FROM users WHERE age >= 21;
SELECT * FROM users WHERE age < 65;
SELECT * FROM users WHERE age = 30;
SELECT * FROM users WHERE age != 30;  -- или <>

-- Логические операторы
SELECT * FROM users WHERE age > 18 AND country = 'Russia';
SELECT * FROM users WHERE age < 18 OR age > 65;
SELECT * FROM users WHERE NOT is_active;

-- BETWEEN
SELECT * FROM users WHERE age BETWEEN 18 AND 65;  -- включительно
SELECT * FROM orders WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31';

-- IN - список значений
SELECT * FROM users WHERE country IN ('Russia', 'USA', 'Germany');
SELECT * FROM users WHERE id IN (1, 2, 3, 5, 8);

-- LIKE - паттерн поиск
SELECT * FROM users WHERE name LIKE 'John%';      -- начинается с John
SELECT * FROM users WHERE name LIKE '%Doe';       -- заканчивается на Doe
SELECT * FROM users WHERE name LIKE '%Smith%';    -- содержит Smith
SELECT * FROM users WHERE name LIKE 'J_hn';       -- _ = один символ

-- ILIKE - case-insensitive LIKE (только PostgreSQL)
SELECT * FROM users WHERE email ILIKE '%@GMAIL.COM';

-- IS NULL / IS NOT NULL
SELECT * FROM users WHERE deleted_at IS NULL;     -- не удалены
SELECT * FROM users WHERE phone IS NOT NULL;      -- есть телефон
```

### ORDER BY - Сортировка

```sql
-- По возрастанию (по умолчанию)
SELECT * FROM users ORDER BY created_at;
SELECT * FROM users ORDER BY created_at ASC;

-- По убыванию
SELECT * FROM users ORDER BY created_at DESC;

-- Несколько колонок
SELECT * FROM users 
ORDER BY country ASC, age DESC;

-- По алиасу
SELECT 
    name,
    YEAR(CURRENT_DATE) - YEAR(birth_date) AS age
FROM users
ORDER BY age DESC;

-- NULLS FIRST / NULLS LAST (PostgreSQL)
SELECT * FROM users ORDER BY last_login DESC NULLS LAST;
```

### LIMIT и OFFSET - Пагинация

```sql
-- LIMIT - ограничение количества
SELECT * FROM users LIMIT 10;

-- OFFSET - пропустить N записей
SELECT * FROM users LIMIT 10 OFFSET 20;  -- пропустить 20, взять 10

-- Пагинация: страница 3, по 10 на странице
-- offset = (page - 1) * per_page = (3 - 1) * 10 = 20
SELECT * FROM users 
ORDER BY created_at DESC
LIMIT 10 OFFSET 20;

-- MySQL альтернативный синтаксис
SELECT * FROM users LIMIT 20, 10;  -- LIMIT offset, count
```

---

## 🔗 JOIN - Объединение таблиц

### Структура данных для примеров

```sql
-- users
| id | name    | city_id |
|----|---------|---------|
| 1  | Alice   | 1       |
| 2  | Bob     | 2       |
| 3  | Charlie | NULL    |

-- cities
| id | name   |
|----|--------|
| 1  | Moscow |
| 2  | London |
| 3  | Paris  |

-- orders
| id | user_id | amount |
|----|---------|--------|
| 1  | 1       | 100    |
| 2  | 1       | 200    |
| 3  | 2       | 150    |
```

### INNER JOIN

**Возвращает только совпадения из обеих таблиц.**

```sql
SELECT 
    users.name,
    cities.name AS city_name
FROM users
INNER JOIN cities ON users.city_id = cities.id;

-- Результат:
| name  | city_name |
|-------|-----------|
| Alice | Moscow    |
| Bob   | London    |
-- Charlie не вернулся (city_id = NULL)
-- Paris не вернулся (нет users)

-- Алиасы для таблиц
SELECT 
    u.name,
    c.name AS city
FROM users u
INNER JOIN cities c ON u.city_id = c.id;

-- Множественные JOIN
SELECT 
    u.name,
    c.name AS city,
    o.amount
FROM users u
INNER JOIN cities c ON u.city_id = c.id
INNER JOIN orders o ON u.id = o.user_id;
```

### LEFT JOIN (LEFT OUTER JOIN)

**Все записи из левой таблицы + совпадения из правой.**

```sql
SELECT 
    u.name,
    c.name AS city
FROM users u
LEFT JOIN cities c ON u.city_id = c.id;

-- Результат:
| name    | city   |
|---------|--------|
| Alice   | Moscow |
| Bob     | London |
| Charlie | NULL   |  ← Charlie вернулся с NULL
```

### RIGHT JOIN (RIGHT OUTER JOIN)

**Все записи из правой таблицы + совпадения из левой.**

```sql
SELECT 
    u.name,
    c.name AS city
FROM users u
RIGHT JOIN cities c ON u.city_id = c.id;

-- Результат:
| name  | city   |
|-------|--------|
| Alice | Moscow |
| Bob   | London |
| NULL  | Paris  |  ← Paris вернулся без пользователей
```

### FULL OUTER JOIN

**Все записи из обеих таблиц.**

```sql
-- PostgreSQL
SELECT 
    u.name,
    c.name AS city
FROM users u
FULL OUTER JOIN cities c ON u.city_id = c.id;

-- Результат:
| name    | city   |
|---------|--------|
| Alice   | Moscow |
| Bob     | London |
| Charlie | NULL   |
| NULL    | Paris  |

-- ⚠️ MySQL не поддерживает FULL OUTER JOIN!
-- Workaround: UNION LEFT и RIGHT
SELECT u.name, c.name AS city
FROM users u
LEFT JOIN cities c ON u.city_id = c.id
UNION
SELECT u.name, c.name AS city
FROM users u
RIGHT JOIN cities c ON u.city_id = c.id;
```

### CROSS JOIN (Декартово произведение)

**Каждая строка первой таблицы со всеми строками второй.**

```sql
SELECT 
    u.name,
    c.name AS city
FROM users u
CROSS JOIN cities c;

-- Результат: 3 users × 3 cities = 9 строк
| name    | city   |
|---------|--------|
| Alice   | Moscow |
| Alice   | London |
| Alice   | Paris  |
| Bob     | Moscow |
| Bob     | London |
| Bob     | Paris  |
| Charlie | Moscow |
| Charlie | London |
| Charlie | Paris  |

-- Альтернативный синтаксис (без WHERE = CROSS JOIN)
SELECT u.name, c.name FROM users u, cities c;
```

### SELF JOIN

**Таблица объединяется сама с собой.**

```sql
-- employees
| id | name    | manager_id |
|----|---------|------------|
| 1  | Alice   | NULL       |
| 2  | Bob     | 1          |
| 3  | Charlie | 1          |
| 4  | David   | 2          |

-- Найти сотрудников с их менеджерами
SELECT 
    e.name AS employee,
    m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;

-- Результат:
| employee | manager |
|----------|---------|
| Alice    | NULL    |
| Bob      | Alice   |
| Charlie  | Alice   |
| David    | Bob     |
```

---

## 📊 GROUP BY и Агрегатные функции

### Агрегатные функции

```sql
-- COUNT - количество
SELECT COUNT(*) FROM users;                    -- все строки
SELECT COUNT(phone) FROM users;                -- не NULL phone
SELECT COUNT(DISTINCT country) FROM users;     -- уникальные страны

-- SUM - сумма
SELECT SUM(amount) FROM orders;                -- общая сумма заказов

-- AVG - среднее
SELECT AVG(amount) FROM orders;                -- средний чек

-- MIN, MAX - минимум, максимум
SELECT MIN(amount), MAX(amount) FROM orders;

-- Пример: статистика заказов
SELECT 
    COUNT(*) AS total_orders,
    SUM(amount) AS total_revenue,
    AVG(amount) AS avg_order,
    MIN(amount) AS min_order,
    MAX(amount) AS max_order
FROM orders;
```

### GROUP BY - Группировка

```sql
-- Группировка по одной колонке
SELECT 
    country,
    COUNT(*) AS user_count
FROM users
GROUP BY country;

-- Результат:
| country | user_count |
|---------|------------|
| Russia  | 150        |
| USA     | 200        |
| Germany | 75         |

-- Группировка по нескольким колонкам
SELECT 
    country,
    city,
    COUNT(*) AS count
FROM users
GROUP BY country, city
ORDER BY country, city;

-- Сумма заказов по пользователям
SELECT 
    user_id,
    COUNT(*) AS order_count,
    SUM(amount) AS total_spent,
    AVG(amount) AS avg_order
FROM orders
GROUP BY user_id
ORDER BY total_spent DESC;

-- GROUP BY с JOIN
SELECT 
    u.name,
    COUNT(o.id) AS order_count,
    COALESCE(SUM(o.amount), 0) AS total_spent
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name;
```

### Порядок выполнения SELECT (логический)

Порядок, в котором СУБД логически обрабатывает запрос, **не совпадает с порядком написания**:

```
1. FROM / JOIN      - берём таблицы, соединяем (здесь появляются имена и алиасы таблиц)
2. WHERE            - фильтруем строки
3. GROUP BY         - группируем
4. HAVING           - фильтруем группы
5. SELECT           - вычисляем выражения, алиасы колонок, оконные функции
6. DISTINCT         - убираем дубли
7. ORDER BY         - сортируем
8. LIMIT / OFFSET   - отрезаем
```

**Следствия:**
```sql
-- ❌ Алиас из SELECT в WHERE: WHERE выполняется раньше SELECT
SELECT price * qty AS total FROM order_items WHERE total > 100;

-- ✅ Повторить выражение или обернуть в подзапрос/CTE
SELECT price * qty AS total FROM order_items WHERE price * qty > 100;

-- ❌ Агрегат в WHERE: группировки ещё нет
SELECT user_id FROM orders WHERE COUNT(*) > 5 GROUP BY user_id;

-- ✅ HAVING
SELECT user_id FROM orders GROUP BY user_id HAVING COUNT(*) > 5;

-- ❌ Оконная функция в WHERE: считается на шаге SELECT
-- ✅ Подзапрос или CTE, фильтр снаружи
```
- Алиас SELECT можно использовать в `ORDER BY` (он выполняется после SELECT)
- В `GROUP BY` и `HAVING` алиасы зависят от СУБД: MySQL разрешает, PostgreSQL разрешает в `GROUP BY`, но не в `HAVING`
- Это **логический** порядок: реальный план выполнения оптимизатор может менять (например, применять фильтры раньше)

### HAVING - Фильтрация групп

**WHERE фильтрует строки ДО группировки, HAVING - ПОСЛЕ.**

```sql
-- Пользователи с более чем 5 заказами
SELECT 
    user_id,
    COUNT(*) AS order_count
FROM orders
GROUP BY user_id
HAVING COUNT(*) > 5;

-- Страны с более чем 100 пользователями
SELECT 
    country,
    COUNT(*) AS user_count
FROM users
GROUP BY country
HAVING COUNT(*) > 100;

-- Комбинация WHERE и HAVING
-- WHERE сначала фильтрует, потом GROUP BY, потом HAVING
SELECT 
    user_id,
    SUM(amount) AS total_spent
FROM orders
WHERE status = 'completed'       -- WHERE: только completed заказы
GROUP BY user_id
HAVING SUM(amount) > 1000        -- HAVING: только пользователи с total > 1000
ORDER BY total_spent DESC;

-- Пример: активные пользователи (заказ за последние 30 дней) с >3 заказами
SELECT 
    user_id,
    COUNT(*) AS recent_orders,
    SUM(amount) AS total
FROM orders
WHERE created_at > CURRENT_DATE - INTERVAL '30 days'
GROUP BY user_id
HAVING COUNT(*) >= 3
ORDER BY total DESC;
```

---

## 🔍 Подзапросы (Subqueries)

### Scalar Subquery (возвращает одно значение)

```sql
-- В SELECT
SELECT 
    name,
    (SELECT COUNT(*) FROM orders WHERE orders.user_id = users.id) AS order_count
FROM users;

-- В WHERE
SELECT * FROM users
WHERE id = (SELECT user_id FROM orders ORDER BY amount DESC LIMIT 1);

-- Средняя цена выше общей средней
SELECT * FROM products
WHERE price > (SELECT AVG(price) FROM products);
```

### IN Subquery

```sql
-- Пользователи, которые делали заказы
SELECT * FROM users
WHERE id IN (SELECT DISTINCT user_id FROM orders);

-- Продукты, которые никогда не заказывали
SELECT * FROM products
WHERE id NOT IN (SELECT DISTINCT product_id FROM order_items);
```

### EXISTS / NOT EXISTS

**Эффективнее IN для больших таблиц.**

```sql
-- Пользователи с заказами
SELECT * FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id
);

-- Пользователи БЕЗ заказов
SELECT * FROM users u
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id
);

-- EXISTS vs IN:
-- EXISTS - останавливается при первом совпадении (быстрее)
-- IN - проверяет все значения
```

### Correlated Subquery (зависимый подзапрос)

```sql
-- Последний заказ каждого пользователя
SELECT 
    u.name,
    (
        SELECT MAX(created_at) 
        FROM orders o 
        WHERE o.user_id = u.id
    ) AS last_order_date
FROM users u;

-- Пользователи, у которых последний заказ больше средней суммы их заказов
SELECT * FROM users u
WHERE (
    SELECT amount FROM orders 
    WHERE user_id = u.id 
    ORDER BY created_at DESC 
    LIMIT 1
) > (
    SELECT AVG(amount) FROM orders WHERE user_id = u.id
);
```

### Derived Table (подзапрос в FROM)

```sql
-- Подзапрос как таблица
SELECT 
    subquery.country,
    subquery.total_users
FROM (
    SELECT 
        country,
        COUNT(*) AS total_users
    FROM users
    GROUP BY country
) AS subquery
WHERE subquery.total_users > 100;

-- Пагинация с сортировкой по агрегату
SELECT u.*, o.order_count
FROM users u
INNER JOIN (
    SELECT 
        user_id,
        COUNT(*) AS order_count
    FROM orders
    GROUP BY user_id
) o ON u.id = o.user_id
ORDER BY o.order_count DESC
LIMIT 10;
```

---

## 🔀 UNION, INTERSECT, EXCEPT

### UNION - Объединение результатов

```sql
-- UNION - уникальные строки из обоих запросов
SELECT name, email FROM customers
UNION
SELECT name, email FROM suppliers;

-- UNION ALL - все строки (с дубликатами, быстрее)
SELECT name FROM customers
UNION ALL
SELECT name FROM suppliers;

-- ⚠️ Требования:
-- - Одинаковое количество колонок
-- - Совместимые типы данных
-- - Названия берутся из первого SELECT

-- Пример: все транзакции (доходы + расходы)
SELECT 
    'income' AS type,
    amount,
    created_at
FROM income_transactions
UNION ALL
SELECT 
    'expense' AS type,
    amount,
    created_at
FROM expense_transactions
ORDER BY created_at DESC;
```

### INTERSECT - Пересечение

**Только строки, присутствующие в ОБОИХ запросах.**

```sql
-- PostgreSQL, MySQL 8.0.31+
SELECT email FROM customers
INTERSECT
SELECT email FROM newsletter_subscribers;

-- Результат: только emails, которые есть и там, и там
```

### EXCEPT (MINUS в Oracle) - Разность

**Строки из первого запроса, которых НЕТ во втором.**

```sql
-- PostgreSQL
SELECT email FROM customers
EXCEPT
SELECT email FROM newsletter_subscribers;

-- Результат: customers, которые НЕ подписаны на рассылку

-- MySQL workaround (нет EXCEPT до 8.0.31):
SELECT c.email
FROM customers c
LEFT JOIN newsletter_subscribers ns ON c.email = ns.email
WHERE ns.email IS NULL;
```

---

## 📋 CTE - Common Table Expressions (WITH)

**CTE (WITH) - временные именованные результаты запросов.**

### Простой CTE

```sql
-- Базовый синтаксис
WITH cte_name AS (
    SELECT ...
)
SELECT * FROM cte_name;

-- Пример: активные пользователи за последний месяц
WITH active_users AS (
    SELECT * 
    FROM users 
    WHERE is_active = TRUE
      AND created_at > CURRENT_DATE - INTERVAL '1 month'
)
SELECT 
    country,
    COUNT(*) AS user_count
FROM active_users
GROUP BY country;
```

### Множественные CTE

```sql
-- Несколько CTE через запятую
WITH 
    active_users AS (
        SELECT id, name, email 
        FROM users 
        WHERE is_active = TRUE
    ),
    user_orders AS (
        SELECT 
            user_id,
            COUNT(*) AS order_count,
            SUM(amount) AS total_spent
        FROM orders
        GROUP BY user_id
    ),
    high_spenders AS (
        SELECT user_id
        FROM user_orders
        WHERE total_spent > 10000
    )
SELECT 
    u.name,
    u.email,
    uo.order_count,
    uo.total_spent
FROM active_users u
INNER JOIN high_spenders hs ON u.id = hs.user_id
INNER JOIN user_orders uo ON u.id = uo.user_id
ORDER BY uo.total_spent DESC;
```

### CTE vs Подзапросы

```sql
-- С подзапросами (плохо читается)
SELECT 
    u.name,
    o.order_count
FROM users u
INNER JOIN (
    SELECT 
        user_id,
        COUNT(*) AS order_count
    FROM orders
    GROUP BY user_id
) o ON u.id = o.user_id
WHERE u.id IN (
    SELECT user_id 
    FROM orders 
    WHERE amount > 1000
);

-- С CTE (читабельно)
WITH 
    user_orders AS (
        SELECT 
            user_id,
            COUNT(*) AS order_count
        FROM orders
        GROUP BY user_id
    ),
    high_value_users AS (
        SELECT DISTINCT user_id 
        FROM orders 
        WHERE amount > 1000
    )
SELECT 
    u.name,
    uo.order_count
FROM users u
INNER JOIN high_value_users hvu ON u.id = hvu.user_id
INNER JOIN user_orders uo ON u.id = uo.user_id;
```

### Рекурсивный CTE

**Для иерархий, деревьев, графов.**

```sql
-- WITH RECURSIVE (PostgreSQL, MySQL 8.0+)
WITH RECURSIVE category_tree AS (
    -- Base case: корневые категории (якорный запрос)
    SELECT 
        id,
        name,
        parent_id,
        0 AS level,
        ARRAY[id] AS path,          -- PostgreSQL
        CAST(name AS VARCHAR(1000)) AS full_path
    FROM categories
    WHERE parent_id IS NULL
    
    UNION ALL
    
    -- Recursive case: дочерние категории (рекурсивный запрос)
    SELECT 
        c.id,
        c.name,
        c.parent_id,
        ct.level + 1,
        ct.path || c.id,            -- PostgreSQL
        CONCAT(ct.full_path, ' > ', c.name)
    FROM categories c
    INNER JOIN category_tree ct ON c.parent_id = ct.id
    WHERE ct.level < 10  -- защита от бесконечной рекурсии
)
SELECT 
    REPEAT('  ', level) || name AS indented_name,
    level,
    full_path
FROM category_tree
ORDER BY path;

-- Результат:
/*
Electronics              | 0 | Electronics
  Computers              | 1 | Electronics > Computers
    Laptops              | 2 | Electronics > Computers > Laptops
    Desktops             | 2 | Electronics > Computers > Desktops
  Phones                 | 1 | Electronics > Phones
Clothing                 | 0 | Clothing
  Men                    | 1 | Clothing > Men
  Women                  | 1 | Clothing > Women
*/
```

### Рекурсивный CTE: Employee Hierarchy

```sql
-- Организационная структура компании
WITH RECURSIVE employee_hierarchy AS (
    -- Топ-менеджеры (CEO, без руководителей)
    SELECT 
        id,
        name,
        manager_id,
        0 AS level,
        name AS hierarchy_path,
        ARRAY[id] AS id_path
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Подчиненные
    SELECT 
        e.id,
        e.name,
        e.manager_id,
        eh.level + 1,
        eh.hierarchy_path || ' -> ' || e.name,
        eh.id_path || e.id
    FROM employees e
    INNER JOIN employee_hierarchy eh ON e.manager_id = eh.id
)
SELECT 
    id,
    REPEAT('  ', level) || name AS employee,
    level,
    hierarchy_path
FROM employee_hierarchy
ORDER BY id_path;
```

### Рекурсивный CTE: Numbers

```sql
-- Генерация последовательности чисел
WITH RECURSIVE numbers AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM numbers WHERE n < 100
)
SELECT * FROM numbers;

-- Генерация дат (каждый день месяца)
WITH RECURSIVE dates AS (
    SELECT DATE '2024-01-01' AS date
    UNION ALL
    SELECT date + INTERVAL '1 day'
    FROM dates
    WHERE date < '2024-01-31'
)
SELECT 
    date,
    TO_CHAR(date, 'Day') AS day_name
FROM dates;
```

### Рекурсивный CTE: Graph Traversal

```sql
-- Поиск всех друзей друзей (граф связей)
WITH RECURSIVE friend_network AS (
    -- Прямые друзья пользователя 1
    SELECT 
        user_id,
        friend_id,
        1 AS degree,
        ARRAY[user_id, friend_id] AS path
    FROM friendships
    WHERE user_id = 1
    
    UNION
    
    -- Друзья друзей (до 3 степени)
    SELECT 
        f.user_id,
        f.friend_id,
        fn.degree + 1,
        fn.path || f.friend_id
    FROM friendships f
    INNER JOIN friend_network fn ON f.user_id = fn.friend_id
    WHERE fn.degree < 3
      AND NOT (f.friend_id = ANY(fn.path))  -- избегаем циклов
)
SELECT DISTINCT
    friend_id AS user_id,
    degree,
    (
        SELECT name FROM users WHERE id = friend_network.friend_id
    ) AS name
FROM friend_network
ORDER BY degree, user_id;
```

### CTE для DELETE/UPDATE

```sql
-- PostgreSQL: DELETE с CTE и RETURNING
WITH deleted_orders AS (
    DELETE FROM orders
    WHERE created_at < CURRENT_DATE - INTERVAL '1 year'
    RETURNING user_id, amount
)
SELECT 
    user_id,
    COUNT(*) AS deleted_count,
    SUM(amount) AS total_amount
FROM deleted_orders
GROUP BY user_id;

-- UPDATE с CTE
WITH top_customers AS (
    SELECT user_id
    FROM orders
    GROUP BY user_id
    HAVING SUM(amount) > 10000
)
UPDATE users
SET vip_status = TRUE
WHERE id IN (SELECT user_id FROM top_customers);
```

### CTE Materialized Hints (PostgreSQL 12+)

```sql
-- MATERIALIZED - вычислить CTE один раз и сохранить
WITH heavy_computation AS MATERIALIZED (
    SELECT 
        user_id,
        COUNT(*) AS order_count,
        SUM(amount) AS total
    FROM orders
    GROUP BY user_id
)
SELECT * FROM heavy_computation WHERE total > 5000
UNION ALL
SELECT * FROM heavy_computation WHERE order_count > 10;
-- heavy_computation выполнится ОДИН раз

-- NOT MATERIALIZED - выполнять inline (по умолчанию для простых CTE)
WITH simple_filter AS NOT MATERIALIZED (
    SELECT * FROM users WHERE is_active = TRUE
)
SELECT * FROM simple_filter WHERE country = 'Russia';
```

### Преимущества CTE

1. **Читабельность** - разбивка сложного запроса на логические части
2. **Переиспользование** - один CTE можно использовать несколько раз
3. **Рекурсия** - единственный способ для рекурсивных запросов
4. **Модульность** - легко тестировать отдельные части
5. **Maintenance** - проще модифицировать и понимать

### CTE vs Временные таблицы

```sql
-- Временная таблица (сессия)
CREATE TEMP TABLE temp_orders AS
SELECT * FROM orders WHERE amount > 1000;

SELECT * FROM temp_orders;  -- можно использовать много раз

DROP TABLE temp_orders;

-- CTE (только в рамках одного запроса)
WITH filtered_orders AS (
    SELECT * FROM orders WHERE amount > 1000
)
SELECT * FROM filtered_orders;  -- нельзя использовать в другом запросе
```

**Когда CTE:**
- Одноразовое использование в одном запросе
- Рекурсивные запросы
- Улучшение читабельности

**Когда временные таблицы:**
- Множественные запросы к одним данным
- Нужны индексы на промежуточных данных
- Очень большие промежуточные результаты

---

## 🪟 Оконные функции (Window Functions)

**Агрегация БЕЗ GROUP BY - сохраняем все строки.**

### ROW_NUMBER, RANK, DENSE_RANK

```sql
-- ROW_NUMBER - уникальный номер строки
SELECT 
    name,
    salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS row_num
FROM employees;

-- Результат:
| name    | salary | row_num |
|---------|--------|---------|
| Alice   | 150000 | 1       |
| Bob     | 120000 | 2       |
| Charlie | 120000 | 3       |  ← уникальный номер
| David   | 100000 | 4       |

-- RANK - одинаковый ранг для одинаковых значений, пропускает следующие
SELECT 
    name,
    salary,
    RANK() OVER (ORDER BY salary DESC) AS rank
FROM employees;

-- Результат:
| name    | salary | rank |
|---------|--------|------|
| Alice   | 150000 | 1    |
| Bob     | 120000 | 2    |
| Charlie | 120000 | 2    |  ← одинаковый ранг
| David   | 100000 | 4    |  ← пропустил 3

-- DENSE_RANK - одинаковый ранг, НЕ пропускает
SELECT 
    name,
    salary,
    DENSE_RANK() OVER (ORDER BY salary DESC) AS dense_rank
FROM employees;

-- Результат:
| name    | salary | dense_rank |
|---------|--------|------------|
| Alice   | 150000 | 1          |
| Bob     | 120000 | 2          |
| Charlie | 120000 | 2          |
| David   | 100000 | 3          |  ← НЕ пропустил
```

### PARTITION BY - Разбивка на группы

```sql
-- ROW_NUMBER внутри каждого отдела
SELECT 
    department,
    name,
    salary,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank
FROM employees;

-- Результат:
| department | name    | salary | dept_rank |
|------------|---------|--------|-----------|
| IT         | Alice   | 150000 | 1         |
| IT         | Bob     | 120000 | 2         |
| Sales      | Charlie | 100000 | 1         |  ← нумерация с 1 для Sales
| Sales      | David   | 80000  | 2         |

-- Последний заказ каждого пользователя
WITH ranked_orders AS (
    SELECT 
        user_id,
        amount,
        created_at,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
    FROM orders
)
SELECT * FROM ranked_orders WHERE rn = 1;
```

### LAG и LEAD - Доступ к соседним строкам

```sql
-- LAG - предыдущая строка
SELECT 
    date,
    revenue,
    LAG(revenue, 1) OVER (ORDER BY date) AS prev_day_revenue,
    revenue - LAG(revenue, 1) OVER (ORDER BY date) AS diff
FROM daily_stats;

-- Результат:
| date       | revenue | prev_day_revenue | diff  |
|------------|---------|------------------|-------|
| 2024-01-01 | 1000    | NULL             | NULL  |
| 2024-01-02 | 1200    | 1000             | 200   |
| 2024-01-03 | 900     | 1200             | -300  |

-- LEAD - следующая строка
SELECT 
    date,
    revenue,
    LEAD(revenue, 1) OVER (ORDER BY date) AS next_day_revenue
FROM daily_stats;

-- LAG/LEAD с дефолтным значением
SELECT 
    date,
    revenue,
    LAG(revenue, 1, 0) OVER (ORDER BY date) AS prev_day_revenue  -- 0 вместо NULL
FROM daily_stats;
```

### FIRST_VALUE, LAST_VALUE, NTH_VALUE

```sql
-- FIRST_VALUE - первое значение в окне
SELECT 
    name,
    salary,
    FIRST_VALUE(name) OVER (ORDER BY salary DESC) AS highest_paid
FROM employees;

-- LAST_VALUE - последнее значение (осторожно с FRAME!)
SELECT 
    name,
    salary,
    LAST_VALUE(name) OVER (
        ORDER BY salary DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS lowest_paid
FROM employees;

-- NTH_VALUE - N-ое значение
SELECT 
    name,
    salary,
    NTH_VALUE(salary, 2) OVER (ORDER BY salary DESC) AS second_highest_salary
FROM employees;
```

### Агрегатные функции как Window Functions

```sql
-- Running total (бегущая сумма)
SELECT 
    date,
    amount,
    SUM(amount) OVER (ORDER BY date) AS running_total
FROM orders;

-- Результат:
| date       | amount | running_total |
|------------|--------|---------------|
| 2024-01-01 | 100    | 100           |
| 2024-01-02 | 200    | 300           |  ← 100 + 200
| 2024-01-03 | 150    | 450           |  ← 100 + 200 + 150

-- Средняя зарплата по отделу для каждого сотрудника
SELECT 
    department,
    name,
    salary,
    AVG(salary) OVER (PARTITION BY department) AS dept_avg_salary
FROM employees;

-- Moving average (скользящее среднее за 3 дня)
SELECT 
    date,
    revenue,
    AVG(revenue) OVER (
        ORDER BY date
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS moving_avg_3d
FROM daily_stats;
```

### NTILE - Разбить на N групп

```sql
-- Разбить пользователей на 4 квартиля по сумме покупок
SELECT 
    user_id,
    total_spent,
    NTILE(4) OVER (ORDER BY total_spent DESC) AS quartile
FROM user_stats;

-- Результат:
| user_id | total_spent | quartile |
|---------|-------------|----------|
| 1       | 10000       | 1        |  ← топ 25%
| 2       | 8000        | 1        |
| 3       | 5000        | 2        |
| 4       | 3000        | 2        |
| 5       | 1000        | 3        |
| 6       | 500         | 3        |
| 7       | 100         | 4        |  ← нижние 25%
| 8       | 50          | 4        |
```

---

## 🔤 Строковые функции

### Конкатенация

```sql
-- PostgreSQL: ||
SELECT 'Hello' || ' ' || 'World';  -- Hello World

-- MySQL: CONCAT
SELECT CONCAT('Hello', ' ', 'World');  -- Hello World

-- С NULL (PostgreSQL || вернет NULL, CONCAT игнорирует NULL)
SELECT CONCAT('Hello', NULL, 'World');  -- HelloWorld (MySQL)
SELECT 'Hello' || NULL || 'World';      -- NULL (PostgreSQL)

-- CONCAT_WS - с разделителем
SELECT CONCAT_WS(', ', 'Apple', 'Banana', 'Cherry');  -- Apple, Banana, Cherry
```

### SUBSTRING / SUBSTR

```sql
-- SUBSTRING(string, start, length)
SELECT SUBSTRING('Hello World', 1, 5);   -- Hello (PostgreSQL 1-based)
SELECT SUBSTR('Hello World', 1, 5);      -- Hello (MySQL 1-based)

-- Без length - до конца
SELECT SUBSTRING('Hello World', 7);      -- World

-- MySQL: SUBSTRING = SUBSTR (синонимы)
SELECT SUBSTRING('Hello World' FROM 7 FOR 5);  -- World (PostgreSQL синтаксис)
```

### UPPER, LOWER, INITCAP

```sql
-- UPPER - верхний регистр
SELECT UPPER('hello world');  -- HELLO WORLD

-- LOWER - нижний регистр
SELECT LOWER('HELLO WORLD');  -- hello world

-- INITCAP - первая буква заглавная (только PostgreSQL)
SELECT INITCAP('hello world');  -- Hello World
```

### TRIM, LTRIM, RTRIM

```sql
-- TRIM - удалить пробелы с обоих концов
SELECT TRIM('  hello  ');  -- 'hello'

-- LTRIM - слева
SELECT LTRIM('  hello  ');  -- 'hello  '

-- RTRIM - справа
SELECT RTRIM('  hello  ');  -- '  hello'

-- TRIM с конкретным символом
SELECT TRIM('x' FROM 'xxxhelloxxx');  -- 'hello'
```

### LENGTH

```sql
-- LENGTH - длина строки
SELECT LENGTH('Hello');  -- 5

-- CHAR_LENGTH / CHARACTER_LENGTH - то же самое
SELECT CHAR_LENGTH('Hello');  -- 5
```

### REPLACE

```sql
-- REPLACE(string, from, to)
SELECT REPLACE('Hello World', 'World', 'PHP');  -- Hello PHP
```

### POSITION / LOCATE

```sql
-- POSITION - найти позицию подстроки (PostgreSQL)
SELECT POSITION('World' IN 'Hello World');  -- 7

-- LOCATE - MySQL
SELECT LOCATE('World', 'Hello World');  -- 7
```

### LEFT, RIGHT

```sql
-- LEFT - N символов слева
SELECT LEFT('Hello World', 5);  -- Hello

-- RIGHT - N символов справа
SELECT RIGHT('Hello World', 5);  -- World
```

### REVERSE

```sql
-- REVERSE - перевернуть строку
SELECT REVERSE('Hello');  -- olleH
```

### SPLIT_PART (PostgreSQL) / SUBSTRING_INDEX (MySQL)

```sql
-- SPLIT_PART - разбить строку по разделителю (PostgreSQL)
SELECT SPLIT_PART('apple,banana,cherry', ',', 2);  -- banana

-- SUBSTRING_INDEX - MySQL аналог
SELECT SUBSTRING_INDEX('apple,banana,cherry', ',', 2);  -- apple,banana (первые 2)
SELECT SUBSTRING_INDEX('apple,banana,cherry', ',', -1);  -- cherry (последний)
```

---

## 🔢 Математические функции

```sql
-- ABS - модуль
SELECT ABS(-10);  -- 10

-- ROUND - округление
SELECT ROUND(3.14159, 2);  -- 3.14

-- CEIL / CEILING - округление вверх
SELECT CEIL(3.2);  -- 4

-- FLOOR - округление вниз
SELECT FLOOR(3.8);  -- 3

-- POWER - степень
SELECT POWER(2, 10);  -- 1024

-- SQRT - квадратный корень
SELECT SQRT(16);  -- 4

-- MOD - остаток от деления
SELECT MOD(10, 3);  -- 1

-- RANDOM / RAND - случайное число
SELECT RANDOM();  -- PostgreSQL, 0 до 1
SELECT RAND();    -- MySQL, 0 до 1

-- GREATEST, LEAST - максимум, минимум из значений
SELECT GREATEST(10, 20, 5, 30);  -- 30
SELECT LEAST(10, 20, 5, 30);     -- 5
```

---

## 📅 Функции даты и времени

### Текущая дата/время

```sql
-- PostgreSQL
SELECT CURRENT_DATE;           -- 2026-01-28
SELECT CURRENT_TIME;           -- 14:30:00
SELECT CURRENT_TIMESTAMP;      -- 2026-01-28 14:30:00
SELECT NOW();                  -- 2026-01-28 14:30:00

-- MySQL
SELECT CURDATE();              -- 2026-01-28
SELECT CURTIME();              -- 14:30:00
SELECT NOW();                  -- 2026-01-28 14:30:00
SELECT CURRENT_TIMESTAMP;      -- 2026-01-28 14:30:00
```

### Извлечение частей даты

```sql
-- EXTRACT (стандарт SQL)
SELECT EXTRACT(YEAR FROM NOW());   -- 2026
SELECT EXTRACT(MONTH FROM NOW());  -- 1
SELECT EXTRACT(DAY FROM NOW());    -- 28

-- PostgreSQL
SELECT DATE_PART('year', NOW());   -- 2026

-- MySQL
SELECT YEAR(NOW());                -- 2026
SELECT MONTH(NOW());               -- 1
SELECT DAY(NOW());                 -- 28
SELECT HOUR(NOW());                -- 14
SELECT MINUTE(NOW());              -- 30
SELECT SECOND(NOW());              -- 0
```

### Арифметика дат

```sql
-- PostgreSQL (INTERVAL)
SELECT NOW() + INTERVAL '1 day';
SELECT NOW() + INTERVAL '3 hours';
SELECT NOW() - INTERVAL '1 month';
SELECT NOW() + INTERVAL '1 year 2 months 3 days';

-- MySQL (DATE_ADD, DATE_SUB)
SELECT DATE_ADD(NOW(), INTERVAL 1 DAY);
SELECT DATE_ADD(NOW(), INTERVAL 3 HOUR);
SELECT DATE_SUB(NOW(), INTERVAL 1 MONTH);

-- Разница между датами
-- PostgreSQL
SELECT AGE('2026-01-28', '2020-01-01');  -- 6 years 27 days

-- MySQL
SELECT DATEDIFF('2026-01-28', '2020-01-01');  -- 2218 (дней)
SELECT TIMESTAMPDIFF(YEAR, '2020-01-01', '2026-01-28');  -- 6 (лет)
```

### Форматирование дат

```sql
-- PostgreSQL (TO_CHAR)
SELECT TO_CHAR(NOW(), 'YYYY-MM-DD');           -- 2026-01-28
SELECT TO_CHAR(NOW(), 'DD/MM/YYYY HH24:MI:SS');  -- 28/01/2026 14:30:00

-- MySQL (DATE_FORMAT)
SELECT DATE_FORMAT(NOW(), '%Y-%m-%d');         -- 2026-01-28
SELECT DATE_FORMAT(NOW(), '%d/%m/%Y %H:%i:%s');  -- 28/01/2026 14:30:00
```

---

## 🎯 CASE WHEN - Условная логика

```sql
-- Простой CASE
SELECT 
    name,
    age,
    CASE 
        WHEN age < 18 THEN 'Minor'
        WHEN age >= 18 AND age < 65 THEN 'Adult'
        ELSE 'Senior'
    END AS age_group
FROM users;

-- CASE в агрегации
SELECT 
    COUNT(CASE WHEN status = 'active' THEN 1 END) AS active_users,
    COUNT(CASE WHEN status = 'inactive' THEN 1 END) AS inactive_users,
    COUNT(CASE WHEN status = 'banned' THEN 1 END) AS banned_users
FROM users;

-- CASE в ORDER BY
SELECT * FROM products
ORDER BY 
    CASE 
        WHEN stock > 100 THEN 1
        WHEN stock > 0 THEN 2
        ELSE 3
    END,
    name;

-- Простой синтаксис CASE (равенство)
SELECT 
    name,
    CASE status
        WHEN 'active' THEN 'Active User'
        WHEN 'inactive' THEN 'Inactive User'
        ELSE 'Unknown'
    END AS status_label
FROM users;
```

---

## 🗂️ Представления (Views)

### Создание View

```sql
-- Простое представление
CREATE VIEW active_users AS
SELECT id, name, email, created_at
FROM users
WHERE is_active = TRUE;

-- Использование
SELECT * FROM active_users;

-- View с JOIN
CREATE VIEW user_orders AS
SELECT 
    u.id AS user_id,
    u.name,
    o.id AS order_id,
    o.amount,
    o.created_at
FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- View с агрегацией
CREATE VIEW user_stats AS
SELECT 
    u.id,
    u.name,
    COUNT(o.id) AS order_count,
    COALESCE(SUM(o.amount), 0) AS total_spent,
    MAX(o.created_at) AS last_order_date
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name;
```

### OR REPLACE

```sql
-- Создать или заменить
CREATE OR REPLACE VIEW active_users AS
SELECT id, name, email, created_at, updated_at
FROM users
WHERE is_active = TRUE;
```

### Удаление View

```sql
DROP VIEW active_users;
DROP VIEW IF EXISTS active_users;
```

### Materialized Views (только PostgreSQL)

**Materialized View хранит результат физически (быстрее, но нужен REFRESH).**

```sql
-- Создать
CREATE MATERIALIZED VIEW user_stats_mv AS
SELECT 
    u.id,
    u.name,
    COUNT(o.id) AS order_count,
    SUM(o.amount) AS total_spent
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name;

-- Создать с индексом
CREATE UNIQUE INDEX idx_user_stats_mv_id ON user_stats_mv(id);

-- Обновить данные
REFRESH MATERIALIZED VIEW user_stats_mv;

-- Обновить конкурентно (без блокировки SELECT)
REFRESH MATERIALIZED VIEW CONCURRENTLY user_stats_mv;

-- Использование
SELECT * FROM user_stats_mv WHERE order_count > 10;

-- Удалить
DROP MATERIALIZED VIEW user_stats_mv;
```

---

## 🔧 Хранимые процедуры (Stored Procedures)

### PostgreSQL

```sql
-- Создать процедуру
CREATE OR REPLACE PROCEDURE create_user(
    p_name VARCHAR,
    p_email VARCHAR,
    p_password VARCHAR
)
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO users (name, email, password_hash, created_at)
    VALUES (p_name, p_email, p_password, NOW());
    
    -- Вывод сообщения
    RAISE NOTICE 'User % created', p_name;
END;
$$;

-- Вызов
CALL create_user('John Doe', 'john@example.com', 'hashed_password');

-- Процедура с транзакцией
CREATE OR REPLACE PROCEDURE transfer_money(
    p_from_account INT,
    p_to_account INT,
    p_amount DECIMAL
)
LANGUAGE plpgsql
AS $$
BEGIN
    -- Снять деньги
    UPDATE accounts 
    SET balance = balance - p_amount 
    WHERE id = p_from_account;
    
    -- Проверка баланса
    IF (SELECT balance FROM accounts WHERE id = p_from_account) < 0 THEN
        RAISE EXCEPTION 'Insufficient funds';
    END IF;
    
    -- Добавить деньги
    UPDATE accounts 
    SET balance = balance + p_amount 
    WHERE id = p_to_account;
    
    COMMIT;
END;
$$;
```

### MySQL

```sql
-- Создать процедуру
DELIMITER $$

CREATE PROCEDURE create_user(
    IN p_name VARCHAR(255),
    IN p_email VARCHAR(255),
    IN p_password VARCHAR(255)
)
BEGIN
    INSERT INTO users (name, email, password_hash, created_at)
    VALUES (p_name, p_email, p_password, NOW());
    
    SELECT LAST_INSERT_ID() AS user_id;
END$$

DELIMITER ;

-- Вызов
CALL create_user('John Doe', 'john@example.com', 'hashed_password');

-- Процедура с OUT параметром
DELIMITER $$

CREATE PROCEDURE get_user_stats(
    IN p_user_id INT,
    OUT p_order_count INT,
    OUT p_total_spent DECIMAL(10,2)
)
BEGIN
    SELECT 
        COUNT(*),
        COALESCE(SUM(amount), 0)
    INTO p_order_count, p_total_spent
    FROM orders
    WHERE user_id = p_user_id;
END$$

DELIMITER ;

-- Вызов
CALL get_user_stats(1, @count, @total);
SELECT @count, @total;
```

---

## ⚙️ Функции (Functions)

### PostgreSQL

```sql
-- Скалярная функция
CREATE OR REPLACE FUNCTION calculate_discount(
    p_amount DECIMAL,
    p_discount_percent INT
)
RETURNS DECIMAL
LANGUAGE plpgsql
AS $$
DECLARE
    v_discount DECIMAL;
BEGIN
    v_discount := p_amount * p_discount_percent / 100;
    RETURN p_amount - v_discount;
END;
$$;

-- Использование
SELECT calculate_discount(1000, 10);  -- 900

-- Функция, возвращающая таблицу
CREATE OR REPLACE FUNCTION get_user_orders(p_user_id INT)
RETURNS TABLE (
    order_id INT,
    amount DECIMAL,
    created_at TIMESTAMPTZ
)
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY
    SELECT id, amount, created_at
    FROM orders
    WHERE user_id = p_user_id
    ORDER BY created_at DESC;
END;
$$;

-- Использование
SELECT * FROM get_user_orders(1);
```

### MySQL

```sql
-- Скалярная функция
DELIMITER $$

CREATE FUNCTION calculate_discount(
    p_amount DECIMAL(10,2),
    p_discount_percent INT
)
RETURNS DECIMAL(10,2)
DETERMINISTIC
BEGIN
    DECLARE v_discount DECIMAL(10,2);
    SET v_discount = p_amount * p_discount_percent / 100;
    RETURN p_amount - v_discount;
END$$

DELIMITER ;

-- Использование
SELECT calculate_discount(1000, 10);  -- 900
```

---

## 🎬 Триггеры (Triggers)

### PostgreSQL

```sql
-- Функция для триггера
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$;

-- Триггер
CREATE TRIGGER users_updated_at_trigger
BEFORE UPDATE ON users
FOR EACH ROW
EXECUTE FUNCTION update_updated_at_column();

-- AFTER INSERT триггер
CREATE OR REPLACE FUNCTION log_new_user()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO user_logs (user_id, action, created_at)
    VALUES (NEW.id, 'created', NOW());
    RETURN NEW;
END;
$$;

CREATE TRIGGER users_after_insert_trigger
AFTER INSERT ON users
FOR EACH ROW
EXECUTE FUNCTION log_new_user();

-- Удалить триггер
DROP TRIGGER users_updated_at_trigger ON users;
```

### MySQL

```sql
-- BEFORE UPDATE триггер
DELIMITER $$

CREATE TRIGGER users_before_update
BEFORE UPDATE ON users
FOR EACH ROW
BEGIN
    SET NEW.updated_at = NOW();
END$$

DELIMITER ;

-- AFTER INSERT триггер
DELIMITER $$

CREATE TRIGGER users_after_insert
AFTER INSERT ON users
FOR EACH ROW
BEGIN
    INSERT INTO user_logs (user_id, action, created_at)
    VALUES (NEW.id, 'created', NOW());
END$$

DELIMITER ;

-- Удалить триггер
DROP TRIGGER users_before_update;
```

### Типы триггеров

```sql
-- BEFORE INSERT - перед вставкой (можно изменить NEW)
-- AFTER INSERT - после вставки (NEW read-only)
-- BEFORE UPDATE - перед обновлением
-- AFTER UPDATE - после обновления
-- BEFORE DELETE - перед удалением (доступ к OLD)
-- AFTER DELETE - после удаления

-- PostgreSQL также поддерживает:
-- INSTEAD OF - для views
-- TRUNCATE triggers
```

---

## 🔢 Последовательности и Auto-Increment (SEQUENCE)

**Автоматическая генерация уникальных ID для primary keys.**

### PostgreSQL: SEQUENCE

#### Создание SEQUENCE

```sql
-- Базовый SEQUENCE
CREATE SEQUENCE users_id_seq;

-- С параметрами
CREATE SEQUENCE users_id_seq
    START WITH 1
    INCREMENT BY 1
    MINVALUE 1
    MAXVALUE 9223372036854775807
    CACHE 1
    NO CYCLE;

-- Использование с таблицей
CREATE TABLE users (
    id INT DEFAULT nextval('users_id_seq') PRIMARY KEY,
    name VARCHAR(255)
);

-- Или через SERIAL (автоматически создаёт sequence)
CREATE TABLE users (
    id SERIAL PRIMARY KEY,          -- INT AUTO_INCREMENT
    name VARCHAR(255)
);

-- BIGSERIAL для больших значений
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,       -- BIGINT AUTO_INCREMENT
    amount DECIMAL(10,2)
);
```

#### Функции для работы с SEQUENCE

```sql
-- nextval() - получить следующее значение (инкрементирует!)
SELECT nextval('users_id_seq');  -- 1
SELECT nextval('users_id_seq');  -- 2

-- currval() - текущее значение в сессии (без инкремента)
-- ⚠️ Работает только после nextval() в текущей сессии!
SELECT currval('users_id_seq');  -- 2

-- lastval() - последнее значение из любого sequence в сессии
SELECT lastval();  -- 2

-- setval() - установить значение
SELECT setval('users_id_seq', 100);       -- следующее будет 101
SELECT setval('users_id_seq', 100, true); -- следующее будет 101
SELECT setval('users_id_seq', 100, false); -- следующее будет 100
```

#### IDENTITY Columns (PostgreSQL 10+)

**Современная альтернатива SERIAL.**

```sql
-- GENERATED ALWAYS (нельзя вставить значение вручную)
CREATE TABLE products (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name VARCHAR(255)
);

-- Попытка вставить значение вручную - ошибка
INSERT INTO products (id, name) VALUES (1, 'Product');  -- ERROR

-- GENERATED BY DEFAULT (можно переопределить)
CREATE TABLE orders (
    id INT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    amount DECIMAL(10,2)
);

-- Можно вставить значение вручную
INSERT INTO orders (id, amount) VALUES (100, 500.00);  -- OK
INSERT INTO orders (amount) VALUES (300.00);           -- auto-generated id

-- С параметрами
CREATE TABLE users (
    id INT GENERATED ALWAYS AS IDENTITY (
        START WITH 1000
        INCREMENT BY 1
        MINVALUE 1000
        MAXVALUE 999999
        CACHE 20
    ) PRIMARY KEY,
    name VARCHAR(255)
);
```

#### Получение ID вставленной записи

```sql
-- RETURNING (лучший способ)
INSERT INTO users (name, email) 
VALUES ('John', 'john@example.com')
RETURNING id;

-- Результат: { id: 123 }

-- RETURNING * - все колонки
INSERT INTO users (name, email) 
VALUES ('Jane', 'jane@example.com')
RETURNING *;

-- Результат: { id: 124, name: 'Jane', email: 'jane@example.com', created_at: '...' }

-- currval() - в той же транзакции/сессии
INSERT INTO users (name) VALUES ('Bob');
SELECT currval('users_id_seq');  -- 125

-- lastval() - последнее значение
INSERT INTO users (name) VALUES ('Alice');
SELECT lastval();  -- 126
```

#### Управление SEQUENCE

```sql
-- Посмотреть текущее значение (без инкремента)
SELECT last_value FROM users_id_seq;

-- Сбросить sequence
ALTER SEQUENCE users_id_seq RESTART WITH 1;

-- Установить следующее значение на MAX(id) + 1
SELECT setval('users_id_seq', (SELECT MAX(id) FROM users));

-- Изменить параметры
ALTER SEQUENCE users_id_seq INCREMENT BY 2;
ALTER SEQUENCE users_id_seq CACHE 100;

-- Владелец (привязка к таблице)
ALTER SEQUENCE users_id_seq OWNED BY users.id;

-- Удалить sequence
DROP SEQUENCE users_id_seq;

-- Информация о sequence
SELECT * FROM pg_sequences WHERE sequencename = 'users_id_seq';
```

### MySQL: AUTO_INCREMENT

```sql
-- Создание таблицы с AUTO_INCREMENT
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255)
);

-- Или BIGINT для больших значений
CREATE TABLE orders (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    amount DECIMAL(10,2)
);

-- Вставка (id генерируется автоматически)
INSERT INTO users (name) VALUES ('John');
INSERT INTO users (name) VALUES ('Jane');

-- Получить последний вставленный ID
SELECT LAST_INSERT_ID();  -- 2

-- В PHP
$pdo->lastInsertId();  // '2'

-- Установить начальное значение
ALTER TABLE users AUTO_INCREMENT = 1000;

-- Вставить значение вручную (если уникально)
INSERT INTO users (id, name) VALUES (5000, 'Bob');

-- Следующее auto_increment будет 5001
INSERT INTO users (name) VALUES ('Alice');
SELECT LAST_INSERT_ID();  -- 5001

-- Посмотреть текущее значение
SELECT AUTO_INCREMENT 
FROM information_schema.TABLES 
WHERE TABLE_SCHEMA = 'database_name' 
  AND TABLE_NAME = 'users';

-- Сброс AUTO_INCREMENT (осторожно!)
ALTER TABLE users AUTO_INCREMENT = 1;
```

### Получение ID после INSERT

#### PostgreSQL

```sql
-- ✅ RETURNING (best practice)
INSERT INTO users (name, email) 
VALUES ('John', 'john@example.com')
RETURNING id, created_at;

-- ✅ currval() - в той же транзакции
INSERT INTO users (name) VALUES ('Jane');
SELECT currval('users_id_seq');

-- ❌ НЕ ДЕЛАЙ ТАК (race condition!)
INSERT INTO users (name) VALUES ('Bob');
SELECT MAX(id) FROM users;  -- может вернуть ID другого пользователя!
```

#### MySQL

```sql
-- ✅ LAST_INSERT_ID()
INSERT INTO users (name) VALUES ('John');
SELECT LAST_INSERT_ID();

-- В одном запросе (через переменную)
INSERT INTO users (name) VALUES ('Jane');
SET @user_id = LAST_INSERT_ID();
SELECT @user_id;

-- ❌ НЕ ДЕЛАЙ ТАК (race condition!)
INSERT INTO users (name) VALUES ('Bob');
SELECT MAX(id) FROM users;  -- опасно в многопоточной среде!
```

#### Laravel (PHP)

```php
// PostgreSQL с RETURNING
$user = DB::table('users')
    ->insertGetId(['name' => 'John', 'email' => 'john@example.com']);
// $user = 123

// MySQL с LAST_INSERT_ID()
$userId = DB::table('users')->insertGetId([
    'name' => 'John',
    'email' => 'john@example.com'
]);

// Eloquent
$user = User::create(['name' => 'John', 'email' => 'john@example.com']);
echo $user->id;  // auto-populated
```

### Проблемы и подводные камни

#### 1. Gaps (пропуски в ID)

```sql
-- Пропуски это НОРМАЛЬНО!
-- users table:
| id  | name  |
|-----|-------|
| 1   | Alice |
| 2   | Bob   |
| 5   | Carol |  ← пропущены 3 и 4
| 6   | David |

-- Причины пропусков:
-- 1. Rollback транзакции (sequence НЕ откатывается!)
BEGIN;
    INSERT INTO users (name) VALUES ('John');  -- получил id=3
ROLLBACK;  -- запись удалена, но sequence остался на 3

-- 2. Удаление записей
INSERT INTO users (name) VALUES ('Test');  -- id=4
DELETE FROM users WHERE id = 4;

-- 3. Batch операции с ошибками
INSERT INTO users (name) VALUES ('User1'), ('User2');  -- одна упала

-- ⚠️ НЕ ПОЛАГАЙСЯ на последовательность ID без пропусков!
```

#### 2. Race Conditions

```sql
-- ❌ ОПАСНО: две одновременные вставки
-- Thread 1:
INSERT INTO users (name) VALUES ('Alice');
SELECT MAX(id) FROM users;  -- может вернуть ID от Thread 2!

-- Thread 2 (в то же время):
INSERT INTO users (name) VALUES ('Bob');
SELECT MAX(id) FROM users;

-- ✅ ПРАВИЛЬНО: используй RETURNING / LAST_INSERT_ID()
INSERT INTO users (name) VALUES ('Alice') RETURNING id;
```

#### 3. Sequence после импорта данных

```sql
-- Проблема: импортировали данные с id=1..1000
COPY users FROM 'users.csv';

-- Sequence остался на начальном значении (1)
INSERT INTO users (name) VALUES ('New User');  -- ERROR: duplicate key id=1

-- ✅ Решение: обновить sequence
SELECT setval('users_id_seq', (SELECT MAX(id) FROM users));

-- Или для всех таблиц с SERIAL
SELECT setval(
    pg_get_serial_sequence('users', 'id'),
    (SELECT MAX(id) FROM users)
);
```

#### 4. Sequence и транзакции

```sql
-- ⚠️ nextval() НЕ откатывается при ROLLBACK!
BEGIN;
    SELECT nextval('users_id_seq');  -- 1
    SELECT nextval('users_id_seq');  -- 2
    SELECT nextval('users_id_seq');  -- 3
ROLLBACK;

-- Sequence остался на 3!
SELECT currval('users_id_seq');  -- 3
SELECT nextval('users_id_seq');  -- 4

-- Это сделано намеренно для производительности (избежать блокировок)
```

#### 5. UUID как альтернатива

```sql
-- PostgreSQL: UUID вместо SERIAL
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE users (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    name VARCHAR(255)
);

-- Вставка
INSERT INTO users (name) VALUES ('John');

-- Результат:
| id                                   | name |
|--------------------------------------|------|
| 550e8400-e29b-41d4-a716-446655440000 | John |

-- Преимущества:
-- ✅ Нет gaps
-- ✅ Нет race conditions при получении ID
-- ✅ Можно генерировать на клиенте
-- ✅ Нет проблем при merge данных из разных БД

-- Недостатки:
-- ❌ Больше размер (16 bytes vs 4/8 bytes)
-- ❌ Медленнее индексы (случайный порядок)
-- ❌ Сложнее читать и дебажить
```

### Best Practices

```sql
-- ✅ Используй SERIAL/BIGSERIAL или IDENTITY
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,  -- PostgreSQL
    -- или
    id BIGINT AUTO_INCREMENT PRIMARY KEY  -- MySQL
);

-- ✅ Получай ID через RETURNING / LAST_INSERT_ID()
INSERT INTO users (name) VALUES ('John') RETURNING id;

-- ✅ Не полагайся на последовательность без пропусков
-- ID могут быть: 1, 2, 5, 8, 9 (gaps это OK)

-- ✅ После импорта данных обнови sequence
SELECT setval('users_id_seq', (SELECT MAX(id) FROM users));

-- ❌ НЕ используй MAX(id) + 1 для получения ID
-- ❌ НЕ пытайся закрыть gaps в ID
-- ❌ НЕ используй ID как порядковый номер (для этого ROW_NUMBER)

-- ✅ Для больших таблиц используй BIGSERIAL/BIGINT
-- INT max: 2,147,483,647 (~2 billion)
-- BIGINT max: 9,223,372,036,854,775,807 (~9 quintillion)

-- ✅ Для распределённых систем рассмотри UUID
CREATE TABLE distributed_data (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    data TEXT
);
```

### Частые вопросы на собеседованиях

**Q: Почему есть пропуски в ID?**
A: Sequence не откатывается при ROLLBACK транзакций, batch операциях с ошибками, или после удаления записей. Это нормально и сделано для производительности.

**Q: Как получить ID только что вставленной записи?**
A: PostgreSQL - используй `RETURNING id`, MySQL - `LAST_INSERT_ID()`, никогда не используй `MAX(id)`.

**Q: Чем SERIAL отличается от SEQUENCE?**
A: SERIAL - это синтаксический сахар, который автоматически создаёт SEQUENCE и устанавливает DEFAULT для колонки.

**Q: Что произойдёт при ROLLBACK транзакции с INSERT?**
A: Запись будет удалена, но SEQUENCE продолжит с последнего значения (не откатится).

**Q: Когда использовать UUID вместо SERIAL?**
A: В распределённых системах, при необходимости генерировать ID на клиенте, или для избежания gaps.

---

## 🎓 Для собеседования: ключевые точки

1. **JOIN** - INNER (пересечение), LEFT (все слева), RIGHT (все справа), FULL (все)
2. **GROUP BY + HAVING** - группировка и фильтрация групп
3. **Window Functions** - ROW_NUMBER/RANK/LAG/LEAD с PARTITION BY
4. **Подзапросы** - scalar, IN, EXISTS, derived tables
5. **UNION vs UNION ALL** - уникальные vs все (быстрее)
6. **CASE WHEN** - условная логика в SELECT
7. **Views** - виртуальные таблицы, Materialized Views физические (PostgreSQL)
8. **Stored Procedures** - для бизнес-логики
9. **Functions** - возвращают значение, используются в SELECT
10. **Triggers** - автоматические действия при INSERT/UPDATE/DELETE

**Главное:** Понимай разницу между WHERE (до группировки) и HAVING (после), умей использовать оконные функции вместо сложных подзапросов.

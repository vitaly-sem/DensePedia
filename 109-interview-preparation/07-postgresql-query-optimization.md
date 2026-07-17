# PostgreSQL — запросы, планы и оптимизация

---

## EXPLAIN ANALYZE — чтение планов запросов

### Основные операции в плане

```
Seq Scan        — последовательное чтение всей таблицы (плохо для больших таблиц)
Index Scan      — чтение индекса + обращение к таблице (heap fetch)
Index Only Scan — чтение ТОЛЬКО индекса (covering index) — быстро!
Bitmap Scan     — Index Scan + сортировка страниц перед чтением (для многих строк)
Nested Loop     — вложенный цикл (для маленьких таблиц)
Hash Join       — хеш-таблица из одной таблицы, пробы из другой
Merge Join      — слияние двух отсортированных наборов
Sort            — сортировка (использует work_mem или диск)
```

### Как читать EXPLAIN ANALYZE

```sql
EXPLAIN ANALYZE
SELECT u.name, COUNT(o.id) as order_count
FROM users u
JOIN orders o ON o.user_id = u.id
WHERE u.created_at > '2024-01-01'
GROUP BY u.id, u.name
HAVING COUNT(o.id) > 5;
```

```
Пример вывода:

HashAggregate  (cost=1543.00..1568.00 rows=200 width=72)
  Group Key: u.id, u.name
  Filter: (count(o.id) > 5)
  ->  Hash Join  (cost=92.00..1518.00 rows=5000 width=64)
        Hash Cond: (o.user_id = u.id)
        ->  Seq Scan on orders o  (cost=0.00..1230.00 rows=50000 width=8)
        ->  Hash  (cost=67.00..67.00 rows=2000 width=64)
              ->  Seq Scan on users u  (cost=0.00..67.00 rows=2000 width=64)
                    Filter: (created_at > '2024-01-01'::date)

Что значат числа:
  cost=1543.00..1568.00 — оценочная стоимость:
    1543.00 — startup cost (до первой строки)
    1568.00 — total cost (в условных единицах)
  rows=200 — оценка кардинальности (сколько строк вернёт)
  width=72 — средняя ширина строки в байтах
  actual time=... — реальное время (только с ANALYZE)
  rows=... — реальное число строк (только с ANALYZE)
```

### На что смотреть в EXPLAIN ANALYZE

```
1. Seq Scan на большой таблице (>> 1000 строк) — кандидат на индекс
2. Большая разница между estimated rows и actual rows — плохая статистика (ANALYZE)
3. Nested Loop с большим числом строк — должен быть Hash Join
4. Sort с external merge (дисковая сортировка) — увеличить work_mem
5. Bitmap Index Scan + Bitmap Heap Scan — хорошо для выборки >5% таблицы
```

---

## Индексы — типы и применение

### B-tree (основной, по умолчанию)

```
Структура: сбалансированное B-дерево
Применение:
  - =, <, <=, >, >=
  - BETWEEN, IN
  - IS NULL / IS NOT NULL
  - LIKE 'prefix%' (но НЕ LIKE '%substring%')
  - ORDER BY (если порядок совпадает с индексом)

Не работает:
  - LIKE '%middle%' (нет префикса)
  - Функции на индексированной колонке без функционального индекса
  - ILIKE (без pg_trgm)
```

```sql
-- Стандартный B-tree индекс
CREATE INDEX idx_users_email ON users(email);

-- Составной (composite) индекс
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at DESC);

-- Частичный (partial) индекс
CREATE INDEX idx_active_users ON users(id)
    WHERE status = 'active';
```

### Составные индексы — порядок колонок имеет значение

```
Правило: колонки с равенством (=) ПЕРЕД колонками с диапазоном (>, <, BETWEEN)

Хорошо:  WHERE a = 1 AND b = 2 AND c > 10
          INDEX (a, b, c) — используются все 3 колонки

Плохо:   WHERE a = 1 AND c > 10  (без b)
          INDEX (a, b, c) — используется только a (c не может быть использован)

Хорошо:  WHERE a = 1 AND b > 2 AND c = 3
          INDEX (a, c, b) — a и c для =, b для > работает частично

Вывод:   Ставьте колонки с = первыми, колонки с диапазоном — последними.
```

### Покрывающий индекс (Covering Index / Index Only Scan)

```sql
-- Без покрывающего индекса:
-- Index Scan → чтение индекса + random access к таблице (дорого!)

-- С покрывающим индексом:
CREATE INDEX idx_orders_covering ON orders(user_id)
    INCLUDE (created_at, total, status);
-- Index Only Scan — все данные в индексе, к таблице не ходим!

-- Проверить, что индекс покрывающий:
EXPLAIN ANALYZE SELECT user_id, created_at, total, status
FROM orders WHERE user_id = 123;
-- Должен показать: Index Only Scan using idx_orders_covering
```

### GIN (Generalized Inverted Index)

```
Применение:
  - JSONB: операторы ?, ?|, ?&, @>, <@
  - Полнотекстовый поиск (tsvector/tsquery)
  - Массивы (@>, &&)
  - pg_trgm: LIKE '%substring%' и ILIKE

Структура: инвертированный индекс — ключ → список позиций
```

```sql
-- JSONB индекс
CREATE INDEX idx_products_data ON products USING GIN (data jsonb_path_ops);

-- Запрос использует индекс:
SELECT * FROM products WHERE data @> '{"category": "electronics"}';

-- Полнотекстовый поиск
CREATE INDEX idx_posts_fts ON posts
    USING GIN (to_tsvector('english', body));

SELECT * FROM posts
WHERE to_tsvector('english', body) @@ to_tsquery('english', 'postgresql & performance');
```

### GiST (Generalized Search Tree)

```
Применение:
  - Геометрические данные (PostGIS)
  - Полнотекстовый поиск (альтернатива GIN)
  - Диапазоны (range types)
  - KNN поиск (оператор <->)
```

### BRIN (Block Range INdex)

```
Применение: 
  - Огромные таблицы (миллиарды строк)
  - Данные коррелируют с физическим порядком (например, created_at)
  
Плюсы: Очень маленький размер (0.1% от таблицы)
Минусы: Менее точный, чем B-tree
```

```sql
CREATE INDEX idx_events_created ON events USING BRIN (created_at)
    WITH (pages_per_range = 32);
```

### Hash Index

```
Применение: ТОЛЬКО для = (равенство)
Плюсы: Быстрее B-tree для = на больших ключах
Минусы: Не поддерживает range queries, не WAL-logged до PG 10
```

---

## Транзакции и MVCC

### Уровни изоляции в PostgreSQL

```
PostgreSQL поддерживает 3 уровня (Read Uncommitted = Read Committed):

Read Committed (по умолчанию):
  - Видит только зафиксированные данные
  - Non-repeatable read: возможен (другой UPDATE зафиксирован между чтениями)
  - Phantom read: возможен (другой INSERT зафиксирован между чтениями)

Repeatable Read:
  - Снимок данных на момент начала транзакции
  - Non-repeatable read: невозможен
  - Phantom read: невозможен в PostgreSQL (благодаря SI — Snapshot Isolation)
  - Serialization failure: возможен при конфликте обновлений

Serializable:
  - Полная изоляция через SSI (Serializable Snapshot Isolation)
  - При конфликте: ERROR: could not serialize access
  - Нужен retry логика в приложении
```

### MVCC — как работает

```
Каждая строка содержит:
  - xmin: ID транзакции, создавшей эту версию
  - xmax: ID транзакции, удалившей эту версию (или 0)

При UPDATE:
  - Старая версия: xmax = текущая транзакция
  - Новая версия: xmin = текущая транзакция
  - Старая версия остаётся до VACUUM!

Следствия MVCC:
  - Читатели НЕ блокируют писателей (и наоборот)
  - UPDATE создаёт новую версию строки (bloat)
  - VACUUM очищает старые версии
  - Transaction ID wrap-around — критическая проблема
```

```
Q: Что такое Transaction ID Wrap-Around?
A: xid — 32-битный счётчик (~4 млрд). Когда счётчик подходит к пределу,
   старые замороженные xid "оборачиваются" и кажутся новыми.
   VACUUM FREEZE помечает старые строки как замороженные.
   Без этого БД останавливается!
```

### Блокировки

```sql
-- Посмотреть текущие блокировки
SELECT blocked.query AS blocked_query,
       blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_locks bl ON bl.pid = blocked.pid AND bl.granted = false
JOIN pg_locks bk ON bk.locktype = bl.locktype
    AND bk.database = bl.database
    AND bk.relation = bl.relation
    AND bk.pid != bl.pid
    AND bk.granted = true
JOIN pg_stat_activity blocking ON blocking.pid = bk.pid;

-- Deadlock example:
-- Tx1: UPDATE users SET balance = balance - 100 WHERE id = 1;  -- блокирует id=1
-- Tx2: UPDATE users SET balance = balance - 50 WHERE id = 2;   -- блокирует id=2
-- Tx1: UPDATE users SET balance = balance + 50 WHERE id = 2;   -- ждёт Tx2
-- Tx2: UPDATE users SET balance = balance + 100 WHERE id = 1;  -- ждёт Tx1 → DEADLOCK!
-- PostgreSQL обнаружит и убьёт одну транзакцию
```

### VACUUM и ANALYZE

```
VACUUM:
  - Очищает dead tuples (старые версии строк)
  - Обновляет visibility map (для Index Only Scan)
  - Обновляет freeze map
  - Не блокирует таблицу (concurrent)

VACUUM FULL:
  - Полная перезапись таблицы (освобождает место на диске)
  - БЛОКИРУЕТ таблицу (ACCESS EXCLUSIVE lock)

ANALYZE:
  - Собирает статистику для планировщика (pg_stats)
  - Не блокирует таблицу

AUTOVACUUM:
  - Автоматически запускает VACUUM и ANALYZE
  - Настраивается: autovacuum_vacuum_scale_factor, autovacuum_analyze_scale_factor
```

---

## Оптимизация запросов — практические приёмы

### 1. Избегайте SELECT \*

```sql
-- ❌ Плохо: читает все колонки, включая большие TEXT/BLOB
SELECT * FROM orders WHERE user_id = 123;

-- ✅ Хорошо: только нужные колонки
SELECT id, created_at, total FROM orders WHERE user_id = 123;
```

### 2. Правильные индексы для запроса

```sql
-- Запрос:
SELECT * FROM orders
WHERE user_id = 123 AND status = 'pending' AND created_at > '2024-01-01'
ORDER BY created_at DESC
LIMIT 20;

-- Идеальный индекс:
CREATE INDEX idx_orders_opt ON orders(user_id, status, created_at DESC);
-- user_id = 123 (равенство) → используется
-- status = 'pending' (равенство) → используется
-- created_at > '...' (диапазон) → используется
-- ORDER BY created_at DESC → используется (уже сортирован!)
-- LIMIT 20 → остановится после 20 строк (не читает всё!)
```

### 3. JOIN порядок и типы

```sql
-- Nested Loop хорош когда:
--   - Одна таблица маленькая (< 1000 строк)
--   - Индекс на колонке JOIN второй таблицы

-- Hash Join хорош когда:
--   - Обе таблицы большие
--   - Нет индекса на колонке JOIN

-- Проверить какой JOIN использован:
EXPLAIN ANALYZE SELECT * FROM orders o
JOIN users u ON u.id = o.user_id
WHERE o.created_at > NOW() - INTERVAL '1 week';
```

### 4. Избегайте функций на индексированных колонках

```sql
-- ❌ Плохо: индекс по created_at не используется
SELECT * FROM orders WHERE DATE(created_at) = '2024-01-15';

-- ✅ Хорошо: range query по индексу
SELECT * FROM orders
WHERE created_at >= '2024-01-15' AND created_at < '2024-01-16';

-- Или функциональный индекс:
CREATE INDEX idx_orders_date ON orders ((DATE(created_at)));
```

---

## Партиционирование

```sql
-- Декларативное партиционирование (PG 10+)
CREATE TABLE orders (
    id BIGSERIAL,
    user_id BIGINT,
    total DECIMAL,
    created_at TIMESTAMPTZ NOT NULL
) PARTITION BY RANGE (created_at);

-- Партиции по месяцам
CREATE TABLE orders_2024_01 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE orders_2024_02 PARTITION OF orders
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Partition pruning: запрос только к нужной партиции
SELECT * FROM orders WHERE created_at = '2024-01-15';
-- Планировщик прочитает ТОЛЬКО orders_2024_01
```

---

## Чек-лист: PostgreSQL на собеседовании

- [ ] Типы индексов: B-tree, Hash, GIN, GiST, BRIN — когда какой?
- [ ] Как читать EXPLAIN ANALYZE? Seq Scan vs Index Scan vs Index Only Scan
- [ ] Составные индексы: порядок колонок
- [ ] Покрывающие индексы (INCLUDE) для Index Only Scan
- [ ] Уровни изоляции транзакций: Read Committed vs Repeatable Read vs Serializable
- [ ] MVCC: как работает, что такое dead tuples, зачем VACUUM
- [ ] VACUUM vs VACUUM FULL vs ANALYZE
- [ ] Приёмы оптимизации: избегать SELECT *, функции на колонках, правильный порядок JOIN
- [ ] Партиционирование: когда и как
- [ ] Connection Pooling (PgBouncer/pgpool-II)

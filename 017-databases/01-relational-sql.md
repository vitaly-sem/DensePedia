# Реляционные базы данных и SQL

---

## ACID

| Свойство | Описание | Пример нарушения |
|----------|----------|-----------------|
| **Atomicity** | Транзакция выполняется полностью или не выполняется | Перевод денег: списалось, но не зачислилось |
| **Consistency** | Данные в согласованном состоянии после транзакции | Нарушение constraints / foreign keys |
| **Isolation** | Конкурентные транзакции не влияют друг на друга | Dirty read, non-repeatable read, phantom read |
| **Durability** | Зафиксированные данные не теряются | После сбоя питания данные сохранены |

### Isolation Levels (ANSI SQL)

```
                        Dirty Read  Non-Repeatable  Phantom Read
Read Uncommitted         ❌ 가능      ❌ 가능          ❌ 가능
Read Committed           ✅          ❌ 가능          ❌ 가능
Repeatable Read          ✅          ✅              ❌ 가능
Serializable             ✅          ✅              ✅
```

### PostgreSQL Default: Read Committed

```sql
-- Уровни изоляции в PostgreSQL
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

---

## Нормализация

### Нормальные формы

| НФ | Правило | Нарушение |
|----|---------|-----------|
| **1NF** | Каждая ячейка — атомарное значение | В ячейке список через запятую |
| **2NF** | 1NF + нет частичной зависимости от составного ключа | Заказ → товар: цена товара зависит только от товара |
| **3NF** | 2NF + нет транзитивной зависимости | Сотрудник → отдел → адрес отдела |

### Пример денормализации

```sql
-- Нормализовано (3NF)
orders (id, user_id, created_at)
order_items (id, order_id, product_id, quantity)
products (id, name, price)

-- Денормализовано (для read-heavy сценариев)
orders (id, user_id, created_at, total_amount, item_count)
```

**Факт:** В реальных проектах баланс — **нормализация до 3NF**, денормализация только по необходимости (performance).

---

## Индексы

### B-tree (стандартный)

```
              [50]
            /      \
        [20,30]    [70,80]
       /  |   \    /  |   \
```

**Когда использовать:** Equality (`WHERE id = 5`) и range (`WHERE age > 18`).

### Hash Index

```sql
CREATE INDEX idx_users_email ON users USING HASH (email);
```

**Когда:** Только equality (`=`) — не для range или sorting.

### Covering Index (покрывающий)

```sql
-- Все поля запроса в индексе — не нужно читать таблицу
CREATE INDEX idx_covering ON orders (user_id, status, created_at);
```

### Composite Index — порядок важен!

```sql
-- Для запроса: WHERE user_id = 5 AND status = 'active'
CREATE INDEX idx_covering ON orders (user_id, status);
-- Для запроса: WHERE status = 'active' (без user_id) — index не используется
```

**Правило:** Левый префикс — столбцы слева направо, ищите WHERE по левым столбцам.

### EXPLAIN — читать планы запросов

```sql
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE user_id = 42 AND status = 'active'
ORDER BY created_at DESC;
```

```
Sort  (cost=... rows=...)
  └── Index Scan using idx_orders_user_status on orders
        Index Cond: ((user_id = 42) AND (status = 'active'))
```

---

## JOIN'ы

```
INNER JOIN  = только пересечение
LEFT JOIN   = все из левой + пересечение
RIGHT JOIN  = все из правой + пересечение
FULL JOIN   = всё из обеих
CROSS JOIN  = декартово произведение
```

### Hash Join vs Nested Loop vs Merge Join

| Метод | Когда | Сложность |
|-------|-------|-----------|
| **Nested Loop** | Малая таблица × большая | O(N × M) — но с index = O(N log M) |
| **Hash Join** | Две большие таблицы | O(N + M) — хеширование |
| **Merge Join** | Отсортированные данные | O(N + M) — без хеша |

---

## Оконные функции (Window Functions)

```sql
SELECT 
  employee_id,
  department_id,
  salary,
  RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) as rank_in_dept,
  AVG(salary) OVER (PARTITION BY department_id) as avg_dept_salary,
  LAG(salary, 1) OVER (PARTITION BY employee_id ORDER BY date) as prev_salary,
  SUM(salary) OVER (ORDER BY date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) as running_total
FROM employees;
```

| Функция | Назначение |
|---------|------------|
| `ROW_NUMBER()` | Номер строки в партиции |
| `RANK()` / `DENSE_RANK()` | Ранжирование с пропусками/без |
| `LAG()` / `LEAD()` | Предыдущая/следующая строка |
| `FIRST_VALUE()` / `LAST_VALUE()` | Первое/последнее значение в окне |

---

## Оптимизация запросов

### Проблемы производительности

| Проблема | Причина | Решение |
|----------|---------|---------|
| **Full table scan** | Нет индекса для WHERE | Добавить индекс |
| **N+1 queries** | ORM ленивая загрузка | Eager loading, JOIN |
| **SELECT *** | Ненужные данные | Только нужные колонки |
| **Неявный CAST** | `WHERE id = '42'` (id — int) | Явный тип |
| **OR в WHERE** | Не использует составной индекс | UNION ALL или два запроса |

### N+1 Problem

```csharp
// ❌ N+1: 1 запрос на заказы + N запросов на items
var orders = db.Orders.ToList();
foreach (var order in orders)
{
    Console.WriteLine(order.Items.Count); // каждый раз SQL!
}

// ✅ Исправление: Eager Loading
var orders = db.Orders.Include(o => o.Items).ToList();
```

---

## Транзакции и блокировки

### Pessimistic vs Optimistic

```sql
-- Pessimistic lock (блокируем строку)
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;
```

```csharp
// Optimistic locking (EF Core)
var account = db.Accounts.First(a => a.Id == 1);
account.Balance -= 100;
account.RowVersion++;  // concurrency token
await db.SaveChangesAsync(); // DbUpdateConcurrencyException если конфликт
```

### Deadlock

```
T1: UPDATE A → UPDATE B
T2: UPDATE B → UPDATE A
       → DEADLOCK! (одна транзакция будет убита)
```

**Профилактика:** Всегда блокировать ресурсы в одном порядке (например, по ID).

---

## Чек-лист

- [ ] Индексы покрывают WHERE, JOIN, ORDER BY
- [ ] EXPLAIN ANALYZE выполнен для сложных запросов
- [ ] N+1 проблема обнаружена и исправлена
- [ ] Уровень изоляции соответствует требованиям
- [ ] Connection pooling настроен (не открывать/закрывать соединения вручную)
- [ ] Транзакции короткие (не держать блокировки долго)
- [ ] Миграции данных тестированы на production-объёме
- [ ] SELECT * заменён на конкретные колонки

# PostgreSQL Advanced — шардирование, репликация, тонкости

---

## Репликация в PostgreSQL

### Streaming Replication (физическая)

```
Архитектура:

┌─────────────────┐     WAL stream     ┌──────────────────┐
│   PRIMARY        │ ─────────────────→│   STANDBY         │
│   (read/write)   │                   │   (read-only)     │
└────────┬────────┘                    └──────────────────┘
         │                              ┌──────────────────┐
         └──────────────────────────────│   STANDBY 2       │
                WAL stream              │   (read-only)     │
                                        └──────────────────┘

WAL (Write-Ahead Log):
  - Все изменения сначала пишутся в WAL (16MB сегменты)
  - Streaming: primary отправляет WAL на standby в реальном времени
  - Standby применяет WAL (recovery process)

Режимы репликации:
  - Асинхронная (по умолчанию): primary не ждёт standby
  - Синхронная: primary ждёт подтверждения от standby
```

### Синхронная vs Асинхронная репликация

```sql
-- Настройка синхронной репликации (postgresql.conf на primary):
synchronous_commit = remote_apply  -- или on, remote_write
synchronous_standby_names = 'ANY 2 (standby1, standby2, standby3)'
-- ANY 2: ждём любые 2 standby из списка
-- FIRST 2: ждём первые 2 standby (по порядку)

-- Уровни synchronous_commit:
-- off          — не ждать записи WAL вообще (потеря транзакции при краше)
-- local        — ждать записи WAL на primary (не на standby)
-- remote_write — ждать записи WAL на диск standby (не применения)
-- on           — ждать применения WAL на standby (но не flush)
-- remote_apply — ждать ПРИМЕНЕНИЯ WAL на standby → read-your-writes!
```

```
Синхронная:
  + Нет потери данных при отказе primary
  + Read-your-writes гарантия (remote_apply)
  - Latency записи = RTT до standby + запись WAL
  - При падении standby → primary БЛОКИРУЕТСЯ

Асинхронная:
  + Минимальная latency записи
  + Primary не блокируется
  - Возможна потеря данных (lag между primary и standby)
```

### Logical Replication

```
Отличие от Streaming:
  - Streaming: побайтовая репликация WAL (физическая)
  - Logical: репликация на уровне строк/таблиц (логическая)

Применение:
  - Выборочная репликация (не все таблицы)
  - Разные версии PostgreSQL на primary и standby
  - Миграция с минимальным downtime
  - Репликация между разными БД (multi-master частично)

Настройка:
  wal_level = logical

CREATE PUBLICATION pub_orders FOR TABLE orders, order_items;

-- На standby:
CREATE SUBSCRIPTION sub_orders
    CONNECTION 'host=primary dbname=mydb'
    PUBLICATION pub_orders;
```

---

## Шардирование PostgreSQL

### Подходы к шардированию

```
1. Ручное шардирование (Application-level):
   Приложение выбирает БД по ключу шарда

2. Нативное партиционирование + Foreign Tables (postgres_fdw):
   Партиции на разных серверах через FDW

3. Citus (распределённый PostgreSQL):
   Open-source расширение для горизонтального масштабирования

4. Мульти-мастер (BDR, pgEdge):
   Несколько пишущих узлов, разрешение конфликтов
```

### Citus — распределённый PostgreSQL

```
Архитектура:

┌─────────────────────────────────────────────────────┐
│                    CITUS CLUSTER                     │
│                                                     │
│  ┌───────────────┐                                  │
│  │  Coordinator  │  (маршрутизирует запросы)        │
│  └───────┬───────┘                                  │
│          │                                          │
│  ┌───────┼───────┬──────────────┐                  │
│  ▼       ▼       ▼              ▼                  │
│ ┌──────┐┌──────┐┌──────┐  ┌──────┐                │
│ │Worker││Worker││Worker│  │Worker│                │
│ │  1   ││  2   ││  3   │  │  4   │                │
│ │shard ││shard ││shard │  │shard │                │
│ │ 1,5  ││ 2,6  ││ 3,7  │  │ 4,8  │                │
│ └──────┘└──────┘└──────┘  └──────┘                │
│                                                     │
│  Таблица распределяется по колонке:                 │
│  SELECT create_distributed_table('orders', 'user_id');│
└─────────────────────────────────────────────────────┘

Два типа таблиц:
  - Distributed: шардированные по ключу (большие)
  - Reference: реплицированы на все worker'ы (маленькие, справочники)
```

### Стратегии шардирования

```
1. Hash Sharding:
   shard = hash(key) % shard_count
   + Равномерное распределение
   - Range queries страдают (запрос ко всем шардам)

2. Range Sharding:
   shard зависит от диапазона ключа (например, user_id: 0-1M, 1M-2M...)
   + Range queries эффективны (только нужный шард)
   - Hotspots (один шард перегружен)

3. Directory-based Sharding:
   Отдельная таблица/сервис: key → shard
   + Гибкость (перемещение между шардами)
   - Дополнительный lookup

4. Geo-sharding:
   Шардирование по географическому региону
   + Data locality (GDPR compliance)
   - Неравномерная нагрузка
```

---

## Connection Pooling — PgBouncer

### Проблема нативных подключений

```
PostgreSQL процесс на connection:
  - Каждый connection = отдельный процесс ОС (fork)
  - Память: ~2-10 MB на connection
  - 1000 connections = 2-10 GB RAM только на процессы
  - Context switching: планировщик ОС переключает процессы
  - max_connections обычно ≤ 200-500

Решение: PgBouncer — пул соединений
```

### PgBouncer — режимы

```
┌──────────┐     ┌──────────┐     ┌──────────────┐
│  App     │────→│ PgBouncer│────→│  PostgreSQL   │
│ (1000    │     │ (пул)    │     │  (30          │
│  клиентов)│     │          │     │   connections)│
└──────────┘     └──────────┘     └──────────────┘

Режимы:

1. Session pooling:
   - Connection закреплён за клиентом на всё время сессии
   - Как прямой коннект к PG, но с пулом
   - Подходит: когда нужны prepared statements, SET, LISTEN/NOTIFY

2. Transaction pooling:
   - Connection возвращается в пул после COMMIT/ROLLBACK
   - Клиент может получить другой connection при следующей транзакции
   - Подходит: веб-приложения, stateless
   - Ограничения: нельзя использовать SET, LISTEN/NOTIFY, prepared statements

3. Statement pooling:
   - Connection возвращается после каждого statement
   - Максимальная утилизация, но самые жёсткие ограничения
```

```ini
# pgbouncer.ini
[databases]
mydb = host=localhost port=5432 dbname=mydb

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = scram-sha-256
pool_mode = transaction          # session | transaction | statement
default_pool_size = 25           # connections к PG на пару user/database
max_client_conn = 1000           # максимум клиентских подключений
reserve_pool_size = 5            # резерв если пул исчерпан
reserve_pool_timeout = 3         # секунд ожидания резервного connection
```

---

## JSONB — продвинутое использование

### GIN индексы для JSONB

```sql
-- Стандартный GIN (операторы ?, ?|, ?&, @>)
CREATE INDEX idx_products ON products USING GIN (data);

-- Более быстрый и компактный (ТОЛЬКО @>)
CREATE INDEX idx_products_path ON products USING GIN (data jsonb_path_ops);

-- Разница размеров:
-- GIN: индексирует ключи + значения → больше, но больше операторов
-- jsonb_path_ops: только значения → компактнее, быстрее @>
```

### JSONB + реляционные колонки

```sql
-- Гибридный подход: часто используемые поля как колонки + JSONB для гибких данных
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    total DECIMAL,
    status VARCHAR(20),
    extra JSONB  -- гибкие поля: delivery_notes, metadata, context
);

-- Индекс на конкретный JSONB путь
CREATE INDEX idx_orders_extra_priority ON orders
    ((extra->>'priority'));

-- Запрос по JSONB полю:
SELECT * FROM orders
WHERE extra->>'priority' = 'high';

-- Частичный индекс для JSONB:
CREATE INDEX idx_high_priority ON orders (id)
    WHERE extra->>'priority' = 'high';
```

---

## Полезные расширения PostgreSQL

| Расширение | Назначение |
|-----------|------------|
| `pg_stat_statements` | Статистика выполнения запросов (находим медленные) |
| `pg_trgm` | Триграммные индексы для LIKE '%середина%' и ILIKE |
| `postgis` | Геопространственные данные (точки, полигоны, расстояния) |
| `pg_cron` | Планировщик задач внутри PG (Cron-like) |
| `pg_partman` | Автоматическое управление партициями |
| `pg_repack` | Переупаковка таблиц БЕЗ блокировки (альтернатива VACUUM FULL) |
| `uuid-ossp` | Генерация UUID (v1, v4) |
| `pgvector` | Векторное хранилище для AI/ML embeddings |
| `hstore` | Key-value хранилище внутри PG |
| `postgres_fdw` | Foreign Data Wrapper — запросы к другим PG серверам |

```sql
-- Анализ самых медленных запросов
SELECT query, calls, mean_time, total_time
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 10;

-- pg_trgm: поиск подстроки
CREATE INDEX idx_users_name_trgm ON users USING GIN (name gin_trgm_ops);
SELECT * FROM users WHERE name ILIKE '%joh%';  -- Использует индекс!

-- postgres_fdw: таблица на другом сервере
CREATE SERVER analytics_server FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (host 'analytics.example.com', dbname 'analytics');

CREATE FOREIGN TABLE remote_metrics (
    id BIGINT, metric_name TEXT, value DOUBLE PRECISION
) SERVER analytics_server;
```

---

## VACUUM — тонкости настройки

### Параметры AUTOVACUUM

```sql
-- Глобальные (postgresql.conf):
autovacuum = on
autovacuum_max_workers = 3             -- параллельных vacuum рабочих
autovacuum_naptime = 1min              -- проверять каждую минуту
autovacuum_vacuum_threshold = 50       -- минимум dead tuples
autovacuum_vacuum_scale_factor = 0.2   -- +20% от размера таблицы
autovacuum_analyze_threshold = 50
autovacuum_analyze_scale_factor = 0.1  -- +10% для analyze

-- Для больших таблиц (100M+ строк) глобальные настройки плохи:
-- 20% от 100M = 20M dead tuples до VACUUM → таблица раздута!

-- Настройка на уровне таблицы:
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.01,   -- 1% для больших таблиц
    autovacuum_vacuum_threshold = 1000,
    autovacuum_analyze_scale_factor = 0.005
);

-- Мониторинг dead tuples:
SELECT relname, n_dead_tup, n_live_tup,
       round(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;
```

### Anti-wraparound VACUUM

```
Transaction ID — 32-битный (~4 млрд). При wrap-around старые xid становятся "новыми".
Защита: VACUUM FREEZE замораживает старые строки (xmin = FrozenTransactionId).

autovacuum_freeze_max_age = 200_000_000
  — При xid возрасте >200M → агрессивный VACUUM
  
Мониторинг:
SELECT datname, age(datfrozenxid) FROM pg_database;
-- Если age > 1_000_000_000 — скоро wrap-around!
```

---

## Чек-лист: PostgreSQL Advanced

- [ ] Streaming vs Logical Replication: когда что?
- [ ] Синхронная vs Асинхронная репликация: trade-off, synchronous_commit уровни
- [ ] Шардирование: стратегии (hash, range, directory, geo), Citus
- [ ] PgBouncer: режимы (session/transaction/statement), когда что использовать?
- [ ] GIN индексы для JSONB: jsonb_path_ops vs стандартный GIN
- [ ] Расширения: pg_stat_statements, pg_trgm, postgis, pgvector
- [ ] AUTOVACUUM tuning: scale_factor для больших таблиц
- [ ] Transaction ID wrap-around: что это, как предотвратить?
- [ ] Важнейшие метрики мониторинга: dead tuples %, connection count, replication lag, cache hit ratio

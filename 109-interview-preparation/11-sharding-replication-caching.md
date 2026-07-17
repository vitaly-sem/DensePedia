# Шардирование, репликация и кеширование — системный подход

---

## Шардирование (Sharding) — стратегии и компромиссы

### Почему шардирование?

```
Проблемы монолитной БД:
  1. Объём данных > размер диска одного сервера
  2. Слишком много запросов → один сервер не справляется
  3. Hotspot: одна таблица/строка получает львиную долю запросов
  4. Geo-locality: данные должны быть близко к пользователю (GDPR, latency)

Шардирование = горизонтальное масштабирование:
  Разбиваем данные по нескольким независимым серверам (шардам).
```

### Hash Sharding

```
Алгоритм:
  shard_id = hash(key) % shard_count

┌────────────────────────────────────────────────────┐
│                    Hash Ring                       │
│                                                    │
│   user_id=1  → hash(1)%4  = 1 → Shard 1           │
│   user_id=2  → hash(2)%4  = 2 → Shard 2           │
│   user_id=42 → hash(42)%4 = 2 → Shard 2           │
│   user_id=100→ hash(100)%4= 0 → Shard 0           │
└────────────────────────────────────────────────────┘

+ Равномерное распределение (при хорошем hash)
+ Простой routing (не нужен lookup)

- Range queries: SELECT * WHERE user_id BETWEEN 1 AND 1000
  → запрос ко ВСЕМ шардам! ("scatter-gather")
- Resharding: при добавлении шарда нужно перехешировать ВСЕ данные
  (consistent hashing решает частично)
```

### Consistent Hashing

```
Проблема: при добавлении шарда обычный hash % N меняется для ВСЕХ ключей.

Решение: Consistent Hashing

  Точки на кольце [0, 2^160):
  
  hash(Shard1) → позиция на кольце
  hash(Shard2) → позиция на кольце
  hash(Shard3) → позиция на кольце
  hash(key)    → идём по часовой → ближайший шард

  При добавлении Shard4:
    - Переезжают только ключи между Shard3 и Shard4 (часть ключей Shard1)
    - Остальные ключи остаются на своих шардах

  Виртуальные ноды (vnodes):
    Каждый физический шард = 100-200 виртуальных точек на кольце
    → более равномерное распределение
```

### Range Sharding

```
shard для ключа зависит от его значения (диапазона):

  Shard 0: user_id 0    .. 1_000_000
  Shard 1: user_id 1M+1 .. 2_000_000
  Shard 2: user_id 2M+1 .. 3_000_000

+ Range queries: SELECT * WHERE user_id BETWEEN 100 AND 200
  → запрос к ОДНОМУ шарду!
+ Простое добавление шарда: новый диапазон

- Hotspot: диапазон свежих пользователей перегружен
  (например, Shard 3 с новыми пользователями)
- Неравномерное распределение (хотя можно динамически)
```

### Geo-Sharding

```
Данные пользователей хранятся в регионе пользователя:

  EU users → EU cluster (Frankfurt)
  US users → US cluster (Virginia)
  APAC users → APAC cluster (Singapore)

+ Data locality (GDPR: данные EU в EU)
+ Минимальная latency для пользователей
+ Изоляция отказов

- Сложность: cross-region запросы (медленно)
- Миграция пользователя между регионами
- Глобальная аналитика требует сбора со всех регионов
```

### Resharding — боль перехода

```
Сценарий: 4 шарда → 8 шардов (удвоение).

Стратегии:

1. Double-write (без downtime):
   - Новые записи идут в оба кластера
   - Старые данные мигрируют в фоне (постепенно)
   - Чтение: сначала новый шард, fallback на старый
   - После полной миграции старый кластер отключается

2. Logical replication:
   - Настраиваем logical replication со старых шардов на новые
   - Мигрируем снапшот
   - Переключаем трафик

3. Исторические партиции не трогаем:
   - Старые данные (архив) остаются на старых шардах
   - Новые данные распределяются по новой схеме
```

---

## Репликация — топологии и паттерны

### Leader-Follower (Primary-Standby)

```
┌──────────┐
│  Leader  │── Запись (Write)
│  (RW)    │
└────┬─────┘
     │ Replication
     ├──────────────┬──────────────┐
     ▼              ▼              ▼
┌──────────┐  ┌──────────┐  ┌──────────┐
│Follower 1│  │Follower 2│  │Follower 3│── Чтение (Read)
│  (RO)    │  │  (RO)    │  │  (RO)    │
└──────────┘  └──────────┘  └──────────┘

+ Простота
+ Read scalability
+ Failover (при отказе leader → follower повышается)

- Единая точка записи (leader)
- Replication lag (follower может отставать)
- Split-brain риск при неверном failover
```

### Leader-Leader (Multi-Master)

```
┌──────────┐     ┌──────────┐
│ Leader A │◄───►│ Leader B │
│  (RW)    │     │  (RW)    │
└──────────┘     └──────────┘

+ Пишем в оба дата-центра
+ Нет единой точки отказа для записи

- Конфликты: одна и та же запись изменена в A и B одновременно
  → нужна стратегия разрешения конфликтов (LWW, CRDT)
- Сложность настройки и эксплуатации
```

### Chain Replication

```
Write → ┌────────┐    ┌────────┐    ┌────────┐
        │ Node 1 │───→│ Node 2 │───→│ Node 3 │
        │ (Head) │    │        │    │ (Tail) │
        └────────┘    └────────┘    └────┬───┘
                                         │
                                    Read ◄─┘

+ Сильная консистентность (читаем с tail — последняя версия)
+ Простая модель

- Latency записи = сумма всех RTT
- Head — единая точка отказа для записи
```

### Quorum-based Replication

```
N = общее число реплик
W = число реплик для подтверждения записи (write quorum)
R = число реплик для чтения (read quorum)

Условие сильной консистентности: W + R > N

Пример (N=3):
  W=2, R=2 → W+R=4 > 3 ✓ (сильная консистентность)
  W=1, R=1 → W+R=2 ≤ 3 ✗ (eventual consistency)
  W=2, R=1 → W+R=3 = N ✗ (не гарантирует свежесть)
  W=3, R=1 → W+R=4 > 3 ✓ (сильная консистентность, но нет доступности при 1 отказе)
```

```csharp
// Упрощённый quorum write
async Task<bool> WriteWithQuorum(string key, string value, int w, int n)
{
    var tasks = replicas.Take(n).Select(r => r.WriteAsync(key, value));
    var results = await Task.WhenAll(tasks);
    return results.Count(r => r.Success) >= w;
}

// Упрощённый quorum read (с разрешением конфликтов по версии)
async Task<string> ReadWithQuorum(string key, int r, int n)
{
    var tasks = replicas.Take(n).Select(r => r.ReadAsync(key));
    var results = await Task.WhenAll(tasks);
    var successful = results.Where(r => r.Success).ToList();
    if (successful.Count < r) throw new QuorumException();
    return successful.MaxBy(r => r.Version).Value;  // Самая свежая версия
}
```

---

## Кеширование — стратегии и паттерны

### Cache-Aside (Look-Aside) — самый распространённый

```
┌─────────┐  1. Запрос   ┌─────────┐  2. Miss  ┌─────────┐
│  App    │─────────────→│  Cache  │──────────→│   DB    │
│         │←─────────────│ (Redis) │←──────────│         │
└─────────┘  3. Вернуть  └─────────┘  4. Вернуть└─────────┘
     │                                        ↑
     └────────────────────────────────────────┘
        3a. Записать в кеш (для следующего запроса)

+ Простой
+ Lazy loading (кешируется только то, что нужно)
+ Отказ кеша не ломает систему (только latency растёт)

- Cache stampede при холодном старте
- Данные в кеше могут устареть (stale)
```

```csharp
// Cache-Aside с защитой от stampede (lock per key)
public async Task<User> GetUserAsync(string userId)
{
    var cached = await cache.GetAsync<User>($"user:{userId}");
    if (cached != null) return cached;

    // Защита от cache stampede: только один поток идёт в БД
    using var lockHandle = await cache.LockAsync($"lock:user:{userId}", TimeSpan.FromSeconds(5));

    // Double-check после получения блокировки
    cached = await cache.GetAsync<User>($"user:{userId}");
    if (cached != null) return cached;

    var user = await db.GetUserAsync(userId);
    await cache.SetAsync($"user:{userId}", user, TimeSpan.FromMinutes(10));
    return user;
}
```

### Read-Through

```
App → Cache → DB (прозрачно для приложения)

Cache сам загружает данные из БД при miss.
Приложение всегда общается только с кешем.

+ Прозрачно для приложения
+ Всегда консистентная логика загрузки

- Нужен cache provider с поддержкой read-through
- Медленный первый запрос (miss)
```

### Write-Through

```
App → Cache → DB (синхронно)

Запись сначала в кеш, затем в БД (синхронно).

+ Данные в кеше всегда актуальны
+ Read сразу после write — всегда попадание

- Latency записи = cache + DB (медленнее прямой записи в DB)
- Неиспользуемые данные тоже в кеше
```

### Write-Behind (Write-Back)

```
App → Cache → DB (асинхронно, отложенно)

Запись только в кеш. Кеш асинхронно сбрасывает в БД.

+ Минимальная latency записи
+ Можно батчить записи в БД

- Риск потери данных при отказе кеша
- Сложная реализация (нужен надёжный кеш)
```

### Refresh-Ahead

```
Кеш proactively обновляет "горячие" ключи до истечения TTL.

При запросе ключа:
  - Если TTL < порог обновления → асинхронно обновить из БД
  - Вернуть текущее (возможно, слегка устаревшее) значение

+ Нет cache miss для горячих ключей
+ Latency всегда как cache hit

- Дополнительная нагрузка на БД (фоновые обновления)
```

---

## Redis — продвинутое использование

### Структуры данных в Redis

```
String   → GET/SET/INCR/DECR/APPEND
Hash     → HSET/HGET/HGETALL (объекты: пользователь, сессия)
List     → LPUSH/RPUSH/LPOP/RPOP (очереди)
Set      → SADD/SMEMBERS/SINTER/SUNION (теги, друзья)
Sorted Set → ZADD/ZRANGE/ZRANK (рейтинги, лидерборды)
Stream   → XADD/XREAD/XGROUP (лёгкий аналог Kafka)
Bitmap   → SETBIT/BITCOUNT (онлайн-статус пользователей)
HyperLogLog → PFADD/PFCOUNT (приблизительный подсчёт уникальных)
Geo      → GEOADD/GEORADIUS (геопоиск)
```

### Redis Cluster

```
┌─────────────────────────────────────────────────┐
│                REDIS CLUSTER                    │
│                                                 │
│  Hash Slots: 16384 слота                        │
│                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐     │
│  │ Master 1 │  │ Master 2 │  │ Master 3 │     │
│  │ Slots    │  │ Slots    │  │ Slots    │     │
│  │ 0-5460   │  │ 5461-    │  │ 10923-   │     │
│  │          │  │ 10922    │  │ 16383    │     │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘     │
│       │              │              │           │
│  ┌────▼─────┐  ┌────▼─────┐  ┌────▼─────┐     │
│  │ Slave 1  │  │ Slave 2  │  │ Slave 3  │     │
│  └──────────┘  └──────────┘  └──────────┘     │
│                                                 │
│  slot = CRC16(key) % 16384                     │
│  Multi-key операции только в одном слоте       │
│  (hash tags: {user:123}:name, {user:123}:age)  │
└─────────────────────────────────────────────────┘
```

### Redis как распределённая блокировка (Redlock)

```csharp
// Алгоритм Redlock (Martin Kleppmann)
// N независимых Redis инстансов (не кластер!)

async Task<IDisposable> AcquireLock(string resource, TimeSpan ttl)
{
    string token = Guid.NewGuid().ToString();
    int acquired = 0;

    foreach (var redis in redisInstances)
    {
        if (await redis.StringSetAsync(resource, token, ttl, StackExchange.Redis.When.NotExists))
            acquired++;
    }

    if (acquired >= redisInstances.Length / 2 + 1)  // Большинство
        return new RedLockHandle(resource, token);

    // Не набрали кворум — освобождаем
    await ReleaseLock(resource, token);
    throw new LockAcquisitionException();
}

// Важно: lock должен сниматься только владельцем (проверка token)
async Task ReleaseLock(string resource, string token)
{
    // Lua script для атомарной проверки токена + удаления
    const string script = @"
        if redis.call('get', KEYS[1]) == ARGV[1] then
            return redis.call('del', KEYS[1])
        else
            return 0
        end";
    foreach (var r in redisInstances)
        await r.ScriptEvaluateAsync(script, new[] { resource }, new[] { token });
}
```

---

## Cache Invalidation Strategies

```
Проблема: как понять, что данные в кеше устарели?

1. TTL (Time-To-Live):
   - Просто: SET key value EX 300
   - Данные могут быть stale в течение TTL

2. Write Invalidation:
   - При обновлении в БД → удалить/обновить кеш
   - Риск: если инвалидация не дошла → stale навсегда

3. Event-based Invalidation:
   - БД публикует событие изменения (CDC, Outbox)
   - Cache subscriber обновляет/инвалидирует

4. Version-based:
   - Каждая запись имеет version в БД
   - Кеш хранит (data, version)
   - При чтении: если version в кеше < version в БД → miss
```

```sql
-- Event-based invalidation через PostgreSQL LISTEN/NOTIFY
-- Триггер на изменение:
CREATE OR REPLACE FUNCTION notify_user_change()
RETURNS TRIGGER AS $$
BEGIN
    PERFORM pg_notify('user_changes', json_build_object(
        'table', TG_TABLE_NAME,
        'op', TG_OP,
        'id', NEW.id
    )::text);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER user_changed
AFTER INSERT OR UPDATE OR DELETE ON users
FOR EACH ROW EXECUTE FUNCTION notify_user_change();
```

---

## NoSQL — сравнение и выбор

### Типы NoSQL баз данных

```
1. Key-Value:
   Redis, DynamoDB, Riak
   + Простейшая модель, максимальная скорость
   - Только по ключу, нет сложных запросов
   Кейсы: кеши, сессии, real-time счётчики

2. Document:
   MongoDB, CouchDB, Firestore
   + Гибкая схема (JSON), вложенные документы
   - Нет JOIN'ов, ограниченные транзакции
   Кейсы: каталоги товаров, CMS, профили пользователей

3. Column-Family (Wide-Column):
   Cassandra, HBase, ScyllaDB
   + Горизонтальное масштабирование, write-heavy
   - Сложная модель запросов (нужно знать ключи)
   Кейсы: временные ряды, логи, IoT, messaging

4. Graph:
   Neo4j, Amazon Neptune, ArangoDB
   + Сложные связи (друзья друзей, рекомендации)
   - Не для общих OLTP нагрузок
   Кейсы: социальные графы, рекомендации, fraud detection

5. Search:
   Elasticsearch, OpenSearch, Solr
   + Полнотекстовый поиск, агрегации
   - Не primary store (не для хранения данных)
   Кейсы: поиск, логирование (ELK), аналитика
```

### Cassandra vs MongoDB vs PostgreSQL

| Характеристика | Cassandra | MongoDB | PostgreSQL |
|---------------|-----------|---------|------------|
| Модель | Wide-column | Document | Relational |
| Схема | Table + колонки (CQL) | Schemaless (BSON) | Строгая схема |
| JOIN'ы | Нет | $lookup (ограничен) | Полная поддержка |
| Транзакции | Нет ACID | Multi-doc (4.0+) | Полный ACID |
| Масштабирование | Горизонтальное (линейное) | Горизонтальное (sharding) | Вертикальное + Citus |
| CAP | AP | CP (по умолчанию) | CP |
| Write perf. | Отличная (append-only) | Хорошая | Хорошая (но MVCC overhead) |
| Read perf. | По ключу — отлично | По индексу — хорошо | Сложные запросы — отлично |
| Консистентность | Eventual (tunable) | Strong (primary) | Strong (ACID) |

### Когда что выбрать?

```
PostgreSQL:
  → Нужны сложные JOIN'ы, транзакции, отчёты
  → Финансовые данные, ERP, бухгалтерия
  → Строгая схема данных важна

Cassandra:
  → Миллионы записей в секунду (write-heavy)
  → Данные растут линейно (time-series)
  → Нет сложных запросов (только по ключу)
  → Нужна геораспределённость (multi-DC)

MongoDB:
  → Быстрый прототип (гибкая схема)
  → Иерархические документы (вложенные объекты)
  → Нет сложных связей между коллекциями
  → Mobile/Web apps с гибкой моделью

Redis (как primary, не только кеш):
  → Счётчики, лидерборды (Sorted Sets)
  → Rate limiting (String + TTL)
  → Pub/Sub, очереди (List/Stream)
  → Сессии (Hash + TTL)
```

---

## Чек-лист: шардирование, репликация, кеширование

### Шардирование
- [ ] Hash vs Range vs Geo sharding: когда что?
- [ ] Consistent Hashing: зачем, как работает?
- [ ] Resharding: стратегии миграции без downtime
- [ ] Cross-shard transactions: как обрабатывать? (2PC, Saga)

### Репликация
- [ ] Leader-Follower vs Leader-Leader vs Quorum
- [ ] R + W > N: доказательство сильной консистентности
- [ ] Replication lag: как бороться? (read-your-writes, monotonic reads)
- [ ] Split-brain: как обнаружить и предотвратить?

### Кеширование
- [ ] Cache-Aside, Read-Through, Write-Through, Write-Behind
- [ ] Cache stampede: что это и как защититься?
- [ ] Redis Cluster: hash slots, hash tags
- [ ] Распределённая блокировка: Redlock алгоритм
- [ ] Cache invalidation strategies: TTL, write-invalidate, event-based, version-based

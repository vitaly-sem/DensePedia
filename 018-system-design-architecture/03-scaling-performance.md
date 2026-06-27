# Масштабирование и performance

---

## Horizontal vs Vertical Scaling

```
Vertical: Увеличиваем мощность одной машины
  (2 CPU → 64 CPU, 8GB → 512GB RAM)
  (+) Просто, не меняет архитектуру
  (-) Есть предел, дорого (bigger machine costs super-linearly)

Horizontal: Добавляем больше машин
  (1 server → 100 servers)
  (-) Нужна distributed архитектура
  (+) Линейное масштабирование, дешевле, fault-tolerant
```

**Факт:** AWS EC2: m6i.large ($0.096/hr) → m6i.24xlarge ($3.072/hr). Разница в цене ×32, но CPU ×48 — сублинейно, а network ×12.

---

## Throughput vs Latency

```
Latency    = время одного запроса (ms)
Throughput = запросов в секунду (req/s)

Закон Литтла: L = λ × W
  L = concurrent requests in system
  λ = throughput
  W = average latency
```

**Пример:** Если latency = 100ms, throughput = 100 req/s, то concurrent requests = 100 × 0.1 = 10.

---

## CAP Theorem (for distributed systems)

```
Consistency: Все видят одни и те же данные
Availability: Каждый запрос получает ответ (не обязательно корректный)
Partition Tolerance: Система работает при потере связи между узлами
```

| Тип | Пример | Компромисс |
|-----|--------|------------|
| **CP** | HBase, Zookeeper, etcd | Отказ при partition |
| **AP** | DynamoDB, Cassandra, Couchbase | Stale reads при partition |
| **CA** | Single-node RDBMS | Нет partition tolerance |

**Факт:** В распределённой системе P (Partition Tolerance) обязателен. Реальный выбор — CP или AP.

---

## Consistency Models

### Strong Consistency

- После записи — все читатели сразу видят
- Минус: медленнее, меньше availability

### Eventual Consistency

- Запись → читатели могут видеть старое → но со временем увидят новое
- Плюс: быстрее, выше availability

### Read-your-writes

- Свои записи вижу сразу, чужие — eventual

### Пример: Amazon DynamoDB

```csharp
// Eventually consistent read (default) — быстро, дешево
var item = await table.GetItemAsync(key);

// Strongly consistent read — дороже, дольше
var item = await table.GetItemAsync(key, new GetItemOperationConfig
{
    ConsistentRead = true
});
```

---

## Caching Strategies

### Multi-level Cache

```
CDN (edge, 10ms)
  ↓ miss
App Cache (in-memory, 1ms)
  ↓ miss
Distributed Cache (Redis, 5ms)
  ↓ miss
Database (50ms)
```

### Cache-aside Pattern

```csharp
public async Task<Order> GetOrderAsync(string orderId)
{
    // 1. Try cache
    var order = await cache.GetAsync<Order>(orderId);
    if (order != null) return order;

    // 2. Cache miss → read DB
    order = await db.Orders.FindAsync(orderId);

    // 3. Write to cache
    await cache.SetAsync(orderId, order, TimeSpan.FromMinutes(10));

    return order;
}
```

### Cache Invalidation — hardest problem?

| Стратегия | Описание | Риск |
|-----------|----------|------|
| **TTL** | Автоматическое удаление | Stale data до TTL |
| **Write-through** | Обновляем cache при записи | Сложнее, но консистентно |
| **Event-based** | Cache invalidation по событию | Сложность инфраструктуры |
| **No cache** | Не кэшируем | Скорость |

---

## Estimation (Capacity Planning)

### Пример: Twitter

**Дано:**
- 500M tweets/day
- Read:Write ratio = 100:1

**Расчёты:**
```
Tweets/sec = 500M / 86400 ≈ 5,800 writes/sec
Reads/sec = 5,800 × 100 = 580,000 reads/sec

Storage:
  Tweet size ≈ 1KB
  500M × 1KB = 500GB/day
  Yearly: 500GB × 365 ≈ 182TB/year
  With replication (×3): ~546TB/year
```

### Правила для оценки

| Компонент | Приближение |
|-----------|-------------|
| **Daily traffic** | Total / 86400 |
| **Peak traffic** | Average × 2-5 |
| **Storage** | Per-item size × items × retention |
| **Bandwidth** | Response size × throughput |
| **Memory for cache** | Hot data (20% that gets 80% traffic) |

---

## Back-of-the-envelope Numbers

| Операция | Время |
|----------|-------|
| L1 cache reference | 0.5 ns |
| Branch mispredict | 5 ns |
| L2 cache reference | 7 ns |
| Mutex lock/unlock | 25 ns |
| Main memory reference | 100 ns |
| SSD random read | 150,000 ns (150 μs) |
| Round trip in datacenter | 500,000 ns (0.5 ms) |
| SSD sequential read (1MB) | 1,000,000 ns (1 ms) |
| Disk seek | 10,000,000 ns (10 ms) |
| Network packet CA→Netherlands→CA | 150,000,000 ns (150 ms) |

---

## Чек-лист

- [ ] Выбрана стратегия scaling (horizontal для stateless)
- [ ] Кэширование на нескольких уровнях
- [ ] TTL настроен (не слишком долго, не слишком часто)
- [ ] CAP-выбор сделан осознанно
- [ ] Capacity planning по трафику и storage
- [ ] Read replicas настроены для read-heavy
- [ ] Load testing проведён (k6, Locust, wrk)
- [ ] Auto-scaling настроен (HPA в K8s)

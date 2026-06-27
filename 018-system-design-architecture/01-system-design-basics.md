# Основы системного дизайна

---

## Load Balancing

```
User ──→ DNS ──→ LB ──→ App Server 1
                  │      App Server 2
                  │      App Server N
```

### Алгоритмы LB

| Алгоритм | Описание | Когда |
|----------|----------|-------|
| **Round Robin** | По очереди | Простые сценарии |
| **Least Connections** | На сервер с наименьшим числом соединений | Длинные сессии |
| **IP Hash** | По IP клиента → sticky session | Нужна привязка к серверу |
| **Weighted** | С учётом мощности серверов | Гетерогенные серверы |

### LB в разных слоях

```
L4 (Transport) — TCP, UDP — быстро, без анализа контента (HAProxy, NLB)
L7 (Application) — HTTP/HTTPS — может смотреть URL, cookies (ALB, Nginx, Traefik)
```

**Факт:** Layer 7 LB может делать **content-based routing**: `/api/v1/*` → серверы v1, `/api/v2/*` → серверы v2.

---

## Caching

### Стратегии кэширования

```
Cache-Aside:
    App → Check Cache → Miss → Read DB → Write Cache → Return

Read-Through:
    App → Cache → Miss → Cache reads DB → Return

Write-Through:
    App → Write to Cache → Cache writes DB → OK

Write-Behind:
    App → Write to Cache → OK (async write to DB)
```

### Cache Invalidation

| Стратегия | Описание |
|-----------|----------|
| **TTL** | Время жизни — самое простое |
| **Write-through** | Синхронное обновление |
| **Event-based** | Инвалидация по событию (pub/sub) |
| **Manual** | Администратор очищает кэш |

### CDN (Content Delivery Network)

```
User in Europe ──→ CDN Edge (Frankfurt) ──→ Origin (US-East)
                                    ↑
Local cache: images, CSS, JS, video
```

**Факт:** CDN может обслуживать **до 95% статического контента**, не доходя до origin.

---

## Database Scaling

### Read Replicas

```
┌─────────┐     ┌──────────┐
│  Write  │────→│  Read    │
│  Master │     │  Replica │
└─────────┘     │  #1      │
                ├──────────┤
                │  Read    │
                │  Replica │
                │  #2      │
                └──────────┘
```

**Когда:** Read > Write. **Не консистентно** — replication lag.

### Database Sharding

```
User ID 1-1000000 ──→ Shard 0
User ID 1000001-2000000 ──→ Shard 1
User ID 2000001-3000000 ──→ Shard 2
```

**Shard key — самый важный выбор:** должен равномерно распределять данные и поддерживать основные паттерны доступа.

---

## Asynchronous Processing

### Message Queue

```
Producer ──→ Queue ──→ Consumer
                 │
            Dead Letter Queue (DLQ) — для упавших сообщений
```

| Queue | Протокол | Особенность |
|-------|----------|-------------|
| **RabbitMQ** | AMQP | Гарантированная доставка, routing |
| **Apache Kafka** | Log-based | Высокий throughput, replay, retention |
| **Azure Service Bus** | AMQP | Managed, sessions, dead-letter |
| **AWS SQS** | Pull-based | Простой, managed |

### Event-Driven Architecture

```
Order Service (OrderCreated Event)
    │
    ├──→ Notification Service (Email/SMS)
    ├──→ Inventory Service (Reserve)
    ├──→ Payment Service (Charge)
    └──→ Analytics Service (Track)
```

---

## Rate Limiting

### Алгоритмы

| Алгоритм | Плюсы | Минусы |
|----------|-------|--------|
| **Token Bucket** | Простой, burst-запросы | Может превысить лимит на burst |
| **Leaky Bucket** | Равномерный поток | Не даёт burst |
| **Fixed Window** | Простой | Edge-case: трафик на границе |
| **Sliding Window Log** | Точный | Много памяти |
| **Sliding Window Counter** | Хороший balance | Реализация сложнее |

### Пример Token Bucket

```python
class TokenBucket:
    def __init__(self, rate, capacity):
        self.rate = rate      # tokens per second
        self.capacity = capacity
        self.tokens = capacity
        self.last_refill = time.time()

    def allow_request(self):
        now = time.time()
        self.tokens += (now - self.last_refill) * self.rate
        self.tokens = min(self.tokens, self.capacity)
        self.last_refill = now

        if self.tokens >= 1:
            self.tokens -= 1
            return True
        return False
```

---

## Чек-лист

- [ ] Load balancer настроен (L4/L7 в зависимости от сценария)
- [ ] Кэширование реализовано на нескольких уровнях (CDN + app + DB)
- [ ] TTL инвалидация настроена
- [ ] Message queue для асинхронных операций
- [ ] Rate limiting защищает API
- [ ] Read replicas для read-heavy сценариев
- [ ] Dead letter queue для упавших сообщений

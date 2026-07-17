# Load Balancing и High Availability

---

## Load Balancing — уровни и алгоритмы

### L4 vs L7 Load Balancing

```
┌─────────────────────────────────────────────────────────┐
│                    OSI MODEL                            │
│                                                         │
│  L7  Application  ── HTTP, gRPC, WebSocket             │
│      (содержимое)   L7 LB: Nginx, Envoy, HAProxy,      │
│                     AWS ALB, Traefik                    │
│                                                         │
│  L4  Transport    ── TCP, UDP                          │
│      (соединение)   L4 LB: HAProxy (tcp mode),         │
│                     AWS NLB, IPVS, LVS                  │
└─────────────────────────────────────────────────────────┘

L4:
  + Быстрее (только заголовки TCP)
  + Работает с любым протоколом
  - Нет routing по URL/headers/cookies
  - Нет TLS termination

L7:
  + Routing по URL, Host header, cookies
  + TLS termination
  + Rate limiting, retries, circuit breaking
  - Больше latency (парсинг HTTP)
  - Меньше throughput
```

### Алгоритмы балансировки

```
1. Round Robin:
   Запросы по очереди на каждый сервер.
   + Простой, равномерный при одинаковой нагрузке
   - Не учитывает загрузку сервера

2. Least Connections:
   Запрос к серверу с минимальным числом активных соединений.
   + Учитывает разную нагрузку запросов
   - Нужно отслеживать состояние соединений

3. Weighted Round Robin:
   Серверы с разными весами (мощнее → больше запросов).
   + Гетерогенное железо
   - Статические веса — не реагируют на реальную загрузку

4. IP Hash / Sticky Sessions:
   hash(client_ip) % server_count → всегда один сервер.
   + Сохраняет сессию (удобно для stateful)
   - Неравномерность при малом числе IP
   - При добавлении/удалении сервера — перехеширование

5. Consistent Hashing:
   hash(client_ip) на кольце → ближайший сервер по часовой.
   + Минимум перемещений при изменении числа серверов
   - Требует виртуальных нод для равномерности

6. Least Response Time:
   Запрос к серверу с минимальным средним временем ответа.
   + Адаптивность
   - Нужно измерять latency
```

### Nginx — типичная конфигурация

```nginx
upstream backend {
    # Алгоритм по умолчанию: round-robin
    # least_conn;  # Least Connections
    # ip_hash;     # Sticky Sessions

    server backend1.example.com:8080 weight=3 max_fails=3 fail_timeout=30s;
    server backend2.example.com:8080 weight=2 max_fails=3 fail_timeout=30s;
    server backend3.example.com:8080 weight=1 backup;  # Включается при отказе остальных
    
    keepalive 32;  # Держать 32 keep-alive соединения
}

server {
    listen 443 ssl http2;
    server_name api.example.com;

    # Rate limiting: 10 запросов/сек с burst до 20
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    limit_req zone=api burst=20 nodelay;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        # Таймауты
        proxy_connect_timeout 5s;
        proxy_read_timeout 30s;
        proxy_send_timeout 30s;
        
        # Retry на другом бэкенде при ошибке
        proxy_next_upstream error timeout http_502 http_503;
        proxy_next_upstream_tries 2;
    }
}
```

---

## High Availability — паттерны

### Active-Passive

```
┌──────────┐     ┌──────────┐
│ Active   │────→│ Passive  │ (репликация данных)
│ (primary)│     │ (standby)│
└────┬─────┘     └──────────┘
     │ FAIL
     ▼
┌──────────┐
│ Active   │  ← Passive promoted to Active
│ (бывший  │
│ standby) │
└──────────┘

+ Простой (нет конфликтов)
+ Гарантированная консистентность
- Ресурс passive простаивает
- Время failover: секунды-минуты
- Split-brain если оба считают себя active (нужен fencing)
```

### Active-Active

```
┌──────────┐     ┌──────────┐
│ Active A │◄───►│ Active B │
│ (RW)     │     │ (RW)     │
└──────────┘     └──────────┘
     │                │
     ▼                ▼
  Users (geo-routed: EU → A, US → B)

+ Оба работают (нет "простоя")
+ Мгновенный failover (трафик переключается)
- Конфликты данных (нужны CRDT или LWW)
- Сложнее (конфликты, синхронизация)
```

### N+1 Redundancy

```
N активных серверов + 1 горячий резерв.

При отказе 1 из N:
  - Резервный принимает нагрузку отказавшего
  - Оставшиеся N-1 активных + 1 резервный = N

Принцип: всегда иметь +1 запасной.
Cloud: auto-scaling group с min=N, max=N+1.
```

### Leadership Election (выборы лидера)

```
Когда несколько серверов, но только один должен быть leader (active):

Алгоритмы:
  - Paxos / Raft (сильная консистентность)
  - ZooKeeper ephemeral nodes
  - Kubernetes Leader Election (Lease API)
  - etcd election API

Пример: только один инстанс cron-сервиса должен запускать задачи.
```

```csharp
// Leader Election через базу данных (простой вариант)
public class DbLeaderElection
{
    public async Task<bool> TryBecomeLeader(string instanceId, TimeSpan ttl)
    {
        var now = DateTime.UtcNow;
        // Пытаемся захватить лидерство
        var result = await db.ExecuteAsync(@"
            INSERT INTO leader_election (lock_name, instance_id, acquired_at, expires_at)
            VALUES ('cron-scheduler', @instanceId, @now, @expires)
            ON CONFLICT (lock_name) DO UPDATE
            SET instance_id = @instanceId,
                acquired_at = @now,
                expires_at = @expires
            WHERE leader_election.expires_at < @now",
            new { instanceId, now, expires = now + ttl });
        return result > 0;
    }

    public async Task RenewLeadership(string instanceId, TimeSpan ttl)
    {
        await db.ExecuteAsync(@"
            UPDATE leader_election
            SET expires_at = @expires
            WHERE lock_name = 'cron-scheduler' AND instance_id = @instanceId",
            new { instanceId, expires = DateTime.UtcNow + ttl });
    }
}
```

---

## Failover Strategies

### DNS Failover

```
Проблема: IP сервера изменился, DNS кеширует старый IP.

Решение:
  - Маленький TTL (60 секунд)
  - DNS Health Check (Route 53, Cloudflare)
  - При отказе → DNS обновляется → через TTL трафик идёт на новый IP

Недостаток: медленно (TTL задержка), не все клиенты уважают TTL.
```

### Virtual IP (VIP) Failover

```
VIP: 10.0.0.100
┌──────────┐     ┌──────────┐
│ Server 1 │     │ Server 2 │
│ 10.0.0.1 │     │ 10.0.0.2 │
│ (Active) │     │ (Standby)│
└────┬─────┘     └──────────┘
     │ VIP здесь

При отказе Server 1:
  - Server 2 забирает VIP 10.0.0.100
  - ARP-запрос: "кто отвечает за 10.0.0.100? Новый MAC — Server 2"

keepalived / VRRP (Virtual Router Redundancy Protocol):
  - Быстрый failover: ~1-3 секунды
  - Только в пределах одного L2-сегмента (одна подсеть)
```

### Database Failover

```
PostgreSQL:

1. Streaming Replication + Automatic Failover:
   - Patroni (рекомендуется) + etcd/ZooKeeper
   - repmgr
   - Cloud-managed: AWS RDS Multi-AZ, Cloud SQL HA

2. Patroni:
   ┌──────────┐   ┌──────────┐   ┌──────────┐
   │ Patroni  │   │ Patroni  │   │ Patroni  │
   │ + PG     │   │ + PG     │   │ + PG     │
   │ (Leader) │   │(Replica) │   │(Replica) │
   └────┬─────┘   └────┬─────┘   └────┬─────┘
        │              │              │
        └──────────────┼──────────────┘
                       │
               ┌───────▼──────┐
               │ etcd Cluster │ (лидер + конфигурация)
               └──────────────┘

   - Patroni пишет в etcd: кто лидер, конфигурация
   - При отказе лидера → Patroni на реплике инициирует failover
   - etcd гарантирует: только один лидер (fencing через leader key)
```

---

## Disaster Recovery

### RPO и RTO

```
RPO (Recovery Point Objective):
  Сколько данных можно позволить потерять?
  → Определяет частоту бэкапов / репликации

RTO (Recovery Time Objective):
  Сколько времени на восстановление?
  → Определяет архитектуру DR

Пример:
  RPO = 5 минут  → streaming replication с минимальным lag
  RPO = 24 часа  → ежедневный бэкап
  RTO = 1 минута → автоматический failover (active-active/active-passive)
  RTO = 4 часа   → ручной failover + восстановление из бэкапа
```

### Стратегии Disaster Recovery

```
1. Backup & Restore (самый дешёвый):
   RPO: часы-дни, RTO: часы-дни
   Регулярный бэкап → S3/Glacier → восстановить при аварии

2. Pilot Light:
   RPO: минуты, RTO: десятки минут
   Минимальная инфраструктура запущена (БД + core)
   При аварии: развернуть остальное вокруг Pilot Light

3. Warm Standby:
   RPO: секунды, RTO: минуты
   Полная копия запущена, но в уменьшенном масштабе
   При аварии: масштабировать до полной мощности

4. Multi-Site Active-Active:
   RPO: секунды (или 0), RTO: секунды
   Все регионы активны, трафик распределён
   Самый дорогой вариант
```

---

## Health Checking

### Типы проверок

```
1. Liveness Probe (жив ли процесс?):
   - TCP connect к порту
   - HTTP GET /healthz → 200
   - Если FAIL → перезапустить контейнер

2. Readiness Probe (готов принимать трафик?):
   - HTTP GET /ready → 200 (проверка зависимостей: БД, Redis, etc.)
   - Если FAIL → убрать из балансировки (но не перезапускать!)
   - Используется Service Discovery / Load Balancer

3. Startup Probe (для медленно стартующих приложений):
   - Более длительный initial delay
   - Защищает от premature readiness failure
```

```csharp
// ASP.NET Core Health Checks
builder.Services.AddHealthChecks()
    .AddNpgSql(connectionString, name: "postgres")
    .AddRedis(redisConnectionString, name: "redis")
    .AddUrlGroup(new Uri("https://api.partner.com/health"), "partner-api");

app.MapHealthChecks("/healthz", new HealthCheckOptions
{
    Predicate = _ => true  // Все проверки
});

app.MapHealthChecks("/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("readiness")
});
```

---

## Graceful Degradation

```
Принцип: отказавшая функциональность не должна ронять всю систему.

Примеры:

1. Рекомендации упали:
   - Показываем "горячие" товары (статический fallback)
   - Без рекомендаций магазин продолжает работать

2. Redis упал:
   - Circuit breaker → запросы идут напрямую в БД
   - Повышенная latency, но система работает

3. Поиск упал:
   - Показываем каталог с фильтрацией на клиенте
   - "Поиск временно недоступен" лучше чем 500 ошибка

Реализация:
  - Таймауты на все внешние вызовы
  - Circuit Breaker
  - Fallback-значения / кешированные данные
  - Feature flags для отключения фич
```

---

## Чек-лист: Load Balancing и HA

- [ ] L4 vs L7 Load Balancing: отличия, когда что?
- [ ] Алгоритмы: Round Robin, Least Connections, IP Hash, Consistent Hashing, Least Response Time
- [ ] Nginx/HAProxy: upstream, health checks, keepalive, rate limiting
- [ ] Active-Passive vs Active-Active: trade-offs
- [ ] Leader Election: ZooKeeper, etcd, database-based
- [ ] Failover: DNS, VIP/VRRP, Patroni (PostgreSQL)
- [ ] RPO и RTO: что значат, как определить?
- [ ] DR стратегии: Backup/Restore, Pilot Light, Warm Standby, Multi-Site
- [ ] Health Checks: Liveness vs Readiness vs Startup
- [ ] Graceful Degradation: Circuit Breaker, fallback, feature flags
- [ ] Split-brain: как возникает, как предотвратить (fencing, quorum)?
- [ ] Fencing: что это и зачем? (STONITH, etcd lease, PostgreSQL lock)

# System Design — методология прохождения собеседования

---

## Структура System Design Interview

```
Время: 45-60 минут

 0-5  мин: Уточнение требований (functional + non-functional)
 5-15 мин: Оценка ресурсов и ограничений (back-of-the-envelope)
15-25 мин: High-level design (архитектурная диаграмма)
25-40 мин: Deep dive в ключевой компонент
40-50 мин: Отказоустойчивость, мониторинг, эксплуатация
50-60 мин: Ответы на вопросы, обсуждение trade-offs
```

---

## Шаг 1: Сбор требований (Requirements Gathering)

### Активные вопросы интервьюеру

```
ФУНКЦИОНАЛЬНЫЕ ТРЕБОВАНИЯ:
  - Кто пользователи? Какие роли?
  - Какие основные use cases?
  - Какие входные/выходные данные?
  - Нужен ли поиск? Фильтрация? Сортировка?
  - Какие интеграции с внешними системами?

НЕФУНКЦИОНАЛЬНЫЕ ТРЕБОВАНИЯ:
  - Ожидаемое количество пользователей? (DAU/MAU)
  - Ожидаемая нагрузка? (RPS — requests per second)
  - Требования к latency? (p50, p95, p99)
  - Требования к availability? (99.9%? 99.99%?)
  - Требования к durability? (нельзя терять данные?)
  - Требования к consistency? (strong vs eventual)
  - География пользователей? (multi-region?)
```

### Пример: проектируем "Сервис сокращения ссылок" (URL Shortener)

```
Функциональные:
  - Пользователь вводит длинный URL → получает короткий
  - Переход по короткому URL → редирект на оригинальный
  - (Опционально): статистика переходов, кастомные алиасы, истечение ссылок

Нефункциональные:
  - 100 млн новых ссылок/месяц → ~40 запросов/сек на создание
  - 1 млрд переходов/месяц → ~400 запросов/сек на чтение (read-heavy: 10:1)
  - Latency: p99 < 50ms для редиректа
  - Availability: 99.99% (сервис должен быть доступен всегда)
  - Ссылки хранятся 5 лет → ~6 млрд записей
```

---

## Шаг 2: Оценка ресурсов (Back-of-the-Envelope Estimation)

### Базовые цифры для расчётов

| Ресурс | Приблизительное значение |
|--------|--------------------------|
| L1 Cache latency | ~1 ns |
| L2 Cache latency | ~5 ns |
| RAM latency | ~100 ns |
| SSD read | ~10-100 µs (микросекунд) |
| HDD seek | ~10 ms |
| Network (same DC) | ~1 ms |
| Network (cross-US) | ~50-150 ms |
| 1 Gbps link | 125 MB/s |
| Read 1MB sequential (SSD) | ~1 ms |
| Read 1MB sequential (RAM) | ~5 µs |

### Расчёт для URL Shortener

```
Дано:
  - 100 млн новых ссылок/месяц
  - Хранение: 5 лет → 6 млрд записей

Одна запись:
  - Short key: 7 символов (Base62) = 7 байт
  - Original URL: среднее 100 байт
  - User ID: 8 байт
  - Created at: 8 байт
  - Expires at: 8 байт
  - Итого: ~150 байт/запись

Storage:
  - 6 млрд × 150 байт = 900 GB (чистые данные)
  - С индексами: ~1.5 TB
  - Репликация ×3: ~4.5 TB

Bandwidth:
  - Write: 40 RPS × 150 байт = 6 KB/s (входящий)
  - Read: 400 RPS × (150 байт + redirect response ~300 байт) = 180 KB/s (исходящий)
  - Это НИЧТОЖНАЯ нагрузка — можно на одном сервере

Cache:
  - 20% ссылок дают 80% переходов (Pareto principle)
  - Горячих ссылок: 20% × 6 млрд = 1.2 млрд
  - Кешируем топ 10%: 120 млн × 150 байт = 18 GB
  - Redis кластер: 2-3 инстанса по 16 GB
```

---

## Шаг 3: High-Level Design

### Диаграмма: URL Shortener

```
                        ┌──────────────┐
                        │   CDN/DNS    │
                        └──────┬───────┘
                               │
                        ┌──────▼───────┐
                        │ Load Balancer│
                        └──┬────────┬──┘
                           │        │
              ┌────────────▼┐  ┌────▼────────────┐
              │  Write API  │  │   Read API       │
              │  (40 RPS)   │  │   (400 RPS)      │
              └──────┬──────┘  └────┬─────────────┘
                     │              │
         ┌───────────▼──────┐  ┌───▼──────────────┐
         │ Key Generation   │  │  Redis Cache     │
         │ Service          │  │  (18 GB, 3 node) │
         │ (Snowflake-like) │  └───┬──────────────┘
         └───────┬──────────┘      │
                 │          ┌──────▼──────────────┐
         ┌───────▼──────────▼─────────────────────┐
         │         PostgreSQL (Primary)           │
         │         + Read Replicas (2×)           │
         │         Партиционирование по дате      │
         └────────────────────────────────────────┘
```

---

## Шаг 4: Детальный дизайн компонента — Key Generation

### Требования к ключу

```
- Уникальный (глобально)
- Короткий (6-8 символов)
- Непредсказуемый (защита от перебора)
- Быстрая генерация (без проверки на уникальность в БД)
```

### Варианты генерации

**Вариант 1: Хеш от URL (MD5/SHA256) + Base62 (первые 7 символов)**

```
Плюсы: не требует БД для генерации
Минусы: коллизии — нужно проверять уникальность
```

**Вариант 2: Счётчик + Base62 кодирование**

```
Плюсы: гарантированная уникальность, последовательная генерация
Минусы: единая точка отказа (single counter), предсказуемость
```

**Вариант 3: Snowflake-like ID (выбор для high-scale)**

```
ID = 64 бита:
┌────────────┬──────────┬──────────┬──────────┐
│ timestamp  │ machine  │ sequence │ random   │
│ 41 bits    │ 10 bits  │ 12 bits  │ 1 bit    │
│ (ms)       │ (1024    │ (4096/ms)│          │
│            │  nodes)  │          │          │
└────────────┴──────────┴──────────┴──────────┘

Затем: ID → Base62 → 7 символов

Плюсы:
  - Распределённая генерация (нет центрального счётчика)
  - Упорядоченность по времени (timestamp в старших битах)
  - Высокая производительность (4K ID/сек на ноду)

Минусы:
  - Зависимость от настройки clock sync
```

```csharp
// Упрощённый Snowflake для URL Shortener
public class SnowflakeKeyGenerator
{
    private static readonly long Epoch = new DateTime(2025, 1, 1).Ticks;
    private static readonly long MachineId = GetMachineId(); // 10 bits
    private long _lastTimestamp = -1;
    private long _sequence = 0;

    public string GenerateKey()
    {
        long timestamp = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds();
        lock (this) // защита от race в пределах одной ноды
        {
            if (timestamp == _lastTimestamp)
                _sequence = (_sequence + 1) & 0xFFF; // 12 bits
            else
                _sequence = 0;

            _lastTimestamp = timestamp;
        }

        long id = (timestamp << 22) | (MachineId << 12) | _sequence;
        return EncodeBase62(id);
    }

    private static string EncodeBase62(long value)
    {
        const string chars = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz";
        var result = new char[7];
        for (int i = 6; i >= 0; i--)
        {
            result[i] = chars[(int)(value % 62)];
            value /= 62;
        }
        return new string(result);
    }
}
```

---

## Шаг 5: Отказоустойчивость (Fault Tolerance)

### Стратегии для URL Shortener

```
1. Read Path (критично — если не работает, ссылки "лежат"):
   - CDN с TTL для сверхпопулярных ссылок
   - Redis Cluster с репликацией (3 ноды, RF=2)
   - PostgreSQL read replicas (2+)
   - Circuit Breaker на Redis → fallback в PostgreSQL
   - Retry с exponential backoff

2. Write Path (менее критично — можно повторить):
   - Key generation — несколько независимых нод Snowflake
   - PostgreSQL primary с synchronous replication на standby
   - Outbox pattern: записать в БД → async опубликовать событие

3. Общие механизмы:
   - Health checks + auto-restart (K8s liveness/readiness probes)
   - Graceful degradation: при отказе кеша — работаем через БД
   - Rate limiting (token bucket per API key)
   - Multi-AZ deployment
```

### SLO/SLI

| Индикатор | Цель |
|-----------|------|
| Availability | 99.99% (~52 минуты простоя/год) |
| Latency p50 | < 5ms |
| Latency p95 | < 20ms |
| Latency p99 | < 50ms |
| Error rate | < 0.01% |

---

## Шаг 6: Схема БД

```sql
CREATE TABLE urls (
    id BIGINT PRIMARY KEY,           -- Snowflake ID
    short_key VARCHAR(8) UNIQUE NOT NULL,
    original_url TEXT NOT NULL,
    user_id BIGINT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    expires_at TIMESTAMPTZ,
    click_count BIGINT DEFAULT 0
);

-- Индексы:
CREATE UNIQUE INDEX idx_short_key ON urls(short_key);
CREATE INDEX idx_user_id ON urls(user_id);
CREATE INDEX idx_expires_at ON urls(expires_at)
    WHERE expires_at IS NOT NULL;

-- Партиционирование по месяцам:
CREATE TABLE urls (
    ...
) PARTITION BY RANGE (created_at);

CREATE TABLE urls_2025_01 PARTITION OF urls
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
```

---

## Чек-лист: System Design Interview

- [ ] Собрал функциональные и нефункциональные требования
- [ ] Оценил QPS, storage, bandwidth (back-of-the-envelope)
- [ ] Нарисовал high-level архитектурную диаграмму
- [ ] Описал data flow (write path + read path)
- [ ] Определил API (REST/gRPC endpoints)
- [ ] Спроектировал схему БД (таблицы + индексы)
- [ ] Погрузился в 1-2 ключевых компонента (детально)
- [ ] Продумал отказоустойчивость (failover, replication, circuit breaker)
- [ ] Оценил мониторинг и эксплуатацию
- [ ] Обсудил trade-offs своих решений
- [ ] Ответил на вопросы интервьюера

### Шпаргалка: числовые оценки

```
Доступность:
  99%     = 3.65 дня/год простоя
  99.9%   = 8.76 часа/год
  99.99%  = 52.5 минуты/год
  99.999% = 5.25 минуты/год

Запрос в секунду:
  1K RPS  × 86,400 сек = 86.4M/день
  10K RPS × 86,400 сек = 864M/день
  100K RPS = ~8.6B/день → нужен distributed system

Хранение:
  1 TB HDD/SSD на сервер (типично)
  Redis: 1M ключей × 1KB ≈ 1 GB RAM
  PostgreSQL: 10M строк × 200 байт ≈ 2 GB + индексы
```

# Kafka Advanced — тонкости для собеседования

---

## Exactly-Once Semantics — глубокое погружение

### Три кита exactly-once в Kafka

```
1. Idempotent Producer          — нет дубликатов при ретраях продюсера
2. Transactional Producer       — атомарная запись в несколько партиций
3. Exactly-Once Streams         — атомарный read-process-write
```

### Idempotent Producer — как работает

```
Включение: enable.idempotence = true

Механизм:
  - Producer получает Producer ID (PID) от брокера при инициализации
  - Каждое сообщение получает (PID, epoch, sequence number)
  - Брокер отслеживает последние 5 sequence number для каждого PID/Partition
  - При дубликате sequence number → сообщение отбрасывается, но ACK возвращается

Ограничения:
  - Не кросс-сессионная идемпотентность (после перезапуска Producer новый PID)
  - max.in.flight.requests.per.connection должен быть ≤ 5 (иначе нарушение порядка)
  - Только в пределах одного кластера
```

### Transactional Producer

```java
// Инициализация
producer.initTransactions();

// Атомарная запись в несколько топиков
producer.beginTransaction();
try {
    producer.send(new ProducerRecord<>("orders", orderJson));
    producer.send(new ProducerRecord<>("order-events", eventJson));
    producer.send(new ProducerRecord<>("notifications", notifyJson));
    producer.commitTransaction();
} catch (Exception e) {
    producer.abortTransaction();
}

// Transactional ID = логический идентификатор (переживает рестарты)
// config: transactional.id = "order-service-01"
```

```
Как работает transactional producer:
  1. Producer регистрирует transactional.id → получает PID + epoch
  2. beginTransaction() — начало транзакции
  3. Все сообщения записываются, но НЕ видны consumer'ам с isolation.level=read_committed
  4. commitTransaction() — брокер записывает commit marker
  5. Сообщения становятся видимыми (read_committed consumers)

Fencing:
  - Если producer с тем же transactional.id перезапускается → новый epoch
  - Старый producer с меньшим epoch "отгораживается" (fenced)
  - Гарантия: ровно один активный producer для transactional.id
```

### Consumer isolation levels

```
isolation.level = read_uncommitted  (по умолчанию):
  - Видит все сообщения, включая незакоммиченные транзакции

isolation.level = read_committed:
  - Видит только закоммиченные транзакции
  - + все нетранзакционные сообщения
```

---

## Log Compaction

### Как работает compaction

```
Обычное удаление (retention):
  Segment 1 (старый) → удаляется целиком по истечении retention.ms

Compaction (лог как KV-store):
  Для каждого ключа сохраняется ТОЛЬКО последнее значение
  Удаление: сообщение с value=null (tombstone)

До compaction:
  key=A, value=1
  key=B, value=2
  key=A, value=3    ← перезаписывает предыдущий A
  key=C, value=4
  key=A, value=null  ← удаляет A

После compaction:
  key=B, value=2
  key=C, value=4
  (A удалён, старые значения A удалены)
```

### Настройки compaction

```
cleanup.policy = compact        — только compaction (без временного retention)
cleanup.policy = compact,delete — compaction + временной retention

min.cleanable.dirty.ratio = 0.5 — начать compaction когда 50% "грязных" данных
min.compaction.lag.ms = 0       — минимальный возраст сегмента до compaction
delete.retention.ms = 86400000  — tombstone держится 24ч перед удалением
                                  (чтобы все consumer'ы успели увидеть удаление)
```

### Применение compaction

```
1. CDC (Change Data Capture): хранение последнего состояния каждой записи
2. KTable в Kafka Streams: восстановление состояния после ребаланса
3. __consumer_offsets: хранение offset'ов consumer groups
4. Конфигурация сервисов: топик с конфигурацией, consumer'ы читают последнюю
```

---

## KRaft — Kafka без ZooKeeper

### Проблемы ZooKeeper

```
- Отдельный кластер для управления (сложность эксплуатации)
- Метаданные в ZooKeeper, данные в Kafka → два несогласованных источника
- ZooKeeper не масштабируется горизонтально (пишет один лидер)
- Медленная загрузка метаданных при большом числе партиций
- Разные протоколы безопасности для ZK и Kafka
```

### KRaft (Kafka Raft) — архитектура

```
KRaft mode (Kafka 3.3+ — production-ready):

┌──────────────────────────────────────────────────────┐
│                  KRAFT CLUSTER                       │
│                                                      │
│  Controller Nodes (Quorum: 3 или 5)                  │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐    │
│  │ Controller │  │ Controller │  │ Controller │    │
│  │ (Leader)   │  │ (Follower) │  │ (Follower) │    │
│  └────────────┘  └────────────┘  └────────────┘    │
│                                                      │
│  Broker Nodes                                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │ Broker 1 │  │ Broker 2 │  │ Broker 3 │          │
│  └──────────┘  └──────────┘  └──────────┘          │
│                                                      │
│  Метаданные хранятся в __cluster_metadata topic       │
│  (внутренний топик с одним партицией)                 │
└──────────────────────────────────────────────────────┘

Controller может быть совмещён с Broker (combined mode)
или выделенный (dedicated mode — рекомендуется для production)
```

### Преимущества KRaft

```
- Единый кластер (нет ZK)
- Метаданные — как обычный топик (репликация, снапшоты)
- Быстрый recovery (миллионы партиций за секунды, не минуты)
- Единая модель безопасности
- Controller failover за ~100ms (vs секунды с ZK)
```

---

## Producer Tuning

### Ключевые параметры

```
# Пропускная способность vs Latency
linger.ms = 5           # Ждать до 5ms для батчинга (0 = отправлять сразу)
batch.size = 16384      # Размер батча в байтах (16384 = 16KB)
buffer.memory = 33554432 # Буфер продюсера (32MB)

# Надёжность
acks = all              # Ждать подтверждения от всех ISR
retries = 2147483647    # Максимум ретраев (Int32.MaxValue)
enable.idempotence = true
max.in.flight.requests.per.connection = 5

# Сжатие
compression.type = lz4  # lz4 > snappy (скорость) > gzip (размер) > zstd (баланс)
                         # zstd = лучший компромисс скорость/сжатие (Kafka 2.1+)

# Партиционирование
partitioner.class = org.apache.kafka.clients.producer.internals.DefaultPartitioner
# Sticky partitioner (Kafka 2.4+): батчи в одну партицию, затем меняет
# Вместо round-robin (каждое сообщение — новая партиция)
```

### Анти-паттерны producer

```
❌ acks=0 + ретраи — потеря данных, ретраи бесполезны
❌ max.in.flight > 5 с idempotence=false — нарушение порядка при ретраях
❌ синхронная отправка в цикле (future.get()) — низкая пропускная способность
❌ слишком большой batch.size с маленьким linger.ms — батчи не заполняются
❌ один producer на все топики — проблемы с изоляцией ошибок
```

---

## Consumer Tuning

### Ключевые параметры

```
# Pull
fetch.min.bytes = 1           # Минимум байт в ответе fetch (1 = сразу)
fetch.max.bytes = 52428800    # Максимум байт в ответе (50MB)
fetch.max.wait.ms = 500       # Ждать до 500ms если min.bytes не достигнут
max.partition.fetch.bytes = 1048576  # Максимум байт с одной партиции (1MB)

# Heartbeat & Session
session.timeout.ms = 45000    # Без heartbeat → мёртв (был 10s, ↑ до 45s)
heartbeat.interval.ms = 3000  # heartbeat каждые 3 секунды
max.poll.interval.ms = 300000 # Максимум между poll() (5 мин)

# Offset
enable.auto.commit = false
auto.offset.reset = earliest  # earliest — сначала; latest — новые
```

### Как избежать rebalance

```
Причина частых ребалансов: медленная обработка → max.poll.interval.ms истекает

Решение 1: Увеличить max.poll.interval.ms
Решение 2: Уменьшить max.poll.records (обрабатывать меньше за poll)
Решение 3: Вынести обработку в отдельный thread pool (не блокировать poll loop)
Решение 4: Cooperative rebalancing (Kafka 2.4+)

CooperativeStickyAssignor:
  - Не отзывает ВСЕ партиции при ребалансе
  - Передаёт только те, что нужно перераспределить
  - Потребитель продолжает читать оставшиеся партиции
```

```java
// Cooperative rebalancing
props.put(ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG,
    "org.apache.kafka.clients.consumer.CooperativeStickyAssignor");
```

---

## Мониторинг Kafka

### Ключевые метрики

```
Producer:
  - record-send-rate          — сообщений/сек
  - record-error-rate         — ошибок/сек
  - request-latency-avg       — средняя latency запроса
  - outgoing-byte-rate        — исходящий трафик
  - buffer-available-bytes    — свободно в буфере (если 0 → backpressure)

Consumer:
  - records-consumed-rate     — сообщений/сек потребляется
  - records-lag-max           — МАКСИМАЛЬНЫЙ lag по партициям группы
  - fetch-rate                — fetch запросов/сек
  - bytes-consumed-rate       — входящий трафик

Broker:
  - UnderReplicatedPartitions — партиции с неполной репликацией (>0 → тревога!)
  - ActiveControllerCount     — должен быть ровно 1
  - OfflinePartitionsCount    — офлайн партиции (>0 → критично!)
  - NetworkProcessorAvgIdlePercent — загрузка сетевых потоков
  - RequestHandlerAvgIdlePercent — загрузка обработчиков запросов
```

### Lag мониторинг (consumer lag)

```
Burrow (LinkedIn) — специализированный мониторинг lag:
  - Алгоритм: не просто текущий lag, а СКОРОСТЬ изменения lag
  - Статусы: OK, WARNING, ERROR
  - WARNING: lag растёт быстрее, чем consumer потребляет
  - ERROR: consumer остановился (lag растёт, offset не меняется)

Формула оценки:
  time_to_clear = lag / consumption_rate
  Если time_to_clear > порог (например, 1 час) → алерт
```

---

## Чек-лист: Kafka Advanced

- [ ] Idempotent Producer: как работает dedup? Что такое PID, epoch, sequence?
- [ ] Transactional Producer: как обеспечить exactly-once запись в несколько топиков?
- [ ] Consumer isolation levels: read_committed vs read_uncommitted
- [ ] Log Compaction: как работает, когда нужен, tombstone retention
- [ ] KRaft vs ZooKeeper: архитектура, преимущества, migration path
- [ ] Producer tuning: linger.ms + batch.size + compression
- [ ] Consumer tuning: session/heartbeat таймауты, как избежать ребалансов
- [ ] CooperativeStickyAssignor: чем лучше RangeAssignor/RoundRobin?
- [ ] Мониторинг: UnderReplicatedPartitions, consumer lag, Burrow
- [ ] MirrorMaker 2: репликация между кластерами (geo-replication)
- [ ] Tiered Storage: разгрузка старых сегментов в S3/HDFS

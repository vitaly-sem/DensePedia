# Kafka и очереди сообщений

---

## Архитектура Kafka — ключевые компоненты

```
┌─────────────────────────────────────────────────────────────┐
│                     KAFKA CLUSTER                           │
│                                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                 │
│  │ Broker 1 │  │ Broker 2 │  │ Broker 3 │                 │
│  │          │  │          │  │          │                 │
│  │ P0 (L)   │  │ P0 (F)   │  │ P1 (L)   │   Topic: orders│
│  │ P1 (F)   │  │ P1 (F)   │  │ P0 (F)   │   Partitions: 3│
│  │ P2 (L)   │  │ P2 (F)   │  │ P2 (F)   │   RF: 3        │
│  └──────────┘  └──────────┘  └──────────┘                 │
│                                                             │
│  ZooKeeper / KRaft — координация, выборы лидера            │
└─────────────────────────────────────────────────────────────┘

  L = Leader    F = Follower    P0..P2 = Partition 0..2
```

### Топик, партиции, сегменты

```
Topic "orders":
  ├── Partition 0 (Leader: Broker1)
  │   ├── Segment 0: offset 0..1000
  │   ├── Segment 1: offset 1001..2000
  │   └── Segment 2: offset 2001..3000
  ├── Partition 1 (Leader: Broker3)
  │   └── ...
  └── Partition 2 (Leader: Broker1)
      └── ...

Ключевые свойства:
  - Партиции — единица параллелизма (пишут и читают конкурентно)
  - Порядок сообщений гарантирован ТОЛЬКО внутри одной партиции
  - Ключ сообщения определяет партицию: partition = hash(key) % partition_count
  - Сегменты — физические файлы на диске (log.segment.bytes)
```

### ISR (In-Sync Replicas)

```
ISR = множество реплик, которые синхронизированы с лидером

Настройки:
  - replica.lag.time.max.ms = 30000 (если реплика отстала >30с — исключается из ISR)
  - min.insync.replicas = 2 (минимум реплик для подтверждения записи)

acks = all + min.insync.replicas = 2:
  Producer ждёт подтверждения от лидера И минимум 2 реплик в ISR.
  Защита от потери данных при отказе лидера.
```

---

## Producer — отправка сообщений

### Ключевые настройки

```
acks = 0:
  - Producer не ждёт подтверждения
  - Максимальная скорость, возможна потеря

acks = 1:
  - Producer ждёт подтверждения от лидера
  - Лидер может упасть до репликации → потеря

acks = all (или -1):
  - Producer ждёт подтверждения от всех ISR
  - Гарантия: данные не потеряны при отказе лидера
  - Требует min.insync.replicas >= 2

enable.idempotence = true:
  - Идемпотентный продюсер (exactly-once семантика)
  - Присваивает Producer ID и sequence number каждому сообщению
  - Брокер дедуплицирует по (ProducerID, sequence)
```

### Выбор партиции

```java
// Java (Kafka Producer API):
// Ключ определяет партицию:
producer.send(new ProducerRecord<>("orders", orderId, orderJson));

// Без ключа — round-robin (или sticky partitioning в новых версиях):
producer.send(new ProducerRecord<>("orders", orderJson));

// Явно указать партицию:
producer.send(new ProducerRecord<>("orders", 2, orderId, orderJson));
```

### Batching и сжатие

```
linger.ms = 5:
  - Producer ждёт до 5ms для накопления батча
  - Увеличивает throughput, но добавляет latency

batch.size = 16384 (16KB):
  - Размер батча в байтах

compression.type = lz4:
  - Сжатие батчей (lz4 — хороший баланс скорость/сжатие)
  - snappy — быстрее, но хуже сжатие
  - gzip — лучше сжатие, но медленнее
  - zstd — отличное сжатие и скорость (Kafka 2.1+)
```

---

## Consumer — чтение сообщений

### Consumer Group

```
┌─────────────────────────────────────────────┐
│              CONSUMER GROUP                  │
│                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │Consumer 1│  │Consumer 2│  │Consumer 3│  │
│  │ P0, P1   │  │ P2, P3   │  │ P4       │  │
│  └──────────┘  └──────────┘  └──────────┘  │
│                                              │
│  Topic: orders (5 partitions)                │
│  Правило: Consumers ≤ Partitions             │
│           Если Consumers > Partitions →      │
│           лишние consumers простаивают        │
└─────────────────────────────────────────────┘

Правила consumer group:
  - Каждая партиция назначается РОВНО одному consumer в группе
  - Один consumer может читать много партиций
  - Разные consumer groups читают независимо (каждая группа — свой offset)
```

### Offset Management

```
offset = позиция consumer'а в партиции

Стратегии коммита:

1. Auto-commit (enable.auto.commit = true):
   - Коммит каждые auto.commit.interval.ms (5 сек)
   - At-most-once: если consumer упал между auto-commit'ами → дубликаты
   - At-least-once: обработал → умер до коммита → перечитает (дубликат)

2. Manual commit (enable.auto.commit = false):
   - Явный вызов consumer.commitSync() или commitAsync()
   - Полный контроль над моментом коммита

3. Хранение offset'ов в Kafka (__consumer_offsets):
   - По умолчанию offsets хранятся в компактном топике __consumer_offsets
   - Можно хранить offsets во внешней системе (БД)
```

### Rebalancing

```
Rebalance — перераспределение партиций между consumer'ами

Триггеры ребаланса:
  - Новый consumer присоединился к группе
  - Consumer покинул группу (упал или закрылся)
  - Увеличение числа партиций топика
  - session.timeout.ms истёк (не было heartbeat)

Стратегии назначения:
  - RangeAssignor (по умолчанию): последовательные партиции
  - RoundRobinAssignor: равномерное распределение
  - StickyAssignor: минимальное перераспределение при ребалансе
  - CooperativeStickyAssignor (Kafka 2.4+): инкрементальный ребаланс без остановки

Во время ребаланса: потребители НЕ читают сообщения!
```

---

## Семантика доставки (Delivery Semantics)

### At-most-once (не более одного раза)

```
Настройка: auto-commit + обработка ПОСЛЕ коммита

Consumer: poll() → commit() → process()
Риск: если process() упадёт — сообщение ПОТЕРЯНО (уже закоммичено)

Применение: метрики, логи (потеря некритична)
```

### At-least-once (как минимум один раз) — СТАНДАРТ

```
Настройка: ручной коммит ПОСЛЕ обработки

Consumer: poll() → process() → commit()
Риск: если процесс упадёт после process() но до commit() → ДУБЛИКАТ

Решение: идемпотентная обработка (idempotency key в БД)
```

### Exactly-once (ровно один раз)

```
Три уровня:

1. Idempotent Producer (enable.idempotence = true):
   - Producer не создаёт дубликаты при ретраях
   - Гарантия: в пределах одной партиции, одной сессии

2. Transactional Producer:
   - Атомарная запись в несколько топиков/партиций
   - producer.initTransactions() + beginTransaction() + commitTransaction()

3. Exactly-once Stream Processing (Kafka Streams):
   - Комбинация transactional producer + consumer
   - Atomic read-process-write
   - processing.guarantee = exactly_once_v2

На практике: at-least-once + идемпотентный consumer (dedup в БД)
               проще, чем exactly-once Kafka.
```

---

## Kafka vs другие брокеры сообщений

### Сравнительная таблица

| Характеристика | Kafka | RabbitMQ | Redis Pub/Sub |
|---------------|--------|----------|---------------|
| Модель | Distributed log | Message broker (AMQP) | In-memory pub/sub |
| Хранение | На диске (persistent) | Опционально | В памяти (не durable) |
| Throughput | ~1M msg/sec | ~50K msg/sec | ~100K msg/sec |
| Latency | 2-5 ms | < 1 ms | < 1 ms |
| Порядок | Per partition | Per queue (FIFO) | Не гарантирован |
| Replay | Да (offset) | Нет | Нет |
| Масштабирование | Партиции, consumer groups | Кластеризация + queues | Нет (single-threaded) |
| Гарантии | At-least-once, exactly-once | At-least-once, at-most-once | Fire-and-forget |
| Протокол | Binary (custom) | AMQP 0-9-1, STOMP, MQTT | RESP |
| Push/Pull | Pull-based (long poll) | Push-based | Push-based |
| Сложность | Высокая | Средняя | Низкая |

### Когда что выбирать

```
Kafka:
  - Event sourcing, stream processing
  - Большие объёмы данных (>100K msg/sec)
  - Replay данных, audit log
  - Интеграция микросервисов (асинхронная)
  - Много consumer groups на один топик

RabbitMQ:
  - RPC/запрос-ответ (reply-to)
  - Сложная маршрутизация (exchanges, bindings)
  - Приоритетные очереди, TTL, DLX
  - Традиционные очереди задач
  - Когда нужна гибкая маршрутизация

Redis Pub/Sub:
  - Быстрые уведомления в реальном времени
  - Потеря сообщений допустима
  - Простая архитектура
```

---

## Kafka Streams — обработка потоков

```
Kafka Streams — библиотека (не отдельный кластер!)

KStream: поток записей (каждое событие независимо)
KTable: таблица (текущее состояние по ключу)
GlobalKTable: реплицированная на все инстансы таблица

Пример: подсчёт заказов по статусам
```

```java
// Kafka Streams: подсчёт количества заказов по статусам
StreamsBuilder builder = new StreamsBuilder();

KStream<String, Order> orders = builder.stream("orders");

KTable<String, Long> countByStatus = orders
    .map((key, order) -> new KeyValue<>(order.getStatus(), order))
    .groupByKey()
    .count();

countByStatus.toStream().to("order-counts-by-status");

// Exactly-once streams:
Properties props = new Properties();
props.put(StreamsConfig.PROCESSING_GUARANTEE_CONFIG, "exactly_once_v2");
```

---

## Типичные проблемы и их решение

### Проблема 1: Дубликаты сообщений

```
Причина: consumer обработал, но не закоммитил offset → перезапуск → дубликат

Решение:
  1. Идемпотентный consumer — dedup ключ в БД
  2. INSERT ... ON CONFLICT DO NOTHING
  3. Проверка уникальности перед обработкой
```

```csharp
async Task ProcessMessage(Message msg)
{
    // Dedup через уникальный constraint в БД
    await db.ExecuteAsync(
        "INSERT INTO processed_messages (idempotency_key) VALUES (@key) " +
        "ON CONFLICT DO NOTHING", new { key = msg.IdempotencyKey });

    if (rows affected == 0) return; // Уже обработано

    await ProcessOrderAsync(msg);
}
```

### Проблема 2: Lag (отставание consumer)

```
Причины:
  - Consumer медленнее producer
  - Долгая обработка одного сообщения
  - Ребалансы

Мониторинг:
  kafka-consumer-groups --bootstrap-server localhost:9092 \
    --group my-group --describe

  # Смотрим колонку LAG — отставание в сообщениях

Решения:
  - Увеличить число consumer'ов (но не больше партиций!)
  - Увеличить число партиций
  - Оптимизировать обработку (батчи, async)
  - Увеличить fetch.max.bytes, fetch.min.bytes
```

### Проблема 3: Потеря сообщений

```
Сценарий: leader упал, follower отстал → потеря

Защита:
  - acks = all (producer)
  - min.insync.replicas >= 2 (брокер)
  - replication.factor >= 3 (топик)
  - unclean.leader.election.enable = false (не выбирать отставшие реплики)
```

---

## Чек-лист: Kafka на собеседовании

- [ ] Архитектура: brokers, topics, partitions, consumer groups
- [ ] Как обеспечивается порядок сообщений? (Только внутри партиции)
- [ ] Что такое ISR и зачем нужен?
- [ ] Семантика доставки: at-most-once, at-least-once, exactly-once
- [ ] Как работает идемпотентный producer?
- [ ] Offset management: auto-commit vs manual commit
- [ ] Consumer group rebalance — триггеры и стратегии
- [ ] Kafka vs RabbitMQ: когда что выбрать?
- [ ] Как гарантировать exactly-once при обработке?
- [ ] Kafka Streams: KStream vs KTable vs GlobalKTable
- [ ] Lag мониторинг и решение проблем отставания
- [ ] ZooKeeper vs KRaft (Kafka Raft) — управление метаданными
- [ ] Compaction в топиках: что это и зачем?

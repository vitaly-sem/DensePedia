# System Design — разбор реальных кейсов

---

## Кейс 1: Проектируем Twitter (News Feed)

### Требования

```
Функциональные:
  - Пользователь публикует твит (280 символов + медиа)
  - Пользователь видит ленту (твиты тех, на кого подписан)
  - Пользователь подписывается / отписывается
  - Лайки, ретвиты, ответы

Нефункциональные:
  - 200M DAU
  - 100M новых твитов / день (~1200 RPS write)
  - 500M чтений ленты / день (~6000 RPS read)
  - Latency рендера ленты: p95 < 200ms
  - Availability: 99.99%
```

### High-Level Design

```
┌────────────────────────────────────────────────────────────┐
│                       TWITTER                              │
│                                                            │
│  ┌──────────┐   Write   ┌──────────────┐                 │
│  │ User     │──────────→│ Tweet Service │                 │
│  │ (Client) │           │ (1200 RPS)   │                 │
│  └────┬─────┘           └──────┬───────┘                 │
│       │                        │                          │
│       │ Read (6000 RPS)        ▼                          │
│       │                ┌──────────────┐                  │
│       └───────────────→│ NewsFeed Svc │                  │
│                        │              │                  │
│                        └──────┬───────┘                  │
│                               │                           │
│                    ┌──────────┼──────────┐               │
│                    ▼          ▼          ▼               │
│              ┌─────────┐┌─────────┐┌─────────┐          │
│              │ Redis   ││ Redis   ││ Redis   │          │
│              │ Timeline││ Timeline││ Timeline│          │
│              │ Cache   ││ Cache   ││ Cache   │          │
│              └─────────┘└─────────┘└─────────┘          │
│                               │                           │
│                    ┌──────────▼──────────┐               │
│                    │   PostgreSQL        │               │
│                    │   Tweets + Users    │               │
│                    │   + Graph DB (follows)│             │
│                    └─────────────────────┘               │
└────────────────────────────────────────────────────────────┘
```

### Feed Generation — два подхода

```
Подход 1: Fan-out on Write (при публикации)
  - При публикации твита → кешируется в ленту ВСЕХ подписчиков
  - Чтение: один запрос к кешу → O(1)
  - Проблема: пользователь с 10M подписчиков → 10M записей в кеш
  - Решение: гибридный подход

Подход 2: Fan-out on Read (при запросе ленты)
  - При запросе: собрать твиты от всех following
  - Чтение: N запросов (по числу following) → медленно для многих
  - Преимущество: нет накладных расходов на запись

Гибридный подход (Twitter использует):
  - Обычные пользователи: Fan-out on Write
  - "Знаменитости" (>100K followers): Fan-out on Read
  - При чтении: мёржим закешированную ленту + "injected" твиты celebrities
```

### Схема БД

```sql
-- Основные таблицы
CREATE TABLE tweets (
    id BIGINT PRIMARY KEY,         -- Snowflake ID
    user_id BIGINT NOT NULL,
    text VARCHAR(280),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    is_deleted BOOLEAN DEFAULT FALSE
);

CREATE TABLE follows (
    follower_id BIGINT NOT NULL,
    followee_id BIGINT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (follower_id, followee_id)
);

-- Кеш ленты в Redis: List (sorted set по времени)
-- Ключ: timeline:{user_id}
-- Значение: tweet_id (последние ~800 твитов)
-- ZADD timeline:123 timestamp tweet_id
-- ZREVRANGEBYSCORE timeline:123 +inf -inf LIMIT 0 20
```

### Медиа (изображения/видео)

```
Pipeline загрузки:

1. Client загружает медиа → Media Service
2. Media Service:
   - Сохраняет оригинал в S3/Blob Storage
   - Асинхронно генерирует ресайзы (thumbnail, medium, large)
   - Возвращает media_id
3. Client прикрепляет media_id к твиту

CDN для раздачи:
  - CloudFront / Cloudflare
  - URL: https://media.twitter.com/{media_id}_{size}.jpg
  - TTL: бесконечный (immutable media)
```

---

## Кейс 2: Проектируем Uber (Ride Matching)

### Требования

```
Функциональные:
  - Пассажир запрашивает поездку (pickup → destination)
  - Система находит ближайшего водителя
  - Водитель принимает / отклоняет
  - Отслеживание поездки в реальном времени
  - Оплата через приложение

Нефункциональные:
  - 50M поездок / день (~600 RPS новых запросов)
  - Matching: < 5 секунд
  - GPS обновления от водителей: каждые 4 секунды
  - 5M активных водителей → 1.25M GPS обновлений / сек
```

### Архитектура

```
┌─────────────────────────────────────────────────────────────┐
│                        UBER                                 │
│                                                             │
│  ┌───────────┐   GPS update   ┌──────────────────┐        │
│  │ Driver App│───────────────→│ Location Service  │        │
│  │ (WebSocket)│←──────────────│ (WebSocket)       │        │
│  └───────────┘   trip update  └────────┬─────────┘        │
│                                        │                    │
│  ┌───────────┐   Ride request  ┌──────▼──────────┐       │
│  │ Rider App │────────────────→│ Ride Matching   │       │
│  │           │←────────────────│ Service         │       │
│  └───────────┘   Driver info   └────────┬─────────┘       │
│                                        │                    │
│                              ┌─────────▼──────────┐       │
│                              │    Geo Index        │       │
│                              │ (Google S2 / H3)    │       │
│                              └────────────────────┘       │
│                                        │                    │
│                              ┌─────────▼──────────┐       │
│                              │    Kafka            │       │
│                              │ (GPS stream, trips) │       │
│                              └────────────────────┘       │
└─────────────────────────────────────────────────────────────┘
```

### Geo-индексация — Google S2 / Uber H3

```
Проблема: как найти N ближайших водителей за O(log n)?

Решение: Geohash / Google S2 / Uber H3

S2 (Google): иерархическая сетка на сфере
  - Ячейки уровня 30: ~1cm²
  - Ячейки уровня 15: ~300m² (используется Uber)
  
H3 (Uber): гексагональная сетка
  - Гексагоны лучше чем квадраты (равное расстояние до соседей)
  - 16 уровней разрешения
  - Уровень 12: ~0.3 km² (hexagon edge ~307m)

Алгоритм поиска:
  1. Определить H3-ячейку пассажира (уровень 12)
  2. Расширять поиск: ячейка → соседние ячейки → ring 2 → ring 3...
  3. Собрать driver_id в этих ячейках
  4. Вычислить точное расстояние (Haversine)
  5. Отсортировать, отфильтровать недоступных
```

```csharp
// Псевдокод поиска ближайших водителей
List<Driver> FindNearbyDrivers(double lat, double lon, int limit)
{
    var hex = H3.GeoToH3(lat, lon, resolution: 12);
    var drivers = new List<Driver>();
    int ring = 0;

    while (drivers.Count < limit && ring <= 5)
    {
        var hexRing = ring == 0
            ? new[] { hex }
            : H3.HexRing(hex, ring);

        foreach (var h in hexRing)
        {
            var cellDrivers = redis.SetMembers($"drivers:{h}");
            drivers.AddRange(cellDrivers.Select(ParseDriver));
        }
        ring++;
    }

    // Точная сортировка по Haversine
    return drivers
        .Distinct()
        .Where(d => d.IsAvailable)
        .OrderBy(d => Haversine(lat, lon, d.Lat, d.Lon))
        .Take(limit)
        .ToList();
}
```

### Оценка ресурсов

```
GPS обновления:
  - 5M водителей × 1/4 Гц = 1.25M сообщений/сек
  - Размер: driver_id(8) + lat(8) + lon(8) + timestamp(8) = 32 байта
  - Bandwidth: 1.25M × 32 = 40 MB/s (входящий)
  - Kafka: 1.25M msg/sec → ~20 партиций (по 60K msg/sec каждая)
  - Хранение GPS за 24ч: 1.25M × 86400 × 32 = 3.5 TB

Geo-индекс (Redis):
  - 5M водителей, H3 ячейки уровня 12 (~0.3 km²)
  - ~50K уникальных ячеек с водителями в городе
  - На ячейку: SET drivers:{hex} → driver_id, lat, lon, status
  - Память: 5M × 200 байт = 1 GB (помещается в Redis Cluster)

Ride Matching Service:
  - 600 RPS новых поездок
  - Поиск: чтение 3-10 ячеек Redis → ~5-10 мс
  - Общее время matching: < 100 мс
```

---

## Кейс 3: Проектируем мессенджер (WhatsApp)

### Требования

```
Функциональные:
  - 1-на-1 чаты и групповые чаты
  - Отправка текста, изображений, видео, документов
  - Индикаторы: доставлено, прочитано (✓✓ синие)
  - Online/Last Seen статус
  - Уведомления (push)

Нефункциональные:
  - 2B пользователей
  - 100B сообщений / день (~1.2M msg/sec)
  - Доставка: < 1 секунда
  - End-to-End Encryption (E2EE)
  - Сообщения не теряются
```

### Архитектура

```
┌─────────────────────────────────────────────────────────────┐
│                     MESSENGER                               │
│                                                             │
│  ┌──────────┐   WebSocket    ┌──────────────────┐         │
│  │ Client A │◄──────────────►│ Chat Gateway      │         │
│  └──────────┘                │ (WebSocket,      │         │
│                              │  persistent conn) │         │
│  ┌──────────┐   WebSocket    │                   │         │
│  │ Client B │◄──────────────►│                   │         │
│  └──────────┘                └────────┬──────────┘         │
│                                       │                     │
│                              ┌────────▼──────────┐         │
│                              │ Message Router    │         │
│                              │ (Kafka)           │         │
│                              └────────┬──────────┘         │
│                                       │                     │
│                    ┌──────────────────┼──────────────┐     │
│                    ▼                  ▼              ▼     │
│              ┌──────────┐    ┌──────────┐    ┌──────────┐ │
│              │ Message   │    │ Group     │    │ Media    │ │
│              │ Store     │    │ Service   │    │ Service  │ │
│              │ (HBase/   │    │           │    │ (S3)     │ │
│              │ Cassandra)│    │           │    │          │ │
│              └──────────┘    └──────────┘    └──────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### Message Flow

```
1. Client A отправляет сообщение через WebSocket:
   { type: "message", to: "user_B", content: "Привет!", msg_id: "uuid-123" }

2. Chat Gateway:
   - Валидирует сессию
   - Отправляет в Kafka (topic: raw-messages, partition: hash(to_user))
   - Возвращает ACK клиенту: { msg_id: "uuid-123", status: "sent" }

3. Message Router (Consumer Group на Kafka):
   - Сохраняет в Message Store (Cassandra):
     PRIMARY KEY ((chat_id), message_id DESC)
     с кластеризацией по времени (timeuuid)
   - Находит активные WebSocket-сессии получателя (Redis)
   - Отправляет сообщение в Chat Gateway → Client B

4. Client B получает сообщение → отправляет ACK:
   { type: "ack", msg_id: "uuid-123", status: "delivered" }

5. Client B открывает сообщение → отправляет Read Receipt:
   { type: "read", msg_id: "uuid-123" }

6. Read Receipt доставляется Client A
```

### Message Store — схема Cassandra

```sql
-- Cassandra оптимизирована для write-heavy сценариев
CREATE TABLE messages (
    chat_id BIGINT,              -- идентификатор чата
    message_id TIMEUUID,         -- timeuuid для сортировки
    sender_id BIGINT,
    content_type TEXT,           -- 'text', 'image', 'video'
    content TEXT,                -- текст или media_id
    created_at TIMESTAMP,
    PRIMARY KEY (chat_id, message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);
-- Запрос последних 50 сообщений чата: ОДИН partition read

CREATE TABLE unread_tracking (
    user_id BIGINT,
    chat_id BIGINT,
    last_read_message_id TIMEUUID,
    PRIMARY KEY (user_id, chat_id)
);
```

### Online Presence

```
Проблема: как отслеживать онлайн-статус 2B пользователей?

Решение: Heartbeat + Adaptive TTL

1. Клиент шлёт heartbeat каждые 30 секунд через WebSocket
2. Chat Gateway записывает: last_heartbeat = now()
3. Online статус: now() - last_heartbeat < 60 секунд

Redis:
  - Ключ: presence:{user_id}
  - Значение: { status: "online", last_seen: timestamp, device: "mobile" }
  - TTL: 60 секунд
  - При каждом heartbeat: обновляем TTL

Last Seen:
  - Когда WebSocket отключается → записываем в БД:
    INSERT INTO last_seen (user_id, last_seen_at) VALUES (?, now())
  - При запросе: если presence:{user_id} нет в Redis → читаем last_seen из БД
```

---

## Кейс 4: Проектируем YouTube (Video Platform)

### Оценка ресурсов

```
Дано:
  - 2B MAU
  - 500 часов видео загружается в минуту
  - Среднее видео: 10 минут, 500 MB (1080p)
  - Просмотров: 1B / день

Storage (загрузки):
  - 500 часов/мин × 60 = 30 000 часов/час
  - 30 000 × 24 × 365 = 262 800 000 часов/год
  - При 500 MB за 10 мин (50 MB/мин) → 3 GB/час
  - 262.8M часов × 3 GB/час = 788 PB/год исходного видео

Transcoding:
  - Каждое видео → 4 разрешения (144p, 360p, 720p, 1080p)
  - + кодек: H.264, VP9, AV1
  - Всего хранимых данных: 788 PB × 4 (разрешения) × 1.5 (кодеки) ≈ 4.7 EB/год

Bandwidth (просмотры):
  - 1B просмотров/день × 10 мин × 10 MB/мин = 100 PB/день
  - 100 PB / 86400 сек ≈ 1.16 TB/s = 9.3 Tbps
  - Это ОГРОМНАЯ цифра → CDN необходим
```

### Transcoding Pipeline

```
Загрузка видео:
  1. Client загружает → Upload Service → Object Storage (оригинал)
  2. Сообщение в очередь: { video_id, original_path, formats: [...] }
  
Transcoding Workers (Kubernetes Job):
  3. Скачивают оригинал из Object Storage
  4. Транскодируют (CPU/GPU intensive):
     - 144p → H.264 (маленький, для слабых соединений)
     - 360p → H.264
     - 720p → H.264 + VP9
     - 1080p → H.264 + VP9
     - (Опционально: 4K, 8K)
  5. Нарезают на HLS/DASH сегменты (2-10 секунд)
  6. Загружают в Object Storage + создают манифест (.m3u8)
  7. Обновляют метаданные: video.status = 'ready'

Adaptive Bitrate Streaming:
  - HLS/DASH: клиент выбирает разрешение под сеть
  - Манифест содержит ссылки на сегменты всех разрешений
  - Клиент переключается между разрешениями без перезагрузки
```

---

## Чек-лист: System Design Cases

### Twitter-like Feed
- [ ] Fan-out on Write vs Fan-out on Read vs гибрид
- [ ] Как обрабатывать celebrities (>1M followers)?
- [ ] Timeline cache in Redis: структура, TTL, лимиты
- [ ] Медиа: upload pipeline, ресайзы, CDN

### Uber-like Matching
- [ ] Geo-индексация: Geohash / S2 / H3 — сравнение
- [ ] Как обновлять GPS 1.25M раз/сек?
- [ ] Matching алгоритм: поиск ближайших водителей
- [ ] Ride lifecycle: запрос → matching → поездка → оплата

### Messenger
- [ ] WebSocket vs long-polling vs Server-Sent Events
- [ ] Message ordering: как гарантировать порядок в чате?
- [ ] Схема Cassandra: почему chat_id + timeuuid?
- [ ] Online presence: heartbeat + Redis TTL
- [ ] Group chat: fan-out сообщений в группе

### Video Platform
- [ ] Transcoding pipeline: очередь, workers, форматы
- [ ] HLS/DASH: adaptive bitrate streaming
- [ ] CDN: зачем, как работает, cache hit ratio
- [ ] Storage: object storage vs file system

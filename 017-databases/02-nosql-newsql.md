# NoSQL и NewSQL

---

## CAP Theorem (Ericks Brewer)

```
         ┌──────────┐
         │  Consistency │
         └─────┬──────┘
               │
    ┌──────────┼──────────┐
    │          │          │
┌───┴────┐ ┌───┴────┐ ┌───┴────┐
│  CA    │ │  CP    │ │  AP    │
│(RDBMS) │ │(HBase) │ │(Dynamo)│
└────────┘ └────────┘ └────────┘
     │          │          │
     │    ┌─────┴──────┐   │
     │    │Partition   │   │
     └────┤ Tolerance  ├───┘
          └────────────┘
```

**Факт:** В распределённой системе choose 2 из 3. На практике — **CP или AP**, потому что Partition Tolerance обязателен.

### PACELC Theorem

Расширение CAP:
- Если **P**artition → choose C или A
- Если **E**lse (нет partition) → choose L (latency) или C (consistency)

---

## Типы NoSQL

### Document Stores (MongoDB, Couchbase, Firestore)

```json
{
  "_id": "order_123",
  "user_id": "user_42",
  "items": [
    { "product": "laptop", "quantity": 1, "price": 1200 },
    { "product": "mouse", "quantity": 2, "price": 25 }
  ],
  "total": 1250,
  "status": "pending",
  "created_at": "2025-06-27T20:00:00Z"
}
```

**Когда:** Данные естественно иерархичны, схема гибкая, read-heavy.

**Когда нет:** Много cross-document связей, сложные JOIN'ы.

### Key-Value Stores (Redis, DynamoDB, LevelDB)

```
key: "user:42:profile" → value: { JSON }
key: "session:abc123"  → value: { "user_id": 42, "expires": ... }
```

**Когда:** Кэш, сессии, счётчики, простые lookups по ключу.

**Когда нет:** Сложные запросы, relationships.

### Wide-Column Stores (Cassandra, HBase, ScyllaDB)

```
Row Key: user_42
  Column Family: profile
    name: "Ivan"
    email: "ivan@example.com"
  Column Family: orders
    order_1: { total: 1250, status: "shipped" }
    order_2: { total: 300, status: "pending" }
```

**Когда:** Time-series, IoT, огромные объёмы записей, write-heavy.

**Когда нет:** ACID, ad-hoc queries.

### Graph Databases (Neo4j, Amazon Neptune, ArangoDB)

```cypher
// Neo4j Cypher
MATCH (u:User {name: "Ivan"})-[:FRIEND]->(friends)-[:BOUGHT]->(product)
WHERE NOT (u)-[:BOUGHT]->(product)
RETURN product.name, count(*) as frequency
ORDER BY frequency DESC
LIMIT 10
```

**Когда:** Social networks, рекомендации, fraud detection, графовые аналитики.

**Когда нет:** Простые CRUD, bulk operations.

---

## NewSQL

**NewSQL** — попытка объединить ACID реляционных БД с горизонтальной масштабируемостью NoSQL.

| БД | Особенность |
|----|------------|
| **CockroachDB** | PostgreSQL-совместимый, автоматический шардинг, geo-distributed |
| **YugabyteDB** | PostgreSQL-совместимый, шардинг, multi-region |
| **Google Spanner** | TrueTime для глобальной консистентности |
| **TiDB** | MySQL-совместимый, HTAP (hybrid transactional/analytical) |

**Когда:** Нужен SQL + ACID + горизонтальное масштабирование. Не нужен — если хватает одного PostgreSQL.

---

## Consistency Models

### Strong Consistency

После записи все читатели видят новое значение.

### Eventual Consistency

После записи читатели могут видеть старое значение, но со временем увидят новое.

### Read-your-writes

Пользователь видит свои записи сразу, другие пользователи — eventual.

### Пример: DynamoDB

```csharp
// Strongly consistent read
var item = await context.LoadAsync<Order>("pk", "sk",
    new DynamoDBOperationConfig
    {
        ConsistentRead = true  // дороже, медленнее
    });

// Eventually consistent read (default)
var item = await context.LoadAsync<Order>("pk", "sk");
```

---

## CQRS и Event Sourcing

```
┌──────────┐       ┌───────────┐
│  Write   │       │   Read    │
│  Model   │       │   Model   │
│ (SQL DB) │       │ (Read DB) │
└────┬─────┘       └─────┬─────┘
     │                    ▲
     │  Event Bus         │
     └────────────────────┘
```

**Факт:** CQRS часто идёт с Event Sourcing — сохранением событий вместо состояния.

---

## Чек-лист выбора БД

- [ ] ACID критичен? → Relational / NewSQL
- [ ] Данные иерархичны → Document (MongoDB)
- [ ] Огромный write-heavy → Wide-Column (Cassandra)
- [ ] Кэш / сессии → Key-Value (Redis)
- [ ] Графовые связи → Graph (Neo4j)
- [ ] SQL + масштабирование → NewSQL (CockroachDB)
- [ ] CAP: Consistency важнее → CP; Availability важнее → AP

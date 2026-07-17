# CAP Theorem, PACELC и модели консистентности

---

## CAP Theorem (Brewer's Theorem)

### Формулировка

```
В распределённой системе можно обеспечить ТОЛЬКО ДВА свойства из трёх:

  C — Consistency (Согласованность):
      Все узлы видят одинаковые данные в один момент времени.
      Каждое чтение возвращает результат последней записи.

  A — Availability (Доступность):
      Каждый запрос к неотказавшему узлу возвращает ответ (не ошибку).
      Система продолжает работать при отказе узлов.

  P — Partition Tolerance (Устойчивость к разделению):
      Система продолжает работать при потере сетевого соединения
      между узлами (network partition).

Важно: P — НЕ опция, это реальность.
      Network partition СЛУЧАЕТСЯ (коммутатор упал, кабель перерублен).
      Выбор: при partition — С или A?
```

### Визуализация

```
        C (Consistency)
        /\
       /  \
      /    \
     /  CP  \   CA (невозможно в распределённой системе)
    /        \
   /__________\
 A    AP       P (Partition Tolerance)

CP системы (жертвуют Availability при partition):
  - Отказывают в записи, если нет кворума
  - Примеры: Zookeeper, etcd, Consul, HBase
  - Используются для: конфигурация, координация, блокировки

AP системы (жертвуют Consistency при partition):
  - Принимают записи даже при partition
  - Позже разрешают конфликты
  - Примеры: Cassandra, DynamoDB, CouchDB, Riak
  - Используются для: высоконагруженные системы, eventual consistency acceptable
```

### Разбор на примере

```
Система из 2 узлов (N1, N2). Network partition: N1 и N2 не видят друг друга.

Если система CP:
  - Запрос записи на N1: "Не могу связаться с N2, нужен кворум"
  - Ответ: ошибка. Данные консистентны, но запись ОТКЛОНЕНА.
  - Жертвуем A ради C.

Если система AP:
  - Запрос записи на N1: ок, записал (без синхронизации с N2)
  - Запрос чтения на N2: может вернуть старые данные
  - Данные НЕ консистентны, но система ДОСТУПНА.
  - Жертвуем C ради A.
```

---

## PACELC — расширение CAP

```
PACELC: If Partition, then Availability or Consistency; Else Latency or Consistency.

  При Partition (P):
    выбери A или C (как в CAP)
  
  При отсутствии Partition (Else — нормальный режим):
    выбери L (Latency) или C (Consistency)

Типы систем:

  PA/EL: при partition → availability, норма → low latency
         Cassandra, DynamoDB (по умолчанию)

  PA/EC: при partition → availability, норма → consistency
         MongoDB (с write concern majority)

  PC/EL: при partition → consistency, норма → low latency
         VoltDB, H-Store

  PC/EC: при partition → consistency, норма → consistency
         Zookeeper, etcd, PostgreSQL (с synchronous_commit=on)
```

---

## Модели консистентности

### Шкала консистентности (от сильной к слабой)

```
Strong (Linearizability) ←──→ Weak (Eventual)
    │                              │
    ├─ Linearizability             │
    ├─ Sequential Consistency      │
    ├─ Causal Consistency          │
    ├─ Read-Your-Writes            │
    ├─ Monotonic Reads             │
    ├─ Consistent Prefix           │
    └─ Eventual Consistency ───────┘
```

### Linearizability (строгая консистентность)

```
Определение:
  Каждая операция выглядит так, как будто она выполняется
  мгновенно в одной точке между началом и завершением вызова.
  Все операции образуют ГЛОБАЛЬНЫЙ порядок.

Пример:

  W(x, 1)     W(x, 2)
  ──────────────────────────→ время
        R(x)    R(x)
        =2      =2

  После записи x=1 и x=2, ВСЕ чтения видят x=2.
  Как будто есть один экземпляр данных.

Достигается: Consensus (Raft/Paxos), синхронная репликация.

Цена: высокая latency (нужен консенсус), низкая доступность при partition.
```

### Sequential Consistency

```
Определение:
  Операции от одного клиента видны в порядке их вызова.
  Но операции разных клиентов могут быть перемешаны.

Слабее чем Linearizability:
  - Нет глобального real-time порядка
  - Но каждый клиент видит свои операции по порядку

Пример:

  Client 1: W(x,1) ────── W(x,2)
  Client 2:         R(x) ──────── R(x)
  
  Допустимо: R(x)=0, R(x)=1, R(x)=2 (порядок Client 1 соблюдён)
  Допустимо: R(x)=1, R(x)=2
  Не допустимо: R(x)=2, R(x)=1 (нарушен порядок Client 1)
```

### Causal Consistency

```
Определение:
  Если операция B причинно зависит от A, то все видят A перед B.
  Операции без причинной связи (concurrent) — можно в любом порядке.

Пример (соц. сеть):

  Alice: публикует фото заката
  Bob:   комментирует "Красиво!" (причинно зависит от фото Alice)

  Charlie: должен видеть фото ДО комментария (causal order)
  Dave:    может видеть комментарий БЕЗ фото? НЕТ — causal consistency запрещает

Реализация: векторные часы (vector clocks), Lamport timestamps.
```

### Read-Your-Writes (Согласованность чтения своих записей)

```
Определение:
  После записи клиент ВСЕГДА видит свою запись при последующих чтениях.

Проблема без Read-Your-Writes:
  1. Пользователь публикует пост
  2. Запись ушла на master, чтение с replica (ещё не синхронизирована)
  3. Пользователь обновляет страницу — поста НЕТ! (паника)

Решение:
  - Привязка к мастеру после записи (на N секунд)
  - Чтение с кворума (R + W > N)
  - Хранение времени последней записи → чтение с реплики не старше
```

### Monotonic Reads

```
Определение:
  Клиент не видит более старые данные после того, как увидел более новые.

Без Monotonic Reads:
  1. Чтение с реплики 1 (синхронизирована) → видит x=2
  2. Чтение с реплики 2 (отстала) → видит x=1
  Пользователь видит, как данные "откатываются" назад!

Решение:
  - Чтение всегда с одной реплики (sticky session)
  - Чтение по времени: не старше, чем предыдущее
```

### Eventual Consistency

```
Определение:
  Если в систему не поступает новых записей, то в конечном счёте
  все реплики сойдутся к одному значению.

  - Нет гарантий СВЕЖЕСТИ данных
  - Нет гарантий ПОРЯДКА
  - Но РАНО ИЛИ ПОЗДНО все реплики согласуются

Пример: DNS — запись обновляется, но старый IP может кешироваться часами.
```

---

## Консенсус — Raft и Paxos

### Raft — алгоритм консенсуса

```
Состояния узла в Raft:

  ┌──────────┐  timeout, election  ┌──────────┐
  │ Follower │────────────────────→│ Candidate│
  │          │←────────────────────│          │
  └────┬─────┘  discovers leader   └────┬─────┘
       │         or new term             │
       │                                │ receives votes
       │                                │ from majority
       ▼                                ▼
  ┌──────────────────────────────────────────┐
  │                 Leader                    │
  │  (принимает записи, реплицирует лог)      │
  └──────────────────────────────────────────┘

Процесс записи в Raft:
  1. Leader получает запись от клиента
  2. Добавляет в свой лог (uncommitted)
  3. Рассылает AppendEntries всем follower'ам
  4. Ждёт подтверждения от БОЛЬШИНСТВА (quorum)
  5. Коммитит запись (apply to state machine)
  6. Отвечает клиенту

Выборы лидера:
  - Случайные таймауты (150-300ms) → минимум split votes
  - Кандидат голосует за себя, просит голоса у других
  - Побеждает набравший большинство голосов
  - Term (срок): увеличивается при каждых выборах
```

### Где нужен консенсус

```
- Leader election: кто primary БД? Кто active контроллер?
- Distributed locking: только один владеет блокировкой
- Configuration management: согласованная конфигурация кластера
- Atomic broadcast: все узлы видят сообщения в одном порядке
- Replicated State Machine: identical лог → identical состояние
```

### Без консенсуса: Gossip Protocol

```
Gossip (Epidemic) Protocol:

  Не даёт строгой консистентности, но обеспечивает eventual consistency
  без единого лидера.

  Алгоритм:
    Каждые T секунд узел выбирает случайного соседа и обменивается данными.
    Как слухи распространяются в популяции: за O(log N) раундов все узнают.

  Применение:
    - Cassandra (membership, dissemination)
    - Consul (service discovery)
    - Redis Cluster (gossip для обнаружения узлов)
    - DynamoDB (membership + failure detection)
```

---

## Примеры выбора консистентности в реальных системах

| Система | Модель | Почему |
|---------|--------|--------|
| PostgreSQL (single) | Linearizability | Один узел, все видят одно |
| PostgreSQL (sync replication) | Linearizability | Ждём подтверждения standby |
| PostgreSQL (async replication) | Read-Your-Writes (если читать с primary) | Реплика может отставать |
| Zookeeper / etcd | Linearizability (CP) | Лидер + кворум на запись |
| Cassandra (QUORUM) | Sequential / Causal | R+W > N даёт сильную, но не линеаризуемость |
| Cassandra (ONE) | Eventual | Читаем с одной реплики |
| DynamoDB (strongly consistent) | Sequential | Чтение с кворума |
| DynamoDB (eventually consistent) | Eventual | Чтение с одной реплики |
| Kafka (ack=all) | Causal (per partition) | Сообщения от одного producer упорядочены |
| Redis (single) | Linearizability | Однопоточный |
| Redis Cluster | Eventual (master-slave) | Async replication |

---

## Чек-лист: CAP, PACELC, консистентность

- [ ] CAP: что значит каждое свойство? Почему P — не выбор?
- [ ] CP vs AP системы: примеры, trade-offs
- [ ] PACELC: что добавляет к CAP?
- [ ] Модели консистентности: linearizability, sequential, causal, read-your-writes, monotonic reads, eventual
- [ ] Read-Your-Writes: проблема и решения
- [ ] Raft: состояния, процесс записи, выборы лидера
- [ ] Gossip Protocol: как работает, где применяется?
- [ ] Когда какая консистентность нужна на практике?
- [ ] Как обеспечить сильную консистентность в распределённой системе?
- [ ] Что такое consensus и зачем он нужен?

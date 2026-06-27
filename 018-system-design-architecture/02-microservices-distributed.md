# Микросервисы и распределённые системы

---

## Service Decomposition

### Как делить монолит

| Стратегия | Критерий | Пример |
|-----------|----------|--------|
| **Business Capability** | Что делает? | Order Service, Payment Service |
| **Subdomain (DDD)** | По bounded contexts | Inventory BC → Inventory Service |
| **Organizational (Conway's Law)** | По командам | Команда платёжей → Payment Service |

### Conway's Law

> "Organizations design systems that mirror their communication structure."

**Следствие:** Если 3 команды пишут один сервис → получите 3 сервиса в одном. Сделайте 3 отдельных сервиса.

---

## Inter-Service Communication

### Synchronous (HTTP/gRPC)

```
Service A ──HTTP/gRPC──→ Service B
              ↑
         (+) Просто, понятно
         (-) Coupling, cascading failures, latency
```

### Asynchronous (Event/Message)

```
Service A ──Publish──→ Event Bus ──Consume──→ Service B
                       (Kafka/RabbitMQ)
              ↑
         (-) Сложнее в отладке
         (+) Loose coupling, resilience, scalability
```

### gRPC vs REST

| | REST | gRPC |
|---|------|------|
| **Protocol** | HTTP/1.1 | HTTP/2 |
| **Data format** | JSON (text) | Protobuf (binary) |
| **Contract** | OpenAPI/Swagger | .proto file |
| **Streaming** | Server-Sent Events | Bidirectional streaming |
| **Code gen** | Manual | Built-in |

```protobuf
// order.proto
service OrderService {
  rpc GetOrder (GetOrderRequest) returns (Order);
  rpc CreateOrder (stream OrderItem) returns (Order);  // Client streaming
  rpc WatchOrders (WatchRequest) returns (stream Order);  // Server streaming
}

message GetOrderRequest {
  string order_id = 1;
}

message Order {
  string id = 1;
  string user_id = 2;
  repeated OrderItem items = 3;
  OrderStatus status = 4;
}
```

---

## Distributed Transactions

### Проблема

```
Service A ──→ DB A ✓
Service A ──→ Service B ──→ DB B ✗ (fail!)
```

**ACID не работает через сеть.**

### Решения

#### 1. Saga Pattern

```
Choreography (каждый сервис знает, что делать после события)
OrderCreated → PaymentService.Charge → InventoryService.Reserve

Orchestration (центральный оркестратор)
OrderSaga:
  1. CreateOrder
  2. ReserveInventory
  3. ProcessPayment
  4. If fail → Compensate (CancelOrder, ReleaseInventory)
```

```csharp
// Saga Orchestrator with MassTransit
public class OrderSaga : MassTransitStateMachine<OrderSagaState>
{
    public State Creating { get; set; }
    public State Reserving { get; set; }
    public State Charging { get; set; }

    public Event<OrderCreated> OrderCreated { get; set; }
    public Event<InventoryReserved> InventoryReserved { get; set; }
    public Event<PaymentProcessed> PaymentProcessed { get; set; }

    public OrderSaga()
    {
        Initially(
            When(OrderCreated)
                .Then(context => context.Saga.OrderId = context.Message.OrderId)
                .TransitionTo(Creating));

        During(Creating,
            When(InventoryReserved)
                .TransitionTo(Reserving));

        During(Reserving,
            When(PaymentProcessed)
                .TransitionTo(Charging)
                .Finalize());

        // Compensating actions
        DuringAny(
            When(Fault<ProcessPayment>)
                .Then(context => context.Publish(new ReleaseInventory())));
    }
}
```

#### 2. Two-Phase Commit (2PC)

```
Coordinator → Prepare → All OK → Commit
                     ↓
                  Any fail → Abort
```

**Минус:** Блокирующий протокол. Если координатор упал на commit — все заблокированы.

#### 3. Outbox Pattern

```
App → Write to Outbox Table (same DB) → Outbox Processor → Publish to Event Bus
```

```csharp
// 1. Сохраняем в одной транзакции
await using var tx = await db.Database.BeginTransactionAsync();
db.Orders.Add(order);
db.OutboxMessages.Add(new OutboxMessage
{
    Type = "OrderCreated",
    Payload = JsonSerializer.Serialize(order),
    CreatedAt = DateTime.UtcNow
});
await db.SaveChangesAsync();
await tx.CommitAsync();

// 2. Outbox Processor читает и публикует
// (фоновая задача, может быть отдельным сервисом)
```

**Факт:** Outbox pattern — де-факто стандарт для гарантированной доставки событий в микросервисах.

---

## Service Discovery

### Client-side Discovery

```
Service A → Service Registry (Consul/Eureka) → Get Service B IP → Call
```

### Server-side Discovery (Kubernetes)

```
Service A → Service B (ClusterIP) → kube-proxy → Pod
```

Kubernetes делает service discovery автоматически через DNS:

```
my-service.namespace.svc.cluster.local → ClusterIP → Pods
```

---

## API Gateway

```
                    ┌──────────┐
                    │   Auth   │
                    ├──────────┤
Client ──→ Gateway ─┤  Routing │──→ Order Service
                    ├──────────┤──→ Payment Service
                    │  Rate    │──→ User Service
                    │  Limit   │
                    ├──────────┤
                    │  Logging │
                    └──────────┘
```

### Gateway vs BFF (Backend For Frontend)

| | API Gateway | BFF |
|---|-------------|-----|
| **Количество** | Один на систему | Один на клиент (web, mobile, desktop) |
| **Когда** | Единый entry point | Разные клиенты требуют разные данные |

---

## Чек-лист

- [ ] Сервисы разделены по business capabilities
- [ ] Event-driven коммуникация для loose coupling
- [ ] Saga pattern для distributed transactions
- [ ] Outbox pattern для гарантированной доставки
- [ ] Retry + circuit breaker (Polly) для resilience
- [ ] Health checks и readiness probes
- [ ] Service discovery (Kubernetes DNS или Consul)
- [ ] API Gateway для маршрутизации
- [ ] Distributed tracing (OpenTelemetry)

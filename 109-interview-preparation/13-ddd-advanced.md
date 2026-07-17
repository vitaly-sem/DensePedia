# Domain-Driven Design — продвинутый уровень

---

## Bounded Contexts — глубокое погружение

### Как определить границы контекста

```
Признаки границы Bounded Context:

1. Лингвистический разрыв:
   Термин "Product" в разных контекстах:
   - Catalog Context: название, цена, фото, категория
   - Inventory Context: SKU, склад, количество, поставщик
   - Shipping Context: вес, габариты, опасный груз?
   - Billing Context: цена, налог, скидка

   → РАЗНЫЕ модели для разных контекстов!

2. Бизнес-процессы:
   - Процессы с разными stakeholders и целями
   - Разные темпы изменений (Core vs Supporting domains)

3. Организационная структура (Закон Конвея):
   - Каждая команда = свой Bounded Context
   - Interface между командами = Interface между контекстами

4. Данные и транзакционные границы:
   - Где нужна сильная консистентность? → Aggregate
   - Где eventual consistency? → Граница контекста
```

### Отношения между Bounded Contexts

```
1. Partnership:
   Команды тесно сотрудничают, координируют изменения.
   Интерфейс эволюционирует вместе.

2. Shared Kernel:
   Общая часть модели между контекстами.
   ОПАСНО: изменения затрагивают все зависимые контексты.
   Минимизировать: только самые стабильные концепции.

3. Customer/Supplier:
   Supplier (поставщик) — upstream, Customer (потребитель) — downstream.
   Upstream диктует интерфейс, downstream адаптируется.
   Но downstream должен влиять на upstream (планирование).

4. Conformist:
   Downstream полностью принимает модель upstream без изменений.
   Когда upstream слишком мощный/дорогой для влияния.

5. Anti-Corruption Layer (ACL):
   Downstream создаёт изолирующий слой для защиты своей модели.
   Трансляция внешней модели → внутреннюю.

6. Open Host Service:
   Upstream предоставляет публичный API (REST/gRPC).
   Множество downstream интегрируются через него.

7. Published Language:
   Общий формат обмена данными (Protobuf, Avro, JSON Schema).
   Например: OrderPlaced event в Avro (схема-реестр).
```

### ACL — Anti-Corruption Layer (пример)

```csharp
// Внешняя модель (Legacy CRM) — не хотим протаскивать в свой домен
public class LegacyCustomerDto
{
    public string CUST_ID { get; set; }    // "C-00123"
    public string FULL_NAME { get; set; }   // "Иванов Иван"
    public string ADDR_LINE1 { get; set; }
    public string ADDR_LINE2 { get; set; }
    public DateTime LAST_MOD { get; set; }
}

// Наша доменная модель
public class Customer
{
    public CustomerId Id { get; private set; }
    public PersonName Name { get; private set; }
    public Address Address { get; private set; }
}

// ACL — трансляция
public class LegacyCustomerTranslator
{
    public Customer Translate(LegacyCustomerDto dto)
    {
        return new Customer(
            new CustomerId(dto.CUST_ID.Replace("C-", "")),
            PersonName.Parse(dto.FULL_NAME),  // "Иванов Иван" → First/Last
            new Address(dto.ADDR_LINE1, dto.ADDR_LINE2)
        );
    }
}

// Репозиторий использует ACL внутри:
public class CustomerRepository : ICustomerRepository
{
    private readonly LegacyCrmClient _crm;
    private readonly LegacyCustomerTranslator _translator;

    public async Task<Customer> GetById(CustomerId id)
    {
        var dto = await _crm.GetCustomerAsync($"C-{id.Value}");
        return _translator.Translate(dto);
    }
}
```

---

## Aggregate Design — правила и паттерны

### 4 правила Aggregate Design (Vaughn Vernon)

```
1. Защищайте бизнес-инварианты внутри Aggregate:
   Инвариант — правило, которое должно быть ВСЕГДА истинно.
   Пример: Сумма Order = сумма всех OrderLine.Subtotal.
   Aggregate — граница консистентности для этих правил.

2. Проектируйте маленькие Aggregate'ы:
   Идеально: один Entity + несколько Value Objects.
   Большие → contention при конкурентных транзакциях.

3. Ссылайтесь на другие Aggregate'ы по ID (не по ссылке!):
   Order.Lines ссылается на Product через ProductId.
   Order НЕ содержит объект Product целиком.
   
4. Eventual Consistency между Aggregate'ами:
   Изменения в разных агрегатах не обязаны быть атомарными.
   Order → OrderSubmitted событие → Payment получит и обработает.
```

### Пример: границы агрегата

```csharp
// Aggregate Root: Order
public class Order
{
    public OrderId Id { get; private set; }
    public CustomerId CustomerId { get; private set; }  // Ссылка по ID
    public OrderStatus Status { get; private set; }
    public Money Total { get; private set; }
    private List<OrderLine> _lines = new();              // Внутренняя коллекция
    public IReadOnlyList<OrderLine> Lines => _lines;

    // Инварианты:
    // - Order не может быть Submitted без хотя бы одной линии
    // - Total = сумма всех Line.Subtotal
    // - Цена каждой линии не может быть отрицательной

    public void AddItem(ProductId productId, Money price, int quantity)
    {
        if (Status != OrderStatus.Draft)
            throw new DomainException("Нельзя менять не черновик");
        if (quantity <= 0) throw new DomainException("Количество > 0");

        var line = _lines.FirstOrDefault(l => l.ProductId == productId);
        if (line != null)
            line.IncreaseQuantity(quantity);
        else
            _lines.Add(new OrderLine(productId, price, quantity));

        RecalculateTotal();  // Инвариант: Total = сумма
    }

    public void Submit()
    {
        if (!_lines.Any()) throw new DomainException("Пустой заказ");
        if (Total <= Money.Zero) throw new DomainException("Сумма должна быть > 0");

        Status = OrderStatus.Submitted;
        AddDomainEvent(new OrderSubmitted(Id, CustomerId, Total, _lines));
    }

    private void RecalculateTotal()
    {
        Total = _lines.Aggregate(Money.Zero, (sum, line) => sum + line.Subtotal);
    }
}

// OrderLine — часть Aggregate (не Entity глобально, Value Object локально)
public class OrderLine
{
    public ProductId ProductId { get; }  // Ссылка по ID!
    public Money Price { get; private set; }
    public int Quantity { get; private set; }
    public Money Subtotal => Price * Quantity;

    internal OrderLine(ProductId productId, Money price, int quantity)
    {
        ProductId = productId;
        Price = price;
        Quantity = quantity;
    }

    internal void IncreaseQuantity(int delta) => Quantity += delta;
}

// Product — ОТДЕЛЬНЫЙ Aggregate. Order знает только ProductId.
public class Product
{
    public ProductId Id { get; }
    public string Name { get; }
    public Money BasePrice { get; }
    // ...
}
```

---

## Domain Events и Integration Events

### Различие

```
Domain Event:
  - Внутри одного Bounded Context
  - Выражает бизнес-факт в терминах домена
  - Используется для связи между Aggregate'ами внутри контекста
  - Формат: в терминах домена (OrderSubmitted, PaymentConfirmed)

Integration Event:
  - Между Bounded Contexts
  - Публичный контракт (схема, версия)
  - Используется для асинхронной интеграции контекстов
  - Формат: Published Language (Avro/Protobuf/JSON Schema)
```

### Domain Events — реализация

```csharp
// Базовый интерфейс
public interface IDomainEvent
{
    DateTime OccurredAt { get; }
}

// Конкретное событие
public class OrderSubmitted : IDomainEvent
{
    public OrderId OrderId { get; }
    public CustomerId CustomerId { get; }
    public Money Total { get; }
    public DateTime OccurredAt { get; }

    public OrderSubmitted(OrderId orderId, CustomerId customerId, Money total)
    {
        OrderId = orderId;
        CustomerId = customerId;
        Total = total;
        OccurredAt = DateTime.UtcNow;
    }
}

// Aggregate с событиями
public abstract class AggregateRoot
{
    private readonly List<IDomainEvent> _events = new();

    protected void AddDomainEvent(IDomainEvent @event) => _events.Add(@event);
    public IReadOnlyList<IDomainEvent> GetDomainEvents() => _events;
    public void ClearDomainEvents() => _events.Clear();
}

// Диспатчер событий (после сохранения в БД)
public class DomainEventDispatcher
{
    public async Task DispatchAsync(AggregateRoot aggregate)
    {
        foreach (var @event in aggregate.GetDomainEvents())
        {
            // Для каждого типа события — свои handlers
            var handlers = ResolveHandlers(@event.GetType());
            foreach (var handler in handlers)
                await handler.HandleAsync((dynamic)@event);
        }
        aggregate.ClearDomainEvents();
    }
}
```

### Outbox Pattern — надёжная публикация

```sql
-- Таблица Outbox в той же БД, что и Aggregate
CREATE TABLE outbox_messages (
    id BIGSERIAL PRIMARY KEY,
    aggregate_id TEXT NOT NULL,
    aggregate_type TEXT NOT NULL,
    event_type TEXT NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    published_at TIMESTAMPTZ,        -- NULL = не опубликовано
    retry_count INT DEFAULT 0
);

-- Транзакция: сохраняем Aggregate + Outbox атомарно
BEGIN;
  INSERT INTO orders (...) VALUES (...);
  INSERT INTO outbox_messages (...) VALUES (...);  -- OrderSubmitted
COMMIT;

-- Outbox Processor (фоновый процесс):
-- SELECT * FROM outbox_messages WHERE published_at IS NULL
-- Отправляет в Kafka → обновляет published_at
```

---

## Event Sourcing

### Концепция

```
Вместо хранения текущего состояния:
  Текущее состояние = свёртка всех событий.

Обычный подход:
  ┌──────────┐
  │  Order   │  (id=1, status='submitted', total=100)
  └──────────┘

Event Sourcing:
  ┌──────────────────────────────────────────────┐
  │  Event Stream (order-1):                     │
  │                                              │
  │  e1: OrderCreated(id=1, customer=42)         │
  │  e2: ItemAdded(product=101, qty=2, price=30) │
  │  e3: ItemAdded(product=205, qty=1, price=40) │
  │  e4: OrderSubmitted()                        │
  │  e5: Total = 30*2 + 40 = 100                 │
  └──────────────────────────────────────────────┘

  Текущее состояние: применить e1→e5 к начальному состоянию.

+ Полный audit trail
+ Временные запросы (состояние заказа на вчера)
+ Легко строить read-модели (проекции)

- Сложность для разработчиков (непривычно)
- Eventual consistency (проекции обновляются асинхронно)
- Миграция событий (версионирование схемы)
```

### Event Store

```csharp
public interface IEventStore
{
    Task AppendToStream(string streamId, long expectedVersion, IEnumerable<object> events);
    Task<IEnumerable<object>> ReadStream(string streamId, long fromVersion = 0);
}

// Восстановление Aggregate из событий
public async Task<Order> LoadOrder(OrderId id)
{
    var events = await _eventStore.ReadStream($"order-{id.Value}");
    var order = new Order();  // Пустой
    foreach (var @event in events)
        order.Apply(@event);  // Применяем события одно за другим
    return order;
}

// Сохранение: только новые события
public async Task SaveOrder(Order order)
{
    var events = order.GetUncommittedEvents();
    await _eventStore.AppendToStream(
        $"order-{order.Id.Value}",
        order.Version,          // Optimistic concurrency check
        events);
    order.ClearUncommittedEvents();
}
```

---

## CQRS (Command Query Responsibility Segregation)

```
Команды (Commands) — изменяют состояние:
  PlaceOrder, AddItem, CancelOrder
  → Обрабатываются Command Handler'ами
  → Используют доменную модель (Aggregate)
  → Публикуют события

Запросы (Queries) — читают состояние:
  GetOrderById, GetOrdersByCustomer
  → Идут к Read Model (оптимизированные проекции)
  → Не используют доменную модель
  → Могут идти к денормализованным таблицам/кешу

┌──────────┐   Command   ┌──────────┐   Event    ┌──────────┐
│  Client  │────────────→│  Write   │───────────→│ Event    │
│          │             │  Model   │            │ Store    │
│          │←────────────│          │            │ (Kafka)  │
│          │   Query     └──────────┘            └────┬─────┘
│          │                                          │
│          │   Query     ┌──────────┐   Project      │
│          │←────────────│  Read    │←───────────────┘
│          │             │  Model   │
└──────────┘             │ (SQL/    │
                         │  Redis)  │
                         └──────────┘
```

```csharp
// Command
public record PlaceOrderCommand(
    CustomerId CustomerId,
    List<OrderItemDto> Items
) : ICommand<OrderId>;

// Command Handler
public class PlaceOrderHandler : ICommandHandler<PlaceOrderCommand, OrderId>
{
    private readonly IOrderRepository _orders;
    
    public async Task<OrderId> Handle(PlaceOrderCommand cmd)
    {
        var order = Order.Create(cmd.CustomerId);
        foreach (var item in cmd.Items)
            order.AddItem(item.ProductId, item.Price, item.Quantity);
        order.Submit();
        
        await _orders.Save(order);
        return order.Id;
    }
}

// Query
public record GetOrderSummaryQuery(OrderId OrderId) : IQuery<OrderSummaryDto>;

// Query Handler — читает Read Model (не доменную модель!)
public class GetOrderSummaryHandler : IQueryHandler<GetOrderSummaryQuery, OrderSummaryDto>
{
    public Task<OrderSummaryDto> Handle(GetOrderSummaryQuery query)
    {
        return _dbContext.OrderSummaries
            .Where(o => o.OrderId == query.OrderId.Value)
            .Select(o => new OrderSummaryDto(o.OrderId, o.Total, o.Status, o.CreatedAt))
            .FirstOrDefaultAsync();
    }
}
```

---

## Чек-лист: DDD Advanced

- [ ] Bounded Context: критерии определения границ
- [ ] Context Map: Partnership, Shared Kernel, Customer/Supplier, Conformist, ACL, OHS, Published Language
- [ ] ACL: когда нужен, как реализовать?
- [ ] Aggregate Design: 4 правила Vernon'а
- [ ] Почему ссылаться на другие Aggregate'ы по ID, а не по объекту?
- [ ] Domain Events vs Integration Events
- [ ] Outbox Pattern: зачем и как?
- [ ] Event Sourcing: плюсы (audit, temporal queries) и минусы (сложность, eventual consistency)
- [ ] CQRS: разделение Write и Read моделей, когда нужно?
- [ ] Как связать DDD с микросервисами? (Bounded Context → микросервис)
- [ ] Event Storming: этапы и что даёт?

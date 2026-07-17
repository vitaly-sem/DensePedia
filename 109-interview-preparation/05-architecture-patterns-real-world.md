# Архитектурные паттерны — практика для собеседования

---

## SOLID — с реальными примерами на C#

### S — Single Responsibility Principle (Принцип единственной ответственности)

```
Каждый класс должен иметь одну и только одну причину для изменения.
```

```csharp
// ❌ НАРУШЕНИЕ: класс делает всё
public class OrderProcessor
{
    public void Process(Order order)
    {
        Validate(order);           // Валидация
        SaveToDb(order);           // Сохранение в БД
        SendEmail(order.Customer); // Отправка email
        LogToFile(order);          // Логирование
    }
}

// ✅ SRP: разделение ответственности
public class OrderValidator
{
    public void Validate(Order o)
    {
        if (o.Items.Count == 0)
            throw new ValidationException("Order must have items");
        if (o.Customer == null)
            throw new ValidationException("Customer is required");
    }
}

public class OrderRepository
{
    public async Task SaveAsync(Order o)
    {
        await using var conn = new NpgsqlConnection(_connectionString);
        await conn.ExecuteAsync(
            "INSERT INTO orders (...) VALUES (...)", o);
    }
}

public class EmailService
{
    public async Task SendConfirmationAsync(Customer c)
    {
        await _smtp.SendAsync(new MailMessage(
            "noreply@shop.com", c.Email, "Order Confirmed",
            $"Thank you for your order, {c.Name}!"));
    }
}
public class OrderProcessor
{
    private readonly OrderValidator _validator;
    private readonly OrderRepository _repo;
    private readonly EmailService _email;
    // Координирует процесс, но НЕ делает всё сам
    public async Task ProcessAsync(Order o)
    {
        _validator.Validate(o);
        await _repo.SaveAsync(o);
        await _email.SendConfirmationAsync(o.Customer);
    }
}
```

### O — Open/Closed Principle (Открытости/закрытости)

```
Классы открыты для расширения, но закрыты для модификации.
```

```csharp
// ❌ Нарушение OCP: каждый новый тип скидки — изменение класса
public class DiscountCalculator
{
    public decimal Calculate(Order order, string discountType)
    {
        switch (discountType)
        {
            case "percentage": return order.Total * 0.9m;
            case "fixed": return order.Total - 50;
            case "loyalty": return order.Total * 0.95m - 10;
            // Новый тип скидки → модификация класса!
            default: return order.Total;
        }
    }
}

// ✅ OCP: стратегия через интерфейс
public interface IDiscountStrategy
{
    decimal Apply(Order order);
}

public class PercentageDiscount : IDiscountStrategy
{
    public decimal Apply(Order o) => o.Total * 0.9m;
}

public class LoyaltyDiscount : IDiscountStrategy
{
    public decimal Apply(Order o) => o.Total * 0.95m - 10;
}

public class DiscountCalculator
{
    public decimal Calculate(Order order, IDiscountStrategy strategy)
        => strategy.Apply(order);
}

// Новая скидка: новый класс, старый код не трогаем!
public class BlackFridayDiscount : IDiscountStrategy
{
    public decimal Apply(Order o) => o.Total * 0.5m;
}
```

### L — Liskov Substitution Principle (Подстановки Лисков)

```
Подтипы должны быть взаимозаменяемы с базовым типом 
без нарушения корректности программы.
```

```csharp
// ❌ НАРУШЕНИЕ LSP: Квадрат не является Behaviour-подтипом Прямоугольника
public class Rectangle
{
    public virtual int Width { get; set; }
    public virtual int Height { get; set; }
    public int Area() => Width * Height;
}

public class Square : Rectangle
{
    public override int Width
    {
        set { base.Width = base.Height = value; }
    }
    public override int Height
    {
        set { base.Width = base.Height = value; }
    }
}

// Тест нарушает LSP:
void Test(Rectangle r)
{
    r.Width = 5;
    r.Height = 10;
    Assert.Equal(50, r.Area()); // FAILS для Square (Area = 100!)
}

// ✅ LSP: разделение через IShape
public interface IShape { int Area(); }
public class RectangleShape : IShape
{
    public int Width { get; set; }
    public int Height { get; set; }
    public int Area() => Width * Height;
}
public class SquareShape : IShape
{
    public int Side { get; set; }
    public int Area() => Side * Side;
}
```

### I — Interface Segregation Principle (Разделения интерфейса)

```
Не заставляйте клиентов зависеть от методов, которые они не используют.
```

```csharp
// ❌ Нарушение ISP: "толстый" интерфейс
public interface IRepository<T>
{
    Task<T> GetById(int id);
    Task<IEnumerable<T>> GetAll();
    Task Add(T entity);
    Task Update(T entity);
    Task Delete(int id);
    Task<IEnumerable<T>> Search(Expression<Func<T, bool>> predicate);
    Task BulkInsert(IEnumerable<T> entities);
    Task ExportToCsv(Stream stream); // WTF?!
}

// ✅ ISP: разделение на ролевые интерфейсы
public interface IReadRepository<T>
{
    Task<T> GetById(int id);
    Task<IEnumerable<T>> GetAll();
    Task<IEnumerable<T>> Search(Expression<Func<T, bool>> predicate);
}

public interface IWriteRepository<T>
{
    Task Add(T entity);
    Task Update(T entity);
    Task Delete(int id);
    Task BulkInsert(IEnumerable<T> entities);
}

// Read-only сервис зависит только от чтения:
public class ReportService
{
    private readonly IReadRepository<Order> _orders; // Не зависит от Write!
}
```

### D — Dependency Inversion Principle (Инверсии зависимостей)

```
Высокоуровневые модули не должны зависеть от низкоуровневых.
Оба должны зависеть от абстракций.
```

```csharp
// ❌ Нарушение DIP: высокоуровневый модуль зависит от низкоуровневого
public class OrderService
{
    private readonly SqlOrderRepository _repo = new SqlOrderRepository();
    // Жёсткая зависимость от SQL! Нельзя подменить для тестов.
}

// ✅ DIP: зависит от абстракции
public class OrderService
{
    private readonly IOrderRepository _repo;
    public OrderService(IOrderRepository repo) => _repo = repo;
    // Теперь можно подставить: SqlOrderRepository, InMemoryRepo, Mock
}
```

---

## Микросервисная архитектура

### Декомпозиция

```
Стратегии декомпозиции монолита:

1. По бизнес-возможностям (Business Capability):
   - Orders, Payments, Shipping, Inventory, Users

2. По доменам (Domain-Driven Design):
   - Bounded Context = микросервис
   - Order Context, Payment Context, etc.

3. По техническим границам:
   - Command (write) vs Query (read) → CQRS
   - CPU-intensive vs I/O-intensive сервисы
```

### Коммуникация между микросервисами

```
┌─────────────────────────────────────────────────┐
│              СИНХРОННАЯ (REST/gRPC)             │
├─────────────────────────────────────────────────┤
│ + Простая отладка                               │
│ + Консистентность (strong consistency)          │
│ - Каскадные отказы (если один упал — все стоят) │
│ - Высокая latency (сумма всех вызовов)          │
│ - Tight coupling по времени                     │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│               АСИНХРОННАЯ (Kafka/RabbitMQ)      │
├─────────────────────────────────────────────────┤
│ + Отказоустойчивость (eventual consistency)     │
│ + Слабая связанность                            │
│ + Высокая throughput                            │
│ - Сложная отладка и tracing                     │
│ - Eventual consistency — нужно проектировать    │
│ - Дубликаты и потеря сообщений                  │
└─────────────────────────────────────────────────┘
```

### Saga Pattern (распределённые транзакции)

```
Проблема: как обновить данные в нескольких сервисах атомарно?

Решение: Saga = последовательность локальных транзакций + компенсация.

Типы Sag:

1. Choreography (хореография):
   Order → (event: OrderCreated) → Payment → (event: PaymentDone) → Shipping
   
   + Простая реализация (только события)
   - Сложно отследить общий статус
   - Циклические зависимости

2. Orchestration (оркестрация):
   OrderSaga (оркестратор):
     Step 1: CreateOrder() → success
     Step 2: ProcessPayment() → success  
     Step 3: StartShipping() → FAIL
     Step 4: RefundPayment() (компенсация)
     Step 5: CancelOrder() (компенсация)
   
   + Централизованный контроль
   + Явный workflow
   - Оркестратор — единая точка отказа (но можно реплицировать)
```

### Circuit Breaker

```
Состояния Circuit Breaker:

     ┌─────────┐   failures > threshold   ┌──────────┐
     │ CLOSED  │ ─────────────────────────→│  OPEN    │
     │ (норма) │                           │ (блок)   │
     └────┬────┘                           └────┬─────┘
          │       timeout elapsed               │
          │  ┌──────────────────────────────────┘
          │  │
     ┌────▼──┴───┐
     │ HALF-OPEN │   success → CLOSED
     │ (проверка)│   failure → OPEN
     └───────────┘

В .NET: Polly или Microsoft.Extensions.Http.Resilience
```

```csharp
// Circuit Breaker с Polly
services.AddHttpClient("payments")
    .AddResilienceHandler("pipeline", builder =>
    {
        builder.AddCircuitBreaker(new()
        {
            FailureRatio = 0.5,        // 50% ошибок
            MinimumThroughput = 10,    // Минимум 10 запросов для анализа
            BreakDuration = TimeSpan.FromSeconds(30), // Время в OPEN
            SamplingDuration = TimeSpan.FromSeconds(30) // Окно сбора статистики
        });
    });
```

---

## Domain-Driven Design (DDD)

### Стратегический дизайн

```
Ключевые концепции:

BOUNDED CONTEXT:
  - Чёткая граница доменной модели
  - Внутри контекста — единая терминология (Ubiquitous Language)
  - Например: "Order" в Order Context ≠ "Order" в Shipping Context

CONTEXT MAP:
  - Отношения между Bounded Contexts:
    • Partnership (партнёрство)
    • Shared Kernel (общее ядро)
    • Customer/Supplier (клиент/поставщик)
    • Conformist (конформист)
    • Anti-Corruption Layer (ACL — защитный слой)
    • Open Host Service (публичный API)
    • Published Language (общий формат, например Avro/Protobuf)
```

### Тактический дизайн — строительные блоки

| Блок | Описание | Пример |
|------|----------|--------|
| **Entity** | Объект с идентичностью (ID не меняется) | Order, Customer |
| **Value Object** | Неизменяемый, равен по значению, без ID | Money, Address, Email |
| **Aggregate** | Кластер Entity + Value Objects, граница консистентности | Order (корень) + OrderLine[] |
| **Repository** | Доступ к Aggregate (сохранение, загрузка) | IOrderRepository |
| **Domain Service** | Бизнес-логика, не принадлежащая конкретному Entity | PricingService |
| **Application Service** | Координация use case (тонкий слой) | PlaceOrderHandler |
| **Domain Event** | Факт в домене, важный для других контекстов | OrderPlaced, PaymentReceived |
| **Specification** | Бизнес-правило как предикат | IsEligibleForDiscountSpec |

### Пример: Aggregate Design

```csharp
// Order — Aggregate Root
// Гарантирует инварианты в своих границах
public class Order : IAggregateRoot
{
    public OrderId Id { get; private set; }
    public CustomerId CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public Money Total { get; private set; }
    private readonly List<OrderLine> _lines = new();
    public IReadOnlyList<OrderLine> Lines => _lines;

    // Фабричный метод — создание заказа
    public static Order Create(CustomerId customerId)
    {
        var order = new Order
        {
            Id = OrderId.New(),
            CustomerId = customerId,
            Status = OrderStatus.Draft,
            Total = Money.Zero
        };
        order.AddDomainEvent(new OrderCreated(order.Id, customerId));
        return order;
    }

    // Метод домена — инкапсуляция бизнес-логики
    public void AddItem(ProductId productId, Money price, int quantity)
    {
        if (Status != OrderStatus.Draft)
            throw new DomainException("Cannot modify non-draft order");

        var line = new OrderLine(OrderLineId.New(), productId, price, quantity);
        _lines.Add(line);
        Total = Total + line.Subtotal;
    }

    public void Submit()
    {
        if (!_lines.Any())
            throw new DomainException("Order must have at least one item");
        if (Total <= Money.Zero)
            throw new DomainException("Order total must be positive");

        Status = OrderStatus.Submitted;
        AddDomainEvent(new OrderSubmitted(Id, CustomerId, Total));
    }
}

// Value Object — Money (неизменяемый)
public record Money(decimal Amount, string Currency)
{
    public static Money Zero => new(0, "USD");

    public static Money operator +(Money a, Money b)
    {
        if (a.Currency != b.Currency)
            throw new DomainException("Cannot add different currencies");
        return new Money(a.Amount + b.Amount, a.Currency);
    }
}
```

### Event Storming — методология discovery

```
Этапы Event Storming:

1. Chaotic Exploration:
   - Каждый наклеивает Domain Events на стену
   - Оранжевые стикеры = события в прошлом времени
   - Например: "Order Placed", "Payment Received"

2. Timeline:
   - Упорядочиваем события во времени
   - Убираем дубликаты

3. Pain Points & Opportunities:
   - Красные стикеры = проблемы/бутылочные горлышки
   - Зелёные = возможности/идеи

4. Key Actors & Commands:
   - Синие стикеры = команды (глаголы)
   - Жёлтые = акторы/роли

5. Aggregates & Bounded Contexts:
   - Группируем события в Aggregates
   - Определяем границы Bounded Contexts

6. Policies & Read Models:
   - Фиолетовые = политики ("Когда X происходит, делаем Y")
   - Зелёные = read models (проекции данных)
```

---

## Чек-лист: архитектурные вопросы на собеседовании

### SOLID
- [ ] Объяснить каждый принцип с примером нарушения и исправления
- [ ] LSP: почему квадрат не наследуется от прямоугольника?
- [ ] DIP vs DI: в чём разница?

### Микросервисы
- [ ] Когда НЕ нужно использовать микросервисы? (Ответ: стартап, один домен, маленькая команда)
- [ ] Как обеспечить консистентность данных между сервисами?
- [ ] Saga: choreography vs orchestration — плюсы/минусы
- [ ] Circuit Breaker, Retry, Bulkhead — как работают?
- [ ] Как дебажить распределённую систему? (Distributed tracing: OpenTelemetry)
- [ ] Service Discovery, API Gateway, Service Mesh

### DDD
- [ ] Что такое Bounded Context и как его определить?
- [ ] Отличие Entity от Value Object
- [ ] Aggregate: как определить правильные границы?
- [ ] Domain Event vs Integration Event
- [ ] CQRS: когда применять и цена сложности
- [ ] Event Sourcing: плюсы (audit trail, temporal queries) и минусы (сложность, eventual consistency)

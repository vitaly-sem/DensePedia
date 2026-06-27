# Enterprise-шаблоны .NET

---

## Repository

**Факт:** Абстракция над хранилищем данных. Скрывает детали доступа, упрощает тестирование.

```csharp
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(Guid id);
    Task<IEnumerable<Order>> GetByUserAsync(Guid userId);
    Task<Order> AddAsync(Order order);
    Task UpdateAsync(Order order);
    Task DeleteAsync(Guid id);
    IQueryable<Order> Query();  // Для сложных запросов
}

public class OrderRepository : IOrderRepository
{
    private readonly AppDbContext _db;
    
    public OrderRepository(AppDbContext db) => _db = db;
    
    public async Task<Order?> GetByIdAsync(Guid id)
        => await _db.Orders
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.Id == id);
    
    public IQueryable<Order> Query() => _db.Orders.AsQueryable();
}
```

**Факт:** Generic Repository — антипаттерн (нарушает ISP, скрывает важные детали). Специфичные репозитории лучше.

---

## Unit of Work

**Факт:** Объединяет несколько операций в одну транзакцию.

```csharp
public interface IUnitOfWork
{
    IOrderRepository Orders { get; }
    IUserRepository Users { get; }
    IProductRepository Products { get; }
    
    Task<int> SaveChangesAsync();
    Task BeginTransactionAsync();
    Task CommitAsync();
    Task RollbackAsync();
}

// EF Core DbContext — встроенный Unit of Work
public class AppDbContext : DbContext
{
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<User> Users => Set<User>();
    
    public override async Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        // Audit logging, validation, etc.
        return await base.SaveChangesAsync(ct);
    }
}

// Использование
public class PlaceOrderService
{
    private readonly IUnitOfWork _uow;
    
    public async Task<Guid> PlaceOrderAsync(CreateOrderRequest request)
    {
        await _uow.BeginTransactionAsync();
        try
        {
            var order = new Order(request.UserId);
            await _uow.Orders.AddAsync(order);
            await _uow.SaveChangesAsync();
            
            await _uow.CommitAsync();
            return order.Id;
        }
        catch
        {
            await _uow.RollbackAsync();
            throw;
        }
    }
}
```

---

## CQRS (Command Query Responsibility Segregation)

**Факт:** Разделение чтения и записи на разные модели.

```
CreateOrderCommand → CommandHandler → Write DB
GetOrderQuery     → QueryHandler  → Read DB (может быть denormalized)
```

```csharp
// Command — изменение данных
public record CreateOrderCommand(
    Guid UserId, string ShippingAddress, List<OrderItemDto> Items) : IRequest<Guid>;

public class CreateOrderHandler : IRequestHandler<CreateOrderCommand, Guid>
{
    private readonly IOrderRepository _orders;
    
    public async Task<Guid> Handle(CreateOrderCommand request, CancellationToken ct)
    {
        var order = new Order(request.UserId);
        // ... бизнес-логика
        await _orders.AddAsync(order);
        return order.Id;
    }
}

// Query — чтение данных
public record GetOrderQuery(Guid OrderId) : IRequest<OrderDto>;

public class GetOrderHandler : IRequestHandler<GetOrderQuery, OrderDto>
{
    // Query handler может читать из отдельной read-модели
    private readonly IDbConnection _db;
    
    public async Task<OrderDto> Handle(GetOrderQuery request, CancellationToken ct)
        => await _db.QueryFirstAsync<OrderDto>(
            "SELECT Id, Total, Status FROM OrderReadModel WHERE Id = @Id",
            new { request.OrderId });
}
```

---

## Outbox Pattern

**Факт:** Гарантирует **at-least-once delivery** событий через БД (предотвращает потерю событий при сбое).

```csharp
// Проблема: сохранение + отправка события — не атомарны
// Решение: Outbox — сохраняем событие в той же транзакции

public class OrderService
{
    public async Task<Order> CreateAsync(CreateOrderRequest request)
    {
        await using var transaction = await _db.BeginTransactionAsync();
        
        // 1. Основная операция
        var order = new Order(request.UserId);
        _db.Orders.Add(order);
        
        // 2. Событие в той же транзакции
        _db.OutboxMessages.Add(new OutboxMessage
        {
            Id = Guid.NewGuid(),
            Type = "OrderCreated",
            Payload = JsonSerializer.Serialize(new OrderCreatedEvent(order.Id)),
            CreatedAt = DateTime.UtcNow
        });
        
        await _db.SaveChangesAsync();
        await transaction.CommitAsync();  // Атомарно
        
        // 3. Фоновый процесс читает Outbox → публикует события
        return order;
    }
}

// Outbox Processor — фоновый worker
public class OutboxProcessor : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            var messages = await _db.OutboxMessages
                .Where(m => !m.ProcessedAt.HasValue)
                .OrderBy(m => m.CreatedAt)
                .Take(100)
                .ToListAsync(ct);
            
            foreach (var msg in messages)
            {
                await _eventBus.PublishAsync(msg.Type, msg.Payload, ct);
                msg.ProcessedAt = DateTime.UtcNow;
            }
            
            await _db.SaveChangesAsync(ct);
            await Task.Delay(1000, ct);
        }
    }
}
```

---

## Saga (Choreography vs Orchestration)

**Факт:** Saga управляет распределённой транзакцией через компенсирующие действия.

```csharp
// Orchestration Saga — центральный координатор
public class OrderSagaOrchestrator
{
    public async Task ExecuteAsync(PlaceOrder request)
    {
        var sagaId = Guid.NewGuid();
        
        try
        {
            // Шаг 1: Reserve Inventory
            await _inventoryClient.ReserveAsync(sagaId, request.Items);
            
            // Шаг 2: Process Payment
            await _paymentClient.ChargeAsync(sagaId, request.Total);
            
            // Шаг 3: Create Shipping
            await _shippingClient.ScheduleAsync(sagaId, request.Address);
            
            // Шаг 4: Confirm Order
            await _orderClient.ConfirmAsync(sagaId);
        }
        catch (Exception)
        {
            // Компенсирующие действия в обратном порядке
            await _shippingClient.CancelAsync(sagaId);
            await _paymentClient.RefundAsync(sagaId);
            await _inventoryClient.ReleaseAsync(sagaId);
        }
    }
}
```

---

## Specification

```csharp
public interface ISpecification<T>
{
    Expression<Func<T, bool>> ToExpression();
    bool IsSatisfiedBy(T entity);
}

public class ActiveUserSpecification : ISpecification<User>
{
    public Expression<Func<User, bool>> ToExpression() 
        => u => u.IsActive && !u.IsDeleted;
    
    public bool IsSatisfiedBy(User user) 
        => user.IsActive && !u.IsDeleted;
}

public class PremiumUserSpecification : ISpecification<User>
{
    private const decimal Threshold = 1000;
    
    public Expression<Func<User, bool>> ToExpression()
        => u => u.TotalSpent >= Threshold;
    
    public bool IsSatisfiedBy(User user)
        => user.TotalSpent >= Threshold;
}

// Композиция спецификаций
public class AndSpecification<T> : ISpecification<T>
{
    private readonly ISpecification<T> _left, _right;
    
    public Expression<Func<T, bool>> ToExpression()
    {
        var left = _left.ToExpression();
        var right = _right.ToExpression();
        var param = Expression.Parameter(typeof(T));
        var body = Expression.AndAlso(
            Expression.Invoke(left, param),
            Expression.Invoke(right, param));
        return Expression.Lambda<Func<T, bool>>(body, param);
    }
}

// Использование
var spec = new ActiveUserSpecification().And(new PremiumUserSpecification());
var users = await _db.Users.Where(spec.ToExpression()).ToListAsync();
```

---

## Чек-лист

- [ ] Repository: абстракция доступа к данным (не generic!)
- [ ] Unit of Work: транзакции, SaveChanges
- [ ] CQRS: разделение Command и Query моделей
- [ ] Outbox Pattern: атомарность через БД
- [ ] Saga: Orchestration vs Choreography, компенсация
- [ ] Specification: композиция бизнес-правил

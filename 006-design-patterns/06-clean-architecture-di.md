# Чистая архитектура и DI

---

## Clean Architecture

**Факты:**
- **Hexagonal Architecture** (Ports & Adapters) — core независим от внешнего мира
- **Onion Architecture** — слои от core наружу
- **Clean Architecture** (Uncle Bob) — dependency rule: зависимости направлены **внутрь**

```
┌─────────────────────────────────────────┐
│           Infrastructure                │  Внешний слой
│   (DB, Queue, Http, File System)       │
│ ┌───────────────────────────────────┐   │
│ │        Application                │   │
│ │   (Use Cases, CQRS, Services)    │   │
│ │ ┌─────────────────────────────┐   │   │
│ │ │        Domain               │   │   │  Внутренний слой
│ │ │ (Entities, Value Objects,  │   │   │
│ │ │  Domain Events, Interfaces)│   │   │
│ │ └─────────────────────────────┘   │   │
│ └───────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

### Domain Layer — Core

```csharp
// Domain — никаких зависимостей от внешнего мира
// Только бизнес-логика

public class Order : IAggregateRoot
{
    public Guid Id { get; private set; }
    public OrderState State { get; private set; }
    private readonly List<OrderItem> _items = new();
    public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();
    
    public void AddItem(Product product, int quantity)
    {
        if (State != OrderState.Pending)
            throw new DomainException("Can't modify non-pending order");
        
        _items.Add(new OrderItem(product.Id, product.Price, quantity));
    }
    
    public Money CalculateTotal() 
        => new Money(_items.Sum(i => i.Total.Amount));
}
```

### Application Layer — Use Cases

```csharp
// Application — orchestrates flow, no infrastructure
public class PlaceOrderHandler : IRequestHandler<PlaceOrderCommand, Guid>
{
    private readonly IOrderRepository _orders;  // Domain interface
    private readonly IUnitOfWork _uow;
    private readonly IEventBus _eventBus;       // Domain interface
    
    public async Task<Guid> Handle(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var order = new Order(cmd.UserId);
        foreach (var item in cmd.Items)
            order.AddItem(item.Product, item.Quantity);
        
        await _orders.AddAsync(order);
        await _uow.SaveChangesAsync(ct);
        
        await _eventBus.PublishAsync(new OrderPlacedEvent(order.Id));
        return order.Id;
    }
}
```

### Infrastructure Layer — Implementations

```csharp
// Infrastructure — реализует domain interfaces
public class EfOrderRepository : IOrderRepository
{
    private readonly AppDbContext _db;
    
    public EfOrderRepository(AppDbContext db) => _db = db;
    
    public async Task<Order?> GetByIdAsync(Guid id)
        => await _db.Orders
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.Id == id);
}
```

---

## Dependency Injection — Lifetimes

| Lifetime | Instance | Когда | Примеры |
|---|---|---|---|
| **Singleton** | Один на всё приложение | Shared state, cache, config | `IMemoryCache`, `ILogger<T>` |
| **Scoped** | Один на HTTP request | EF Core DbContext, Unit of Work | `AppDbContext`, `IUnitOfWork` |
| **Transient** | Новый каждый раз | Lightweight, stateless | `IMediator`, `IRequestHandler<>` |

```csharp
// Правила выбора lifetime:
// 1. Если класс stateless — Transient
// 2. Если класс state на request — Scoped
// 3. Если класс global state — Singleton
// 4. Captive dependency: Singleton → Scoped — ошибка!
// 5. Scoped → Singleton — безопасно
// 6. Transient → Singleton — безопасно

// ❌ Captive Dependency — Singleton зависит от Scoped
builder.Services.AddScoped<IUserService, UserService>();
builder.Services.AddSingleton<OrderService>();  // OrderService зависит от IUserService
// UserService будет закеширован в Singleton — одинаковая инстанция для всех запросов!

// ✅ Решение: сделать OrderService Scoped или использовать IServiceProvider
```

**Факт:** В ASP.NET Core **ValidateScopes** — проверяет captive dependencies в production:

```csharp
builder.Host.UseDefaultServiceProvider(options =>
{
    options.ValidateScopes = true;
    options.ValidateOnBuild = true;
});
```

---

## Composition Root

**Факт:** Composition Root — **единственное место** в приложении, где собирается граф зависимостей.

```csharp
// Program.cs — Composition Root
var builder = WebApplication.CreateBuilder(args);

// Infrastructure
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

builder.Services.AddScoped<IOrderRepository, EfOrderRepository>();
builder.Services.AddScoped<IUnitOfWork>(sp => sp.GetRequiredService<AppDbContext>());

// Application
builder.Services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(typeof(PlaceOrderHandler).Assembly));

// External services
builder.Services.AddHttpClient<IPaymentGateway, StripePaymentGateway>(client =>
{
    client.BaseAddress = new Uri(builder.Configuration["Stripe:ApiUrl"]);
});

// Health checks, monitoring
builder.Services.AddHealthChecks()
    .AddDbContextCheck<AppDbContext>();
```

**Факт:** Scrutor — assembly scanning для DI:

```csharp
// Scrutor — массовая регистрация
builder.Services.Scan(scan => scan
    .FromAssemblies(typeof(PlaceOrderHandler).Assembly)
    .AddClasses(classes => classes.AssignableTo(typeof(IRequestHandler<,>)))
    .AsImplementedInterfaces()
    .WithScopedLifetime());
```

---

## Антипаттерны DI

```csharp
// ❌ Service Locator
public class OrderService
{
    public async Task ProcessAsync(Guid orderId)
    {
        var repo = ServiceLocator.Get<IOrderRepository>();  // Скрытая зависимость
    }
}

// ❌ Property Injection (для опциональных зависимостей)
public class OrderService
{
    public ILogger? Logger { get; set; }  // Nullable — легко забыть
}

// ❌ Ambient Context
public static class CurrentUser
{
    public static AsyncLocal<Guid> UserId = new();  // Глобальное состояние
}

// ✅ Constructor Injection
public class OrderService
{
    public OrderService(
        IOrderRepository repo,        // Обязательная
        ILogger<OrderService>? logger = null)  // Опциональная с default
    { }
}
```

---

## Чек-лист

- [ ] Clean Architecture: Domain (core) → Application → Infrastructure
- [ ] Порядок зависимостей: интерфейсы в Domain, реализация в Infrastructure
- [ ] DI Lifetimes: Singleton, Scoped, Transient
- [ ] Captive Dependency: Singleton → Scoped — ошибка
- [ ] Composition Root: единственное место регистрации
- [ ] Scrutor: assembly scanning
- [ ] Антипаттерны: Service Locator, Property Injection

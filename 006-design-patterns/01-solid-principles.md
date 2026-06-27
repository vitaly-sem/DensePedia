# SOLID и базовые принципы

---

## SOLID

### SRP (Single Responsibility Principle)

**Факт:** Класс должен иметь **одну причину для изменения**. Не «класс делает одно», а «класс отвечает одному актору».

```csharp
// ❌ Нарушение: UserService и сохраняет, и отправляет email, и логирует
public class UserService
{
    public void CreateUser(string name, string email)
    {
        Validate(name, email);
        SaveToDb(name, email);
        SendWelcomeEmail(email);
        LogAudit("User created");
    }
}

// ✅ Соблюдение: разделение ответственности
public class UserService
{
    private readonly IUserRepository _repo;
    private readonly IEmailService _email;
    private readonly IAuditLogger _audit;
    
    public async Task<User> CreateAsync(string name, string email)
    {
        var user = new User(name, email);
        await _repo.AddAsync(user);
        await _email.SendWelcomeAsync(email);
        await _audit.LogAsync("User created", user.Id);
        return user;
    }
}
```

### OCP (Open-Closed Principle)

**Факт:** Класс открыт для **расширения**, закрыт для **изменения**. Используйте наследование, композицию, Strategy pattern.

```csharp
// ❌ Нарушение: switch для каждого нового типа платежа
public class PaymentProcessor
{
    public void Process(Order order, string paymentType)
    {
        switch (paymentType)
        {
            case "CreditCard": /* ... */ break;
            case "PayPal": /* ... */ break;
        }
    }
}

// ✅ Соблюдение: Strategy pattern + DI
public interface IPaymentMethod
{
    string Type { get; }
    Task<PaymentResult> ProcessAsync(Order order);
}

public class PaymentProcessor
{
    private readonly IEnumerable<IPaymentMethod> _methods;
    
    public PaymentProcessor(IEnumerable<IPaymentMethod> methods)
        => _methods = methods;
    
    public async Task<PaymentResult> ProcessAsync(Order order, string type)
    {
        var method = _methods.FirstOrDefault(m => m.Type == type)
            ?? throw new NotSupportedException($"Payment {type} not supported");
        return await method.ProcessAsync(order);
    }
}
```

### LSP (Liskov Substitution Principle)

**Факт:** Подтипы должны быть **заменяемы** базовым типом без изменения корректности программы.

```csharp
// ❌ Нарушение: Rectangle → Square меняет поведение
public class Rectangle
{
    public virtual int Width { get; set; }
    public virtual int Height { get; set; }
    public int Area => Width * Height;
}

public class Square : Rectangle
{
    public override int Width { set => base.Width = base.Height = value; }
    public override int Height { set => base.Width = base.Height = value; }
}

// Клиент ожидает Rectangle
void SetSize(Rectangle r) { r.Width = 5; r.Height = 4; Console.WriteLine(r.Area); }
// Для Rectangle: 20, для Square: 16 ❌

// ✅ Решение: не наследовать, использовать общий интерфейс
public interface IShape { int Area { get; }
```

### ISP (Interface Segregation Principle)

**Факт:** Интерфейсы должны быть **специфичными**, не заставляйте клиента зависеть от методов, которые он не использует.

```csharp
// ❌ Нарушение: большой интерфейс
public interface IWorker
{
    void Work();
    void Eat();
    void Sleep();
}

public class Robot : IWorker
{
    public void Work() { /* OK */ }
    public void Eat() => throw new NotSupportedException();
    public void Sleep() => throw new NotSupportedException();
}

// ✅ Segregation
public interface IWorkable { void Work(); }
public interface IFeedable { void Eat(); }
public interface IRestable { void Sleep(); }

public class Human : IWorkable, IFeedable, IRestable { /* ... */ }
public class Robot : IWorkable { /* ... */ }
```

### DIP (Dependency Inversion Principle)

**Факт:** Модули верхнего уровня **не должны** зависеть от модулей нижнего уровня. Оба должны зависеть от **абстракций**.

```csharp
// ❌ Нарушение: верхний уровень зависит от конкретной реализации
public class OrderService
{
    private readonly SqlServerRepository _repo = new();  // Конкретный класс
    // Теперь OrderService нельзя протестировать без БД
}

// ✅ Соблюдение: DI через интерфейс
public class OrderService
{
    private readonly IOrderRepository _repo;
    
    public OrderService(IOrderRepository repo) => _repo = repo;
}
```

---

## Дополнительные принципы

### DRY (Don't Repeat Yourself)

```csharp
// ❌ Дублирование
public void SaveCustomer(Customer c) { /* validate + save */ }
public void UpdateCustomer(Customer c) { /* validate + update */ }

// ✅ DRY: единый метод с параметром
public void SaveCustomer(Customer c, bool isNew) { /* validate + save/update */ }
```

### YAGNI (You Ain't Gonna Need It)

```csharp
// ❌ Over-engineering: "на всякий случай"
public class FlexibleReportGenerator
{
    private readonly IReportRenderer _renderer;
    private readonly IDataSource _dataSource;
    private readonly IDataTransformer _transformer;
    private readonly ICacheProvider _cache;
    // 10 интерфейсов для того, чтобы просто вывести таблицу
}

// ✅ Минимально
public class ReportGenerator
{
    public Task<byte[]> GenerateAsync(ReportRequest request) { /* ... */ }
}
```

### Law of Demeter (Principle of Least Knowledge)

```csharp
// ❌ Нарушение: «поезд» вызовов
var city = customer.Address.City.Name;  // Customer → Address → City → Name

// ✅ Tell, Don't Ask
var city = customer.GetCityName();  // Customer делегирует
```

### KISS (Keep It Simple, Stupid)

```csharp
// ❌ Overcomplicated
var result = items.Select(x => x.Value)
    .Where(v => v != null)
    .Aggregate(new { Sum = 0, Count = 0 },
        (acc, v) => new { Sum = acc.Sum + v.Value, Count = acc.Count + 1 },
        acc => acc.Count > 0 ? acc.Sum / acc.Count : 0);

// ✅ Simple
var values = items.Select(x => x.Value).Where(v => v != null).ToList();
var average = values.Count > 0 ? values.Average(v => v.Value) : 0;
```

---

## Чек-лист

- [ ] SRP: одна причина для изменения, разделение акторов
- [ ] OCP: расширение без изменения (Strategy, Template Method)
- [ ] LSP: подтипы заменяемы, не ломают контракт
- [ ] ISP: специфичные интерфейсы, без throw NotSupported
- [ ] DIP: абстракции, не конкретные классы
- [ ] DRY vs YAGNI: баланс
- [ ] Law of Demeter: Tell Don't Ask
- [ ] KISS: простота > сложность

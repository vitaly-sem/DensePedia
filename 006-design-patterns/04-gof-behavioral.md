# GoF — Поведенческие шаблоны (Behavioral)

---

## Strategy

**Факт:** Определяет семейство алгоритмов, инкапсулирует их и делает взаимозаменяемыми.

```csharp
public interface ITaxStrategy
{
    decimal CalculateTax(decimal amount);
}

public class UsaTaxStrategy : ITaxStrategy
{
    public decimal CalculateTax(decimal amount) => amount * 0.07m;  // 7%
}

public class EuTaxStrategy : ITaxStrategy
{
    public decimal CalculateTax(decimal amount) => amount * 0.20m;  // 20%
}

public class NoTaxStrategy : ITaxStrategy
{
    public decimal CalculateTax(decimal amount) => 0;
}

public class OrderCalculator
{
    private readonly ITaxStrategy _taxStrategy;
    
    public OrderCalculator(ITaxStrategy taxStrategy) => _taxStrategy = taxStrategy;
    
    public decimal CalculateTotal(decimal subtotal)
        => subtotal + _taxStrategy.CalculateTax(subtotal);
}

// DI выбирает стратегию
builder.Services.AddScoped<ITaxStrategy, EuTaxStrategy>();
```

---

## Observer

**Факт:** Уведомление зависимых объектов об изменениях. В C# — события `event` + `IObservable<T>`/`IObserver<T>`.

```csharp
// Reactive Extensions (Rx.NET) — реализация Observer
public class PriceService : IObservable<PriceUpdate>
{
    private readonly List<IObserver<PriceUpdate>> _observers = new();
    
    public IDisposable Subscribe(IObserver<PriceUpdate> observer)
    {
        _observers.Add(observer);
        return new Unsubscriber(_observers, observer);
    }
    
    public void UpdatePrice(string symbol, decimal price)
    {
        var update = new PriceUpdate(symbol, price, DateTime.UtcNow);
        foreach (var observer in _observers)
            observer.OnNext(update);
    }
}

public class PriceDisplay : IObserver<PriceUpdate>
{
    public void OnNext(PriceUpdate value)
        => Console.WriteLine($"{value.Symbol}: {value.Price}");
    
    public void OnError(Exception error) { /* handle */ }
    public void OnCompleted() { }
}

// Использование
var service = new PriceService();
using var sub = service.Subscribe(new PriceDisplay());
service.UpdatePrice("AAPL", 150.25m);
```

---

## Command

**Факт:** Инкапсулирует запрос как объект (undo/redo, queue, logging).

```csharp
public interface ICommand
{
    Task ExecuteAsync();
    Task UndoAsync();
}

public class TransferCommand : ICommand
{
    private readonly IAccountRepository _repo;
    private readonly Guid _fromId;
    private readonly Guid _toId;
    private readonly decimal _amount;
    
    public async Task ExecuteAsync()
    {
        await _repo.WithdrawAsync(_fromId, _amount);
        await _repo.DepositAsync(_toId, _amount);
    }
    
    public async Task UndoAsync()
    {
        await _repo.DepositAsync(_fromId, _amount);
        await _repo.WithdrawAsync(_toId, _amount);
    }
}

// Command Invoker — очередь команд
public class CommandQueue
{
    private readonly Queue<ICommand> _commands = new();
    private readonly Stack<ICommand> _history = new();
    
    public void Enqueue(ICommand command) => _commands.Enqueue(command);
    
    public async Task ExecuteNextAsync()
    {
        if (_commands.TryDequeue(out var command))
        {
            await command.ExecuteAsync();
            _history.Push(command);
        }
    }
    
    public async Task UndoLastAsync()
    {
        if (_history.TryPop(out var command))
            await command.UndoAsync();
    }
}
```

---

## Template Method

```csharp
public abstract class DataExporter
{
    // Template method — определяет шаги
    public async Task<byte[]> ExportAsync(IEnumerable<Record> data)
    {
        var transformed = Transform(data);
        var serialized = Serialize(transformed);
        return await PostProcess(serialized);
    }
    
    // Шаги, которые могут быть переопределены
    protected abstract IEnumerable<Record> Transform(IEnumerable<Record> data);
    protected abstract byte[] Serialize(IEnumerable<Record> data);
    
    // Шаг с реализацией по умолчанию (hook)
    protected virtual Task<byte[]> PostProcess(byte[] data) => Task.FromResult(data);
}

public class CsvExporter : DataExporter
{
    protected override IEnumerable<Record> Transform(IEnumerable<Record> data) => data;
    
    protected override byte[] Serialize(IEnumerable<Record> data)
    {
        var csv = string.Join("\n", data.Select(r => $"{r.Id},{r.Name}"));
        return Encoding.UTF8.GetBytes(csv);
    }
    
    protected override async Task<byte[]> PostProcess(byte[] data)
    {
        return await GZipAsync(data);  // Hook — сжатие
    }
}
```

---

## State

```csharp
public interface IOrderState
{
    Task<IOrderState> ProcessPaymentAsync();
    Task<IOrderState> ShipAsync();
    Task<IOrderState> CancelAsync();
    string Status { get; }
}

public class PendingState : IOrderState
{
    public Task<IOrderState> ProcessPaymentAsync() => Task.FromResult<IOrderState>(new PaidState());
    public Task<IOrderState> ShipAsync() => throw new InvalidOperationException("Not paid");
    public Task<IOrderState> CancelAsync() => Task.FromResult<IOrderState>(new CancelledState());
    public string Status => "Pending";
}

public class PaidState : IOrderState
{
    public Task<IOrderState> ProcessPaymentAsync() => throw new InvalidOperationException("Already paid");
    public Task<IOrderState> ShipAsync() => Task.FromResult<IOrderState>(new ShippedState());
    public Task<IOrderState> CancelAsync() => Task.FromResult<IOrderState>(new RefundedState());
    public string Status => "Paid";
}

public class Order
{
    private IOrderState _state = new PendingState();
    
    public async Task ProcessPaymentAsync() => _state = await _state.ProcessPaymentAsync();
    public async Task ShipAsync() => _state = await _state.ShipAsync();
    public async Task CancelAsync() => _state = await _state.CancelAsync();
    public string Status => _state.Status;
}
```

---

## Chain of Responsibility

```csharp
public abstract class ApprovalHandler
{
    private ApprovalHandler? _next;
    
    public ApprovalHandler SetNext(ApprovalHandler next)
    {
        _next = next;
        return next;
    }
    
    public virtual async Task<ApprovalResult> HandleAsync(ApprovalRequest request)
    {
        var result = await ProcessAsync(request);
        return result ?? await (_next?.HandleAsync(request) 
            ?? Task.FromResult(ApprovalResult.Denied("No handler")));
    }
    
    protected abstract Task<ApprovalResult?> ProcessAsync(ApprovalRequest request);
}

public class ManagerApproval : ApprovalHandler
{
    protected override Task<ApprovalResult?> ProcessAsync(ApprovalRequest request)
        => request.Amount <= 1000
            ? Task.FromResult<ApprovalResult?>(ApprovalResult.Approved("Manager"))
            : Task.FromResult<ApprovalResult?>(null);
}

public class DirectorApproval : ApprovalHandler
{
    protected override Task<ApprovalResult?> ProcessAsync(ApprovalRequest request)
        => request.Amount <= 10000
            ? Task.FromResult<ApprovalResult?>(ApprovalResult.Approved("Director"))
            : Task.FromResult<ApprovalResult?>(null);
}

// Chain: Manager → Director → CEO
var handler = new ManagerApproval();
handler.SetNext(new DirectorApproval()).SetNext(new CeoApproval());
var result = await handler.HandleAsync(new ApprovalRequest(5000));
```

---

## Mediator

```csharp
// Mediator — централизация коммуникации между компонентами
public interface IChatMediator
{
    Task SendMessageAsync(User sender, string message);
    Task JoinAsync(User user);
}

public class ChatRoom : IChatMediator
{
    private readonly List<User> _users = new();
    
    public Task SendMessageAsync(User sender, string message)
    {
        foreach (var user in _users.Where(u => u != sender))
            user.ReceiveMessage(sender, message);
        return Task.CompletedTask;
    }
    
    public Task JoinAsync(User user) { _users.Add(user); return Task.CompletedTask; }
}
```

---

## Чек-лист

- [ ] Strategy: взаимозаменяемые алгоритмы, DI
- [ ] Observer: события, IObservable/IObserver, Rx.NET
- [ ] Command: undo/redo, очередь команд
- [ ] Template Method: скелет алгоритма, hook методы
- [ ] State: конечный автомат, смена поведения
- [ ] Chain of Responsibility: цепочка обработчиков
- [ ] Mediator: централизация коммуникации
- [ ] Iterator: foreach, yield return
- [ ] Memento: снимок состояния (serialization)

# Задачи для собеседований: Архитектура, GC, Паттерны — с ответами

---

## Задача 1: Разница Scoped vs Transient vs Singleton DI

**Условие:** Есть сервис, который хранит `RequestId`. Какой lifetime выбрать? Что будет, если зарегистрировать как Singleton?

```csharp
// ❌ Singleton — все запросы делят один RequestId!
public class RequestContext
{
    public string RequestId { get; set; }  // Один экземпляр на ВСЁ приложение
}

builder.Services.AddSingleton<RequestContext>();
// Запрос 1: RequestId = "abc"
// Запрос 2: перезапишет RequestId = "xyz" — гонка данных!

// ✅ Scoped — новый экземпляр на каждый HTTP-запрос
builder.Services.AddScoped<RequestContext>();
// Каждый запрос получает свой экземпляр — изоляция

// ✅ Transient — новый экземпляр при каждом Resolve
builder.Services.AddTransient<IDatabaseConnectionFactory, SqlConnectionFactory>();
// Каждый вызов создаёт новый connection factory
```

**Ловушка:** Scoped сервис, захваченный Singleton:

```csharp
builder.Services.AddScoped<RequestContext>();
builder.Services.AddSingleton<IMessageProcessor, MessageProcessor>();

public class MessageProcessor : IMessageProcessor
{
    private readonly RequestContext _context;  // ❌ Scoped внутри Singleton!
    
    public MessageProcessor(RequestContext context) => _context = context;
    // Scoped сервис живёт пока жив Singleton → memory leak + неверный RequestId!
}
// Решение: IServiceScopeFactory или перевести MessageProcessor в Scoped
```

**Почему спрашивают:** Captive dependency — классическая ошибка DI. Проверяет понимание lifetime management и DI-контейнера.

---

## Задача 2: Когда вызывается финализатор?

**Условие:** Объяснить порядок вызова конструктора, Dispose, финализатора в этом коде:

```csharp
public class ResourceHolder : IDisposable
{
    private IntPtr _handle;
    
    public ResourceHolder()
    {
        _handle = NativeMethods.Allocate();
        Console.WriteLine("Constructor");
    }
    
    ~ResourceHolder()
    {
        Console.WriteLine("Finalizer");
        Dispose(false);
    }
    
    public void Dispose()
    {
        Console.WriteLine("Dispose (public)");
        Dispose(true);
        GC.SuppressFinalize(this);
    }
    
    protected virtual void Dispose(bool disposing)
    {
        Console.WriteLine($"Dispose (disposing={disposing})");
        if (_handle != IntPtr.Zero)
        {
            NativeMethods.Free(_handle);
            _handle = IntPtr.Zero;
        }
    }
}

// Использование 1: using
using (var r = new ResourceHolder()) { }
// Constructor → Dispose (public) → Dispose (disposing=True)
// Финализатор НЕ вызывается (GC.SuppressFinalize)

// Использование 2: забыли Dispose
var r = new ResourceHolder();
// Constructor
// ... позже GC:
// Finalizer → Dispose (disposing=False)

// Использование 3: неправильно
var r = new ResourceHolder();
r.Dispose();
r.Dispose();  // Второй вызов — безопасно (проверка _handle != IntPtr.Zero)
// ... позже GC: финализатор НЕ вызывается (SuppressFinalize)
```

**Почему спрашивают:** Финализаторы — одна из самых непонимаемых частей .NET. Проверяет знание Dispose pattern, SuppressFinalize, разницу managed/unmanaged ресурсов.

---

## Задача 3: Структура GC Heap — что где лежит?

**Условие:** Для каждого объекта указать, в какой heap он попадёт:

```csharp
// SOH (Small Object Heap) — Gen 0/1/2, объекты < 85000 байт
byte[] small = new byte[1000];         // SOH

// LOH (Large Object Heap) — объекты >= 85000 байт (.NET 5+)
byte[] large = new byte[100_000];      // LOH

// POH (Pinned Object Heap) — .NET 5+, закреплённые объекты
byte[] pinned = GC.AllocateArray<byte>(1000, pinned: true);  // POH

// ArrayPool
byte[] rented = ArrayPool<byte>.Shared.Rent(1024);  // SOH (из пула)
ArrayPool<byte>.Shared.Return(rented);

// Frozen Heap (NonGC Heap) — .NET 8+
FrozenSet<string> frozen = new[] { "a", "b", "c" }.ToFrozenSet();
// FrozenSet внутри использует NonGC Heap

// Интернированные строки — special heap
string interned = string.Intern("hello");  // String Intern Pool
```

**Почему спрашивают:** Понимание структуры managed heap — ключ к memory-оптимизации. Senior знает SOH/LOH/POH и когда что использовать.

---

## Задача 4: Разница между struct и class — когда struct?

**Условие:** Объяснить, почему этот код аллоцирует память, и как исправить:

```csharp
// Много аллокаций — каждый KeyValuePair это class? Нет, struct!
// Но лист растёт — каждый List.Add потенциально удваивает массив

public List<KeyValuePair<string, int>> ProcessLargeData(string[] keys)
{
    var result = new List<KeyValuePair<string, int>>();  // Аллокация: List + массив
    foreach (var key in keys)  // keys.Length = 1_000_000
    {
        result.Add(new KeyValuePair<string, int>(key, key.Length));
        // KeyValuePair — struct, НО List.Add копирует его в массив
        // При росте List — копирование ВСЕГО массива
    }
    return result;
}

// ✅ Исправление 1: Capacity
var result = new List<KeyValuePair<string, int>>(keys.Length);
// Предварительно аллоцирует массив — без роста

// ✅ Исправление 2: Array (если размер известен)
var result = new KeyValuePair<string, int>[keys.Length];
for (int i = 0; i < keys.Length; i++)
    result[i] = new(keys[i], keys[i].Length);

// Когда struct, а когда class?
// Struct: маленький (< 16 байт), неизменяемый, короткоживущий, нет наследования
// Class: всё остальное
```

**Почему спрашивают:** Разница struct/class — фундаментальная. Проверяет понимание value vs reference types, копирования, аллокаций.

---

## Задача 5: Event Sourcing — банковский счёт

**Условие:** Реализовать банковский счёт через Event Sourcing. Баланс — проекция событий.

```csharp
// События
public abstract record AccountEvent(Guid AccountId, DateTime Timestamp);
public record AccountOpened(Guid AccountId, string Owner, DateTime Timestamp) : AccountEvent(AccountId, Timestamp);
public record MoneyDeposited(Guid AccountId, decimal Amount, DateTime Timestamp) : AccountEvent(AccountId, Timestamp);
public record MoneyWithdrawn(Guid AccountId, decimal Amount, DateTime Timestamp) : AccountEvent(AccountId, Timestamp);

// Aggregate
public class BankAccount
{
    private readonly List<AccountEvent> _events = new();
    public Guid Id { get; private set; }
    public string Owner { get; private set; }
    public decimal Balance { get; private set; }
    
    public static BankAccount Open(Guid id, string owner)
    {
        var account = new BankAccount();
        account.Apply(new AccountOpened(id, owner, DateTime.UtcNow));
        return account;
    }
    
    public void Deposit(decimal amount)
    {
        if (amount <= 0) throw new ArgumentException("Amount must be positive");
        Apply(new MoneyDeposited(Id, amount, DateTime.UtcNow));
    }
    
    public void Withdraw(decimal amount)
    {
        if (amount <= 0) throw new ArgumentException("Amount must be positive");
        if (Balance < amount) throw new InvalidOperationException("Insufficient funds");
        Apply(new MoneyWithdrawn(Id, amount, DateTime.UtcNow));
    }
    
    private void Apply(AccountEvent @event)
    {
        _events.Add(@event);
        
        // Проекция состояния из событий
        switch (@event)
        {
            case AccountOpened e:
                Id = e.AccountId;
                Owner = e.Owner;
                break;
            case MoneyDeposited e:
                Balance += e.Amount;
                break;
            case MoneyWithdrawn e:
                Balance -= e.Amount;
                break;
        }
    }
    
    // Восстановление из истории
    public static BankAccount FromHistory(IEnumerable<AccountEvent> events)
    {
        var account = new BankAccount();
        foreach (var @event in events)
            account.Apply(@event);
        return account;
    }
    
    // Сохраняем ТОЛЬКО события, не состояние
    public IReadOnlyList<AccountEvent> GetUncommittedEvents() => _events.AsReadOnly();
}
```

**Почему спрашивают:** Event Sourcing — ключевой enterprise-паттерн. Проверяет понимание DDD, CQRS, separation of concerns. Обратите внимание: состояние — проекция событий, а не первичный источник.

---

## Задача 6: Unit of Work + Repository (без Entity Framework)

**Условие:** Реализовать простой Unit of Work с транзакцией.

```csharp
public interface IUnitOfWork : IDisposable
{
    IRepository<T> Repository<T>() where T : class;
    Task CommitAsync();
    Task RollbackAsync();
}

public class UnitOfWork : IUnitOfWork
{
    private readonly DbConnection _connection;
    private DbTransaction? _transaction;
    private readonly Dictionary<Type, object> _repositories = new();
    
    public UnitOfWork(DbConnection connection)
    {
        _connection = connection;
        _connection.Open();
    }
    
    public IRepository<T> Repository<T>() where T : class
    {
        if (!_repositories.TryGetValue(typeof(T), out var repo))
        {
            repo = new Repository<T>(_connection, _transaction!);
            _repositories[typeof(T)] = repo;
        }
        return (IRepository<T>)repo;
    }
    
    public async Task CommitAsync()
    {
        if (_transaction != null)
        {
            await _transaction.CommitAsync();
            await _transaction.DisposeAsync();
        }
    }
    
    public async Task RollbackAsync()
    {
        if (_transaction != null)
        {
            await _transaction.RollbackAsync();
            await _transaction.DisposeAsync();
        }
    }
    
    public void Dispose()
    {
        _transaction?.Dispose();
        _connection.Dispose();
    }
}
```

**Почему спрашивают:** Понимание, что стоит за EF Core DbContext. UoW + Repository — фундамент data access layer.

---

## Задача 7: Связный список с удалением за O(1)

**Условие:** Реализовать двусвязный список, где удаление узла — O(1) (по ссылке на узел, без поиска).

```csharp
public class FastLinkedList<T>
{
    private Node? _head;
    private Node? _tail;
    private int _count;
    
    public int Count => _count;
    
    public Node AddLast(T value)
    {
        var node = new Node(value);
        if (_tail == null)
        {
            _head = _tail = node;
        }
        else
        {
            _tail.Next = node;
            node.Previous = _tail;
            _tail = node;
        }
        _count++;
        return node;
    }
    
    // Удаление за O(1) — по ссылке на узел!
    public void Remove(Node node)
    {
        if (node.Previous != null)
            node.Previous.Next = node.Next;
        else
            _head = node.Next;
        
        if (node.Next != null)
            node.Next.Previous = node.Previous;
        else
            _tail = node.Previous;
        
        _count--;
    }
    
    public class Node(T value)
    {
        public T Value { get; } = value;
        public Node? Next { get; set; }
        public Node? Previous { get; set; }
    }
}

// Использование — LRU Cache:
var list = new FastLinkedList<string>();
var node1 = list.AddLast("item1");
var node2 = list.AddLast("item2");
list.Remove(node1);  // O(1) — без поиска!
```

**Почему спрашивают:** Прямая аналогия с `LinkedList<T>` в .NET. Проверяет понимание, когда связный список эффективен (удаление по ссылке), а когда нет (поиск O(n), cache misses).

---

## Задача 8: Сага (Saga) — распределённая транзакция с компенсацией

**Условие:** Реализовать упрощённую Saga для создания заказа: резервирование товара → списание денег → подтверждение. Если любой шаг падает — компенсация.

```csharp
public class CreateOrderSaga
{
    private readonly IStockService _stock;
    private readonly IPaymentService _payment;
    private readonly IOrderRepository _orders;
    
    public async Task<Result<Order>> ExecuteAsync(CreateOrderCommand cmd)
    {
        // Шаг 1: Резервирование
        var reservation = await _stock.ReserveAsync(cmd.Items);
        if (reservation.IsError)
            return reservation.Error;
        
        // Шаг 2: Оплата (компенсация: отмена резерва)
        var payment = await _payment.ChargeAsync(cmd.CustomerId, cmd.Total);
        if (payment.IsError)
        {
            await _stock.ReleaseAsync(reservation.Value.ReservationId);  // Компенсация!
            return payment.Error;
        }
        
        // Шаг 3: Создание заказа (компенсация: возврат + отмена резерва)
        try
        {
            var order = await _orders.CreateAsync(cmd, reservation.Value, payment.Value);
            return order;
        }
        catch
        {
            await _payment.RefundAsync(payment.Value.TransactionId);      // Компенсация!
            await _stock.ReleaseAsync(reservation.Value.ReservationId);    // Компенсация!
            throw;
        }
    }
}

// Шаги саги — каждый с compensate:
public record SagaStep<TInput, TOutput>(
    Func<TInput, Task<Result<TOutput>>> Execute,
    Func<TOutput, Task> Compensate);

public class Saga<TInput>
{
    private readonly List<object> _steps = new();
    
    public Saga<TInput> AddStep<TStepOutput>(
        Func<object, Task<Result<TStepOutput>>> execute,
        Func<TStepOutput, Task> compensate)
    {
        _steps.Add(new SagaStep<object, TStepOutput>(
            async input => await execute(input),
            async output => await compensate(output)));
        return this;
    }
    
    public async Task<Result> ExecuteAsync(TInput input)
    {
        var executed = new List<object>();
        
        foreach (dynamic step in _steps)
        {
            var result = await step.Execute(executed.Count == 0 ? input : executed.Last());
            
            if (result.IsError)
            {
                // Компенсация в обратном порядке!
                for (int i = executed.Count - 1; i >= 0; i--)
                {
                    dynamic s = _steps[i];
                    await s.Compensate(executed[i]);
                }
                return result.Error;
            }
            
            executed.Add(result.Value);
        }
        
        return Result.Ok();
    }
}
```

**Почему спрашивают:** Saga — критический паттерн для микросервисов. Проверяет понимание распределённых транзакций, compensating transactions, eventual consistency.

---

## Задача 9: Разница Singleton vs Static class

**Условие:** Когда Singleton-паттерн через DI, а когда static class?

```csharp
// Static class — нельзя mock, нельзя заменить реализацию
public static class DateTimeHelper
{
    public static bool IsWeekend(DateTime date)
        => date.DayOfWeek is DayOfWeek.Saturday or DayOfWeek.Sunday;
}

// Singleton через DI — можно mock, можно заменить
public interface IDateTimeProvider
{
    DateTime UtcNow { get; }
}

public class SystemDateTimeProvider : IDateTimeProvider
{
    public DateTime UtcNow => DateTime.UtcNow;
}

// В тестах — mock:
var mock = new Mock<IDateTimeProvider>();
mock.Setup(p => p.UtcNow).Returns(new DateTime(2024, 1, 1));
// Тестируем код с фиксированной датой!

// Static — для stateless pure functions (Math, Path, Enumerable)
// Singleton DI — для сервисов, которые могут меняться (time, config, network)
```

**Почему спрашивают:** Проверяет понимание testability, DI, разницу между stateless и stateful.

---

## Задача 10: GC Pressure — как измерить и уменьшить?

**Условие:** Дан код обработки 1M записей. Найти источники аллокаций и оптимизировать:

```csharp
// До оптимизации
public List<string> ProcessRecords(string[] lines)
{
    var results = new List<string>();
    foreach (var line in lines)
    {
        var parts = line.Split(',');             // string[] аллокация!
        var name = parts[0].Trim();              // string аллокация!
        var category = parts[1].ToUpper();       // string аллокация!
        var price = decimal.Parse(parts[2]);
        
        if (price > 100)
            results.Add($"{category}: {name}");  // string аллокация!
    }
    return results;
}

// После оптимизации
public List<string> ProcessRecordsOptimized(ReadOnlySpan<string> lines)
{
    var results = new List<string>();
    
    foreach (var line in lines)
    {
        ReadOnlySpan<char> span = line.AsSpan();
        
        // Поиск запятых без аллокаций
        int comma1 = span.IndexOf(',');
        int comma2 = span.Slice(comma1 + 1).IndexOf(',') + comma1 + 1;
        
        ReadOnlySpan<char> name = span[..comma1].Trim();
        ReadOnlySpan<char> category = span[(comma1 + 1)..comma2];
        ReadOnlySpan<char> priceSpan = span[(comma2 + 1)..];
        
        decimal.TryParse(priceSpan, NumberStyles.Any, CultureInfo.InvariantCulture, out decimal price);
        
        if (price > 100)
        {
            // Только одна аллокация — результирующая строка
            results.Add($"{category.ToString().ToUpperInvariant()}: {name}");
        }
    }
    
    return results;
}
```

**Сравнение аллокаций на 1M строк:**
| Метод | Аллокаций | Байт |
|---|---|---|
| Исходный | ~6 000 000 | ~480 MB |
| Span-based | ~500 000 | ~64 MB |

**Почему спрашивают:** Оптимизация аллокаций — ключевой навык для high-performance .NET. Проверяет понимание Span, string allocations, GC pressure.

---

## Чек-лист

- [ ] DI lifetimes: Singleton/Scoped/Transient, captive dependency
- [ ] Dispose pattern: финализатор, SuppressFinalize, managed/unmanaged
- [ ] GC Heap: SOH, LOH, POH, NonGC Heap, когда что использовать
- [ ] Struct vs Class: когда struct, копирование, аллокации
- [ ] Event Sourcing: события как source of truth, проекция состояния
- [ ] Unit of Work + Repository: без EF Core, транзакции
- [ ] LinkedList: O(1) удаление по ссылке, vs O(n) поиск
- [ ] Saga: compensating transactions, обратный порядок компенсации
- [ ] Singleton vs Static: testability, DI, pure functions
- [ ] GC Pressure: Span для zero-allocation, бенчмарк аллокаций

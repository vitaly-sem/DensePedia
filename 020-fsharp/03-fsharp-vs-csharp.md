# F# там, где он лучше C# — прямые сравнения

---

## Не честно сравнивать языки по синтаксису. Давайте сравним по реальным сценариям.

Каждый раздел ниже — один и тот же сценарий, реализованный на C# и на F#. Выводы делайте сами.

---

## 1. Domain Modelling — моделирование предметной области

### Сценарий: интернет-магазин, состояние заказа

```csharp
// C# — 25 строк
public enum OrderStatus { Pending, Confirmed, Shipped, Delivered, Cancelled }

public class Order
{
    public Guid Id { get; set; }
    public Guid CustomerId { get; set; }
    public List<OrderItem> Items { get; set; }
    public decimal Total { get; set; }
    public OrderStatus Status { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? ShippedAt { get; set; }
    public string CancellationReason { get; set; }
    
    // Бизнес-правило: ShippedAt только для Shipped
    // Бизнес-правило: CancellationReason только для Cancelled
    // Но типовая система C# НЕ гарантирует эти правила!
}

// Использование (баг невидим для компилятора!)
var order = new Order
{
    Status = OrderStatus.Pending,
    CancellationReason = "whatever"  // Не должно быть, но компилятор молчит
};
```

```fsharp
// F# — 12 строк + КОМПИЛЯТОР гарантирует бизнес-правила
type OrderItem = { ProductId: Guid; Quantity: int; Price: decimal }

type Order =
    | Pending of id: Guid * customerId: Guid * items: OrderItem list * total: decimal * createdAt: DateTime
    | Confirmed of id: Guid * customerId: Guid * items: OrderItem list * total: decimal * createdAt: DateTime
    | Shipped of id: Guid * customerId: Guid * items: OrderItem list * total: decimal * createdAt: DateTime * shippedAt: DateTime
    | Delivered of id: Guid * customerId: Guid * items: OrderItem list * total: decimal * createdAt: DateTime * shippedAt: DateTime * deliveredAt: DateTime
    | Cancelled of id: Guid * customerId: Guid * items: OrderItem list * total: decimal * createdAt: DateTime * reason: string

// CancellationReason существует ТОЛЬКО для Cancelled!
// ShippedAt существует ТОЛЬКО для Shipped и Delivered!
// НЕВОЗМОЖНО создать Pending с CancellationReason!

let order = Cancelled(
    Guid.NewGuid(), Guid.NewGuid(), [], 0m, DateTime.UtcNow,
    reason = "Customer request")
```

**Вердикт:** F# **3x короче** + компилятор гарантирует корректность состояния. В C# вы пишете валидацию в рантайме (и тесты на неё). В F# компилятор делает это за вас.

---

## 2. Обработка ошибок — без исключений

### Сценарий: оформление заказа с валидацией, проверкой склада, оплатой

```csharp
// C# — исключения + try-catch
public async Task<OrderConfirmation> PlaceOrderAsync(PlaceOrderRequest request)
{
    try
    {
        var user = await _userRepo.GetByIdAsync(request.UserId);
        if (user == null)
            throw new NotFoundException("User not found");
        
        if (!user.IsActive)
            throw new DomainException("User is inactive");
        
        var stock = await _stockService.CheckAvailabilityAsync(request.Items);
        if (!stock.AllAvailable)
            throw new DomainException($"Insufficient stock: {string.Join(", ", stock.Unavailable)}");
        
        var payment = await _paymentService.ChargeAsync(user, request.Total);
        if (payment.Status != PaymentStatus.Success)
            throw new DomainException($"Payment failed: {payment.ErrorMessage}");
        
        var order = await _orderRepo.CreateAsync(user.Id, request.Items, request.Total);
        return order;
    }
    catch (NotFoundException ex) { /* 404 */ throw; }
    catch (DomainException ex) { /* 400 */ throw; }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Unexpected error placing order");
        throw new InternalServerException("Something went wrong");
    }
}
```

```fsharp
// F# — Result type, без исключений
type PlaceOrderError =
    | UserNotFound of userId: Guid
    | UserInactive of userId: Guid
    | InsufficientStock of unavailable: (Guid * int) list
    | PaymentFailed of reason: string
    | UnexpectedError of exn

let placeOrderAsync (request: PlaceOrderRequest) = asyncResult {
    // Каждый шаг возвращает Result<T, PlaceOrderError>
    // let! — разворачивает Ok, если Error — сразу возвращает Error
    
    let! user = userRepo.GetByIdAsync request.UserId
                |> AsyncResult.requireSome (UserNotFound request.UserId)
    
    do! user.IsActive |> Result.requireTrue (UserInactive request.UserId)
    
    let! stock = stockService.CheckAvailabilityAsync request.Items
    do! stock.AllAvailable |> Result.requireTrue (
        InsufficientStock stock.Unavailable)
    
    let! payment = paymentService.ChargeAsync user request.Total
    do! (payment.Status = Success) |> Result.requireTrue (
        PaymentFailed payment.ErrorMessage)
    
    let! order = orderRepo.CreateAsync user.Id request.Items request.Total
    return order
}

// В контроллере — один match обрабатывает ВСЕ ошибки
match! placeOrderAsync request with
| Ok confirmation -> Ok confirmation
| Error UserNotFound id -> NotFound $"User {id} not found"
| Error UserInactive id -> BadRequest "User inactive"
| Error InsufficientStock items -> BadRequest $"Insufficient stock: {items}"
| Error PaymentFailed reason -> BadRequest $"Payment: {reason}"
| Error UnexpectedError ex -> StatusCode 500
```

**Вердикт:** C# — вложенные try-catch, нужно помнить какие исключения бросает каждый метод. F# — все возможные ошибки закодированы в типе `PlaceOrderError`. Компилятор гарантирует, что вы обработаете **все** случаи.

---

## 3. Трансформация данных — pipeline vs LINQ

### Сценарий: анализ заказов за неделю

```csharp
// C# — LINQ (хорошо, но ограниченно)
var weeklyStats = orders
    .Where(o => o.CreatedAt >= DateTime.UtcNow.AddDays(-7))
    .GroupBy(o => o.CustomerId)
    .Select(g => new CustomerStats
    {
        CustomerId = g.Key,
        OrderCount = g.Count(),
        TotalSpent = g.Sum(o => o.Total),
        AverageOrder = g.Average(o => o.Total),
        LargestOrder = g.Max(o => o.Total),
        TopCategory = g
            .SelectMany(o => o.Items)
            .GroupBy(i => i.Category)
            .OrderByDescending(cg => cg.Sum(i => i.Price * i.Quantity))
            .First().Key
    })
    .Where(s => s.TotalSpent > 1000)
    .OrderByDescending(s => s.TotalSpent)
    .Take(10)
    .ToList();
```

```fsharp
// F# — pipeline (читаемее, легче отлаживать)
let weeklyStats =
    orders
    |> List.filter (fun o -> o.CreatedAt >= DateTime.UtcNow.AddDays(-7))
    |> List.groupBy (fun o -> o.CustomerId)
    |> List.map (fun (customerId, customerOrders) ->
        let items = customerOrders |> List.collect (fun o -> o.Items)
        let topCategory =
            items
            |> List.groupBy (fun i -> i.Category)
            |> List.map (fun (cat, catItems) -> cat, catItems |> List.sumBy (fun i -> i.Price * decimal i.Quantity))
            |> List.sortByDescending snd
            |> List.head
            |> fst
        
        { CustomerId = customerId
          OrderCount = customerOrders.Length
          TotalSpent = customerOrders |> List.sumBy (fun o -> o.Total)
          AverageOrder = customerOrders |> List.averageBy (fun o -> o.Total)
          LargestOrder = customerOrders |> List.maxBy (fun o -> o.Total)
          TopCategory = topCategory })
    |> List.filter (fun s -> s.TotalSpent > 1000m)
    |> List.sortByDescending (fun s -> s.TotalSpent)
    |> List.truncate 10
```

**Вердикт:** LINQ хорош, но pipeline в F# позволяет **вставить `printfn` в любом месте цепочки** для отладки. В LINQ нужно материализовать промежуточный результат или использовать breakpoint.

---

## 4. Древовидные структуры — рекурсия vs циклы

### Сценарий: файловая система, подсчёт размера директории

```csharp
// C# — итеративный обход (стек)
public static long GetDirectorySize(string path)
{
    long totalSize = 0;
    var stack = new Stack<string>();
    stack.Push(path);
    
    while (stack.Count > 0)
    {
        var current = stack.Pop();
        foreach (var file in Directory.GetFiles(current))
        {
            totalSize += new FileInfo(file).Length;
        }
        foreach (var dir in Directory.GetDirectories(current))
        {
            stack.Push(dir);
        }
    }
    return totalSize;
}
```

```fsharp
// F# — рекурсивная функция (читается как определение)
let rec getDirectorySize path =
    let fileSize =
        Directory.GetFiles(path)
        |> Array.sumBy (fun f -> FileInfo(f).Length)
    
    let subDirSize =
        Directory.GetDirectories(path)
        |> Array.sumBy getDirectorySize  // Рекурсивно — читается естественно!
    
    fileSize + subDirSize
```

**Вердикт:** 4 строки F# vs 15 строк C#. Рекурсия в F# — нативный инструмент. Компилятор оптимизирует хвостовую рекурсию. Код читается как определение: «размер директории = сумма размеров файлов + сумма размеров поддиректорий».

---

## 5. Парсинг и трансформация — active patterns

### Сценарий: парсинг bank statement CSV с вариативным форматом

```csharp
// C# — if/else + exception handling
public static Transaction? ParseLine(string line)
{
    var parts = line.Split(',');
    if (parts.Length < 3) return null;
    
    if (DateTime.TryParse(parts[0], out var date) &&
        decimal.TryParse(parts[2], out var amount))
    {
        var description = parts[1];
        var sign = parts.Length > 3 ? parts[3] : "+";
        
        if (sign == "-")
            amount = -amount;
        
        return new Transaction(date, description, amount);
    }
    return null;
}
```

```fsharp
// F# — Active Patterns + Pattern Matching
let (|Date|_|) (s: string) =
    match DateTime.TryParse(s) with
    | true, dt -> Some dt
    | _ -> None

let (|Decimal|_|) (s: string) =
    match Decimal.TryParse(s) with
    | true, d -> Some d
    | _ -> None

let parseLine (line: string) =
    match line.Split(',') with
    | [| Date date; description; Decimal amount |] ->
        Some { Date = date; Description = description; Amount = amount }
    | [| Date date; description; Decimal amount; sign |] when sign = "-" ->
        Some { Date = date; Description = description; Amount = -amount }
    | _ -> None
```

**Вердикт:** Active Patterns позволяют превратить Pattern Matching в DSL для парсинга. Каждый `|Date|_|` — это переиспользуемый «кирпичик». Композиция шаблонов невозможна в C# без библиотек.

---

## 6. Immutable data + преобразования

### Сценарий: конвейер обработки заказа с несколькими шагами

```csharp
// C# — мутабельный подход (или with для records)
public record Order(Guid Id, decimal Total, OrderStatus Status, decimal? Discount);

public Order ApplyDiscount(Order order, decimal percent)
{
    return order with
    {
        Discount = order.Total * percent,
        Total = order.Total - order.Total * percent
    };
}

public Order EnrichWithMetadata(Order order)
{
    return order with
    {
        // ... копирование всех полей даже если меняем одно
    };
}
// Каждый метод возвращает новый объект, но синтаксис with — ограничен
```

```fsharp
// F# — Discriminated Union + record update
type OrderStatus = Pending | Confirmed | Shipped | Cancelled

type Order = {
    Id: Guid
    Total: decimal
    Status: OrderStatus
    Discount: decimal option
    Metadata: Map<string, string>
}

let applyDiscount percent (order: Order) =
    let discount = order.Total * percent
    { order with
        Total = order.Total - discount
        Discount = Some discount }

let addMetadata key value (order: Order) =
    { order with Metadata = order.Metadata.Add(key, value) }

// Pipeline — каждое преобразование создаёт НОВЫЙ Order
let processOrder order =
    order
    |> applyDiscount 0.15m
    |> addMetadata "source" "web"
    |> addMetadata "processedBy" "pipeline-v2"

// Type inference: processOrder: Order -> Order
```

**Вердикт:** F# `{ record with Field = value }` — лаконичнее C# `with`. Immutability по умолчанию означает, что вы **не можете случайно** мутировать состояние — компилятор запретит.

---

## 7. Многопоточность: MailboxProcessor vs Channels/Tasks

### Сценарий: банковский аккаунт с конкурентными операциями

```csharp
// C# — lock/mutex (ручное управление)
public class BankAccount
{
    private readonly object _lock = new();
    private decimal _balance;
    
    public BankAccount(decimal initialBalance) => _balance = initialBalance;
    
    public decimal Balance { get { lock (_lock) return _balance; } }
    
    public bool Withdraw(decimal amount)
    {
        lock (_lock)
        {
            if (_balance >= amount)
            {
                _balance -= amount;
                return true;
            }
            return false;
        }
    }
    
    public void Deposit(decimal amount)
    {
        lock (_lock) { _balance += amount; }
    }
}
// Concurrent access — нужно помнить lock, легко забыть
```

```fsharp
// F# — MailboxProcessor (Agent) — без lock'ов!
type AccountMessage =
    | Deposit of amount: decimal * reply: AsyncReplyChannel<decimal>
    | Withdraw of amount: decimal * reply: AsyncReplyChannel<Result<decimal, string>>
    | GetBalance of AsyncReplyChannel<decimal>

let createAccount (initialBalance: decimal) =
    MailboxProcessor.Start(fun inbox ->
        let rec loop balance = async {
            let! msg = inbox.Receive()
            match msg with
            | Deposit(amount, reply) ->
                let newBalance = balance + amount
                reply.Reply(newBalance)
                return! loop newBalance
            | Withdraw(amount, reply) ->
                if balance >= amount then
                    let newBalance = balance - amount
                    reply.Reply(Ok newBalance)
                    return! loop newBalance
                else
                    reply.Reply(Error "Insufficient funds")
                    return! loop balance
            | GetBalance(reply) ->
                reply.Reply(balance)
                return! loop balance
        }
        loop initialBalance)

// Использование — без lock'ов, без deadlock'ов
let account = createAccount 1000m

let balance = account.PostAndReply GetBalance  // 1000
let result = account.PostAndReply(fun reply -> Withdraw(300m, reply))
// Ok 700

// Все операции сериализованы через mailbox — race condition невозможен
```

**Вердикт:** MailboxProcessor — это actor model (как в Erlang/Akka) «из коробки». Нет lock'ов, нет ManualResetEvent, нет SemaphoreSlim. Модель простая: один поток обрабатывает сообщения последовательно. Для 95% сценариев этого достаточно.

---

## 8. Тестирование — property-based testing

```fsharp
// F# + FsCheck — property-based testing (PBT)
// Вместо «эти 3 примера работают» → «для ВСЕХ целых чисел свойство выполняется»

open FsCheck

// Тест: reverse (reverse list) = list для ЛЮБОГО списка
let ``reverse twice returns original`` (list: int list) =
    list |> List.rev |> List.rev = list

Check.Quick ``reverse twice returns original``
// Ok, passed 100 tests.

// Тест: сортировка не теряет элементы
let ``sort preserves length`` (list: int list) =
    List.sort list |> List.length = List.length list

Check.Quick ``sort preserves length``
// Ok, passed 100 tests. FsCheck генерирует edge cases автоматически!

// В C# для этого нужен NUnit Theory + ручные generatoры или библиотека FsCheck
```

**Вердикт:** Property-based testing встроен в экосистему F# (FsCheck, Expecto, Unquote). C# может использовать FsCheck тоже, но экосистема F# глубже интегрирована.

---

## 9. Итоговая таблица: F# лучше C# для...

| Сценарий | F# преимущество | Насколько |
|---|---|---|
| **Domain Modelling** | Компилятор гарантирует бизнес-правила через DU | 🔥🔥🔥🔥🔥 |
| **Обработка ошибок** | `Result<T,E>` — типизированные ошибки, нет исключений | 🔥🔥🔥🔥🔥 |
| **Трансформация данных** | Pipeline + модули коллекций + отладка в середине цепочки | 🔥🔥🔥🔥 |
| **Рекурсивные структуры** | Рекурсия нативная, tail-call оптимизация | 🔥🔥🔥🔥 |
| **Парсинг / DSL** | Active Patterns + Pattern Matching = DSL без библиотек | 🔥🔥🔥🔥 |
| **Неизменяемость** | По умолчанию, компилятор запрещает мутации | 🔥🔥🔥🔥 |
| **Многопоточность** | MailboxProcessor — actor model без lock'ов | 🔥🔥🔥🔥 |
| **Тестирование** | Property-based testing, Expecto, Unquote | 🔥🔥🔥 |
| **Скриптинг** | `.fsx` скрипты — выполнение без проекта | 🔥🔥🔥🔥🔥 |
| **Прототипирование** | Type Providers + REPL — увидеть результат немедленно | 🔥🔥🔥🔥🔥 |
| **CRUD / MVC / Web API** | C# проще (знакомые паттерны, больше примеров) | C# 🔥🔥🔥 |
| **Game Dev (Unity)** | Только C# | C# 🔥🔥🔥🔥🔥 |
| **Найм разработчиков** | C# разработчиков больше | C# 🔥🔥🔥🔥🔥 |

---

## Чек-лист: где F# даёт максимальный выигрыш

- [ ] Domain Modelling — замените enum + классы на Discriminated Union
- [ ] Валидация — замените исключения на `Result<T, Error>`
- [ ] Конвейеры данных — замените вложенные вызовы на `|>`
- [ ] Асинхронность — замените `Task.WhenAll` на `Async.Parallel`
- [ ] Многопоточное состояние — замените `lock` на `MailboxProcessor`
- [ ] Парсинг — замените regex/if на Active Patterns
- [ ] Скрипты автоматизации — замените PowerShell/ bash на `.fsx`
- [ ] Тесты — добавьте property-based тесты (FsCheck)

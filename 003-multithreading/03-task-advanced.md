# Task — продвинутые сценарии

---

## Task.WhenAll / WhenAny

### WhenAll — ожидание всех

```csharp
var task1 = _service.FetchUserAsync(id);
var task2 = _service.FetchOrdersAsync(id);
var task3 = _service.FetchProfileAsync(id);

// ✅ Параллельно + ожидание всех
await Task.WhenAll(task1, task2, task3);

// После WhenAll — .Result безопасен (Task уже завершён)
var user = task1.Result;
var orders = task2.Result;

// ⚠️ Обработка ошибок — AggregateException (до .NET 6)
try
{
    await Task.WhenAll(task1, task2, task3);
}
catch (Exception ex)
{
    // До .NET 6: ex — первый из AggregateException.InnerExceptions
    // .NET 6+: ex — первый упавший exception напрямую
    // Остальные исключения в Task.Exception (наблюдаемые)
}

// Получить все исключения
try
{
    await Task.WhenAll(tasks);
}
catch
{
    var exceptions = tasks
        .Where(t => t.IsFaulted)
        .SelectMany(t => t.Exception?.InnerExceptions ?? Enumerable.Empty<Exception>());
}
```

**Факт:** Task.WhenAll **не останавливается** при первом исключении — все задачи выполняются. Исключения агрегируются.

**Факт:** Если одна задача упала с `OperationCanceledException` — `Task.WhenAll` не бросает её как cancellation, а падает как обычное исключение.

### WhenAny — первый завершившийся

```csharp
// ✅ Первый успешный ответ (race pattern)
var completed = await Task.WhenAny(
    _primaryService.GetDataAsync(),
    _secondaryService.GetDataAsync());

var result = await completed;  // Результат первого

// ⚠️ Другие задачи продолжают выполняться!
// Нужно отменить остальные:
using var cts = new CancellationTokenSource();
var tasks = new[]
{
    _primaryService.GetDataAsync(cts.Token),
    _secondaryService.GetDataAsync(cts.Token)
};

var completed = await Task.WhenAny(tasks);
cts.Cancel();  // Отмена остальных

// Таймаут
using var timeoutCts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
var completed = await Task.WhenAny(
    _service.ProcessAsync(),
    Task.Delay(-1, timeoutCts.Token));  // ∞ до отмены
```

---

## TaskCompletionSource

**Факт:** `TaskCompletionSource<T>` создаёт `Task<T>`, который можно завершить вручную. Используется для конвертации callback-based API в Task-based.

```csharp
// 1. Преобразование EAP → Task
public Task<byte[]> ReadAsync(Stream stream, byte[] buffer, int offset, int count)
{
    var tcs = new TaskCompletionSource<byte[]>(
        TaskCreationOptions.RunContinuationsAsynchronously);  // ← важно!
    
    stream.BeginRead(buffer, offset, count, asyncResult =>
    {
        try
        {
            int bytesRead = stream.EndRead(asyncResult);
            tcs.TrySetResult(buffer[..bytesRead]);
        }
        catch (Exception ex)
        {
            tcs.TrySetException(ex);
        }
    }, null);
    
    return tcs.Task;
}

// 2. Преобразование APM → Task
public Task ConnectAsync(string host, int port)
{
    var tcs = new TaskCompletionSource(TaskCreationOptions.RunContinuationsAsynchronously);
    var socket = new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp);
    
    socket.BeginConnect(host, port, ar =>
    {
        try
        {
            socket.EndConnect(ar);
            tcs.TrySetResult();
        }
        catch (Exception ex)
        {
            tcs.TrySetException(ex);
        }
    }, null);
    
    return tcs.Task;
}
```

**Факт:** `TaskCreationOptions.RunContinuationsAsynchronously` — **критичен**! Без него, continuation (await) выполняется синхронно внутри `TrySetResult`, что может вызвать deadlock или неожиданный порядок выполнения.

---

## CancellationToken — кооперативная отмена

```csharp
public async Task ProcessBatchAsync(IEnumerable<Item> items, CancellationToken ct)
{
    // 1. Проверка перед началом
    ct.ThrowIfCancellationRequested();
    
    foreach (var item in items)
    {
        // 2. Проверка внутри цикла (периодическая)
        ct.ThrowIfCancellationRequested();
        
        // 3. Передача token в дочерние методы
        await ProcessItemAsync(item, ct);
    }
}

// Как создать CancellationToken:
using var cts = new CancellationTokenSource();
cts.CancelAfter(TimeSpan.FromSeconds(30));  // Автоотмена через 30 сек
var token = cts.Token;

// Composite (логическое ИЛИ)
using var timeoutCts = new CancellationTokenSource(TimeSpan.FromSeconds(10));
using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
    timeoutCts.Token, userCancellationToken);
await ProcessAsync(linkedCts.Token);
```

**Методы, поддерживающие CancellationToken:**

| Метод | Реакция на отмену |
|---|---|
| `Task.Delay(ms, ct)` | Бросает `OperationCanceledException` немедленно |
| `HttpClient.GetAsync(url, ct)` | Отменяет HTTP запрос |
| `Stream.ReadAsync(ct)` | Отменяет чтение |
| `Channel.Reader.ReadAsync(ct)` | Отменяет ожидание |
| `Task.Run(action, ct)` | Не отменяет action, но не начинает, если токен отозван |

**Факт:** `RegisterCancellation` — callback при отмене:

```csharp
using var cts = new CancellationTokenSource();
cts.Token.Register(() => Console.WriteLine("Cancelled!"));

// Для очистки ресурсов
cts.Token.Register(() =>
{
    _httpClient.CancelPendingRequests();
    _semaphore.Release();
});
```

---

## TaskScheduler

```csharp
// SynchronizationContextTaskScheduler — для UI потоков
// ThreadPoolTaskScheduler — по умолчанию

// Кастомный планировщик — ограничение параллелизма
public class SingleThreadTaskScheduler : TaskScheduler
{
    private readonly BlockingCollection<Task> _tasks = new();
    private readonly Thread _thread;
    
    public SingleThreadTaskScheduler()
    {
        _thread = new Thread(() =>
        {
            foreach (var task in _tasks.GetConsumingEnumerable())
                TryExecuteTask(task);
        })
        {
            IsBackground = true,
            Name = "SingleThreadScheduler"
        };
        _thread.Start();
    }
    
    protected override IEnumerable<Task> GetScheduledTasks() => _tasks;
    
    protected override void QueueTask(Task task) => _tasks.Add(task);
    
    protected override bool TryExecuteTaskInline(Task task, bool taskWasPreviouslyQueued)
        => Thread.CurrentThread == _thread && TryExecuteTask(task);
    
    protected override int MaximumConcurrencyLevel => 1;
}

// Использование
var scheduler = new SingleThreadTaskScheduler();
var factory = new TaskFactory(scheduler);
await factory.StartNew(() => SequentialWork());
```

**Факт:** `await` не использует TaskScheduler для продолжения — он использует `SynchronizationContext`. Чтобы продолжить на определённом scheduler'е, нужно:

```csharp
await Task.Run(() => { }).ConfigureAwait(false);
// Или:
await Task.Factory.StartNew(() => { }, CancellationToken.None, 
    TaskCreationOptions.None, scheduler).Unwrap();
```

---

## Чек-лист

- [ ] Task.WhenAll: все исключения, не останавливается при первой ошибке
- [ ] Task.WhenAny: отмена остальных задач после первой
- [ ] TaskCompletionSource: RunContinuationsAsynchronously
- [ ] CancellationToken: ThrowIfCancellationRequested, CreateLinkedTokenSource
- [ ] TaskScheduler: кастомные, ограничение параллелизма

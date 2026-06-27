# Thread Local Storage (TLS) и TPL Dataflow

---

## Thread Local Storage

### ThreadLocal\<T\>

**Факты:**
- Каждый поток получает **свою копию** значения
- `IsValueCreated` — проверка, инициализирован ли
- Значение **не передаётся** между потоками (через await — будет другой поток)

```csharp
// Каждый поток имеет свой Connection (избегаем синхронизации)
private static ThreadLocal<DbConnection> _connection = new(() =>
{
    var conn = new SqlConnection(connectionString);
    conn.Open();
    return conn;
}, trackAllValues: false);  // trackAllValues — если нужен перебор всех значений

public DbConnection Connection => _connection.Value!;

// ⚠️ Не забывайте очищать!
// ThreadLocal — IDisposable (держит dictionary всех значений)
~MyClass()
{
    _connection.Dispose();
}

// Перебор всех значений (trackAllValues = true)
private static ThreadLocal<int> _counters = new(trackAllValues: true);

void Report()
{
    foreach (var counter in _counters.Values)
        Console.WriteLine(counter);
}
```

**Факт:** `ThreadLocal<T>` хранит значения в **внутреннем dictionary** (ключ = Thread ID). При `trackAllValues = false` — GC может собрать значения мёртвых потоков.

**Факт:** `ThreadLocal<T>` — самый быстрый способ изоляции данных между потоками (без блокировок).

### AsyncLocal\<T\>

**Факты:**
- Аналог ThreadLocal, но для async (передаётся через await)
- Работает через **ExecutionContext** (копируется при await)
- Идеально для: correlation ID, transaction scope, логирование

```csharp
// AsyncLocal — значение "течёт" через await
private static AsyncLocal<string> _requestId = new();

public async Task HandleRequestAsync()
{
    _requestId.Value = Guid.NewGuid().ToString();  // Устанавливаем в начале
    _logger.LogInformation("Start request {Id}", _requestId.Value);
    
    await ProcessAsync();  // _requestId.Value сохраняется через await
    
    _logger.LogInformation("End request {Id}", _requestId.Value);
}

public async Task ProcessAsync()
{
    // _requestId.Value — то же, что установили выше
    _logger.LogInformation("Processing {Id}", _requestId.Value);
    
    await Task.Delay(100);
    // Task.Delay может сменить поток, но AsyncLocal сохранит значение
}

// Вложенные AsyncLocal
public class CorrelationContext
{
    private static readonly AsyncLocal<CorrelationContext> _current = new();
    
    public static CorrelationContext? Current => _current.Value;
    
    public static CorrelationContext Start()
    {
        var ctx = new CorrelationContext
        {
            CorrelationId = Guid.NewGuid(),
            StartedAt = DateTime.UtcNow
        };
        _current.Value = ctx;
        return ctx;
    }
    
    public Guid CorrelationId { get; private set; }
    public DateTime StartedAt { get; private set; }
}

// Использование
CorrelationContext.Start();
await DoWorkAsync();
// CorrelationContext.Current — доступен внутри await цепочки
```

**Факт:** `AsyncLocal<T>` использует **`ExecutionContext`**, который копируется при `await`. Это даёт overhead ~100-200ns на копирование. Для тысяч await — может быть заметно.

**Факт:** `ExecutionContext.SuppressFlow()` — отключает копирование (для fire-and-forget):

```csharp
using (ExecutionContext.SuppressFlow())
{
    Task.Run(() =>
    {
        // Здесь AsyncLocal НЕ будет доступен
        _ = DoFireAndForgetAsync();
    });
}
```

---

## TPL Dataflow

**Факты:**
- Библиотека: `System.Threading.Tasks.Dataflow` (NuGet)
- Pipeline обработки данных с буферизацией, параллелизмом и backpressure
- Блоки: `TransformBlock`, `ActionBlock`, `BufferBlock`, `BatchBlock`, `JoinBlock`

```csharp
// Pipeline: Download → Process → Save

// 1. Download блок (Transform — получает URL, отдаёт HTML)
var downloadBlock = new TransformBlock<string, string>(
    async url => await DownloadAsync(url),
    new ExecutionDataflowBlockOptions
    {
        MaxDegreeOfParallelism = 4,
        BoundedCapacity = 100,         // Backpressure: не больше 100 в очереди
    });

// 2. Process блок
var processBlock = new TransformBlock<string, ProcessedData>(
    async html => await ProcessAsync(html),
    new ExecutionDataflowBlockOptions
    {
        MaxDegreeOfParallelism = 8,
        BoundedCapacity = 200
    });

// 3. Save блок (ActionBlock — терминальный, без выхода)
var saveBlock = new ActionBlock<ProcessedData>(
    async data => await SaveToDbAsync(data),
    new ExecutionDataflowBlockOptions
    {
        MaxDegreeOfParallelism = 2,    // Ограничение БД
        BoundedCapacity = 50,
        SingleProducerConstrained = false
    });

// 4. Link blocks (с PropagateCompletion — завершение pipeline)
var linkOptions = new DataflowLinkOptions { PropagateCompletion = true };

downloadBlock.LinkTo(processBlock, linkOptions);
processBlock.LinkTo(saveBlock, linkOptions);

// 5. Отправка данных
foreach (var url in urls)
    await downloadBlock.SendAsync(url);  // Асинхронно, с backpressure

downloadBlock.Complete();  // Сигнал завершения
await saveBlock.Completion;  // Ожидание завершения всего pipeline
```

**Dataflow block types:**

| Block | Вход | Выход | Назначение |
|---|---|---|---|
| `BufferBlock<T>` | T | T | Очередь с bounded capacity |
| `TransformBlock<TInput,TOutput>` | TInput | TOutput | Преобразование 1→1 |
| `TransformManyBlock<TInput,TOutput>` | TInput | IEnumerable<TOutput> | Разделение (1→N) |
| `ActionBlock<T>` | T | — | Терминальный (Side-effect) |
| `BatchBlock<T>` | T | T[] | Группировка (N штук в массив) |
| `JoinBlock<T1,T2>` | T1, T2 | Tuple<T1,T2> | Соединение (из двух источников) |
| `BroadcastBlock<T>` | T | T (всем) | Вещание (каждому подписчику) |

**Пример с фильтрацией:**

```csharp
// Filter: передаём только валидные URL
downloadBlock.LinkTo(processBlock, linkOptions, url => url.IsValid());
downloadBlock.LinkTo(DataflowBlock.NullTarget<string>());  // Невалидные → в никуда
```

**Пример с TransformManyBlock:**

```csharp
var pageBlock = new TransformManyBlock<string, string>(
    async url =>
    {
        var html = await DownloadAsync(url);
        return ParseLinks(html);  // Возвращает IEnumerable<string>
    });
```

**Пример с BatchBlock:**

```csharp
var batchBlock = new BatchBlock<Order>(batchSize: 100);  // Группирует по 100

// После LinkTo — ActionBlock получит Order[100]
var batchSaveBlock = new ActionBlock<Order[]>(
    async orders => await BulkSaveAsync(orders));
```

**Backpressure (обратное давление):**

```
downloadBlock.SendAsync(url)
    ↑ если BoundedCapacity заполнен → SendAsync ждёт
    ↓
downloadBlock (BoundedCapacity=100)
    ↓ если processBlock заполнен → downloadBlock замедляется
processBlock (BoundedCapacity=200)
    ↓ если saveBlock заполнен → processBlock замедляется
saveBlock (BoundedCapacity=50)
```

**Факт:** BoundedCapacity создаёт **backpressure по всей цепочке** — каскадное замедление до источника.

**Факт:** Dataflow блоки **не завершаются автоматически**. Всегда вызывайте `Complete()` на первом блоке и используйте `PropagateCompletion`.

---

## Чек-лист

- [ ] ThreadLocal: изоляция данных между потоками, Dispose
- [ ] AsyncLocal: correlation ID через await, ExecutionContext
- [ ] TPL Dataflow: TransformBlock, ActionBlock, BufferBlock
- [ ] BoundedCapacity — backpressure
- [ ] PropagateCompletion — завершение pipeline
- [ ] BatchBlock для bulk операций
- [ ] TransformManyBlock для разделения (1 URL → много ссылок)

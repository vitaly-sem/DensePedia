# async/await — внутреннее устройство

---

## State Machine — как работает await

**Факт:** Компилятор C# превращает `async` метод в **state machine** (структура, реализующая `IAsyncStateMachine`).

```csharp
// Исходный код
public async Task<int> GetDataAsync()
{
    int localValue = 42;
    var result = await _service.FetchAsync();
    return result + localValue;
}

// То, во что компилятор превращает (упрощённо):
// 1. Структура-машина состояний
[AsyncStateMachine(typeof(GetDataAsyncStateMachine))]
public Task<int> GetDataAsync()
{
    var machine = new GetDataAsyncStateMachine
    {
        _builder = AsyncTaskMethodBuilder<int>.Create(),
        _state = -1  // -1 = начальное состояние
    };
    machine._builder.Start(ref machine);  // Запуск
    return machine._builder.Task;
}

// 2. MoveNext — основная логика
private struct GetDataAsyncStateMachine : IAsyncStateMachine
{
    public AsyncTaskMethodBuilder<int> _builder;
    public int _state;
    private TaskAwaiter<int> _awaiter;
    private int _localValue;
    
    public void MoveNext()
    {
        int result = default;
        try
        {
            if (_state == -1)  // Начальное состояние
            {
                _localValue = 42;
                _awaiter = _service.FetchAsync().GetAwaiter();
                
                if (!_awaiter.IsCompleted)
                {
                    _state = 0;  // Следующее состояние
                    _builder.AwaitUnsafeOnCompleted(ref _awaiter, ref this);
                    return;  // Выход → возврат управления
                }
            }
            
            if (_state == 0)  // После await
            {
                result = _awaiter.GetResult() + _localValue;
            }
        }
        catch (Exception ex)
        {
            _state = -2;  // Завершено с ошибкой
            _builder.SetException(ex);
            return;
        }
        
        _state = -2;  // Завершено успешно
        _builder.SetResult(result);
    }
    
    public void SetStateMachine(IAsyncStateMachine stateMachine) { }
}
```

**Факт:** Каждый `await` создаёт **новое состояние** в state machine. Цепочка из 10 await = 10 состояний.

**Факт:** State machine — **struct** (value type), не class. Если await выполняется синхронно (Task уже завершён), **аллокации нет**. Если асинхронно — struct box'ится в `MoveNextRunner`.

---

## SynchronizationContext — захват контекста

**Факт:** При `await` не-завершённого Task, текущий `SynchronizationContext.Current` захватывается. После завершения Task, продолжение (остаток метода) выполняется на захваченном контексте.

```csharp
// UI поток (WPF/WinForms)
var ctx = SynchronizationContext.Current;  // DispatcherSynchronizationContext

await Task.Delay(100);

// Продолжение — на DispatcherSynchronizationContext (UI поток)
this.TextBlock.Text = "Done";  // Безопасно

// ASP.NET Core — SynchronizationContext.Current = null!
// Продолжение выполняется на ThreadPool
```

**Где есть SynchronizationContext:**
| Среда | SynchronizationContext | После await |
|---|---|---|
| WPF/WinForms | `DispatcherSynchronizationContext` | На UI thread |
| ASP.NET Core (до .NET 8) | `AspNetSynchronizationContext` | На исходном контексте |
| ASP.NET Core (.NET 8+) | `null` | На ThreadPool |
| Console / Library | `null` | На ThreadPool |
| Blazor Server | `RendererSynchronizationContext` | На renderer'е |

---

## ConfigureAwait(false) — когда и зачем

```csharp
// В библиотеках — не захватываем контекст
public async Task<string> LoadDataAsync()
{
    // ConfigureAwait(false) — продолжение на ThreadPool
    // Даже если вызывающий код — на UI thread
    var data = await _httpClient.GetStringAsync(url)
        .ConfigureAwait(false);
    
    // Здесь мы на ThreadPool (или любом другом потоке)
    return ProcessData(data);  // Работает, т.к. не трогаем UI
}

// В .NET Core библиотеках — ConfigureAwait(false) по умолчанию не нужен
// SynchronizationContext в ASP.NET Core всё равно null
public async Task<Data> GetDataAsync()
{
    // В ASP.NET Core — без разницы, т.к. нет контекста
    var result = await _repository.FindAsync(id);
    return result;
}
```

**Правило Senior:**
- **Библиотеки** (не UI): используйте `ConfigureAwait(false)` для публичных методов
- **UI код**: НЕ используйте `ConfigureAwait(false)` если после await нужен UI
- **ASP.NET Core**: не нужно (SynchronizationContext = null)

**Факт:** Начиная с .NET 8, `ConfigureAwait(false)` не даёт преимущества в ASP.NET Core (библиотеки тоже, т.к. нет контекста по умолчанию).

**Факт:** В .NET Framework `ConfigureAwait(false)` мог **предотвратить deadlock**:

```csharp
// .NET Framework — deadlock!
public string GetSync()
{
    return GetAsync().Result;  // UI thread блокируется
}

public async Task<string> GetAsync()
{
    // await пытается вернуться на UI thread (SynchronizationContext)
    // Но UI thread заблокирован .Result — DEADLOCK!
    return await _service.FetchAsync();
}

// Решение: ConfigureAwait(false)
public async Task<string> GetAsync()
{
    return await _service.FetchAsync().ConfigureAwait(false);
    // Продолжение не на UI — .Result не блокирует UI
}
```

---

## async void vs async Task

```csharp
// ❌ async void — нельзя await, исключения в SynchronizationContext
public async void Button_Click(object sender, RoutedEventArgs e)
{
    await LoadDataAsync();  // Если исключение — упадёт в SynchronizationContext
}                           // Приложение может упасть

// ✅ async Task — можно await, исключения в Task
public async Task LoadDataAsync()
{
    await _service.FetchAsync();
}

// Для event handlers — async void нормально, но с try-catch
public async void OnClick(object sender, RoutedEventArgs e)
{
    try
    {
        await LoadDataAsync();
    }
    catch (Exception ex)
    {
        // Логирование, обработка
    }
}
```

**Факт:** `async void` методы нельзя await — исключение либо теряется (если нет try-catch), либо падает в `SynchronizationContext.Post`, вызывая `AppDomain.UnhandledException`.

---

## async lazy initialization

```csharp
// Lazy<Task<T>> — асинхронная инициализация с thread safety
private static Lazy<Task<ExpensiveResource>> _resource =
    new(() => InitializeAsync(), LazyThreadSafetyMode.ExecutionAndPublication);

public static Task<ExpensiveResource> GetResourceAsync() => _resource.Value;

private static async Task<ExpensiveResource> InitializeAsync()
{
    var resource = new ExpensiveResource();
    await resource.ConnectAsync();
    return resource;
}

// Использование
var resource = await GetResourceAsync();
```

---

## ValueTask vs Task

| | Task | ValueTask |
|---|---|---|
| **Тип** | Reference type (class) | Value type (struct) |
| **Аллокация** | Всегда (если не завершён) | Только если async (state machine box) |
| **Многократный await** | ✅ | ❌ (одноразовый) |
| **.Result / .GetAwaiter().GetResult()** | ✅ | ❌ (после первого) |
| **Когда использовать** | Async большинство случаев | Hot path (частый sync completion) |

```csharp
// ValueTask — для методов, часто завершающихся синхронно
public ValueTask<CachedItem> GetAsync(string key)
{
    if (_cache.TryGetValue(key, out var item))
    {
        return new ValueTask<CachedItem>(item);  // Без аллокации
    }
    
    return new ValueTask<CachedItem>(FetchFromDbAsync(key));  // Асинхронно
}

// ❌ Неправильное использование
var vt = GetAsync("key");
await vt;      // OK
await vt;      // ❌ InvalidOperationException!

// ❌ .Result на ValueTask
var result = GetAsync("key").Result;  // Может работать, но не рекомендуется
```

**Факт:** `ValueTask<T>` может быть создан от `Task<T>` (через конструктор) или от значения. **Ограничение:** нельзя await дважды, нельзя параллельно.

**Факт:** `ValueTask` (без generic) — struct, хранит либо `Task`, либо `IValueTaskSource`. `ValueTask<T>` — хранит `Task<T>`, `T`, или `IValueTaskSource<T>`.

---

## IAsyncEnumerable — асинхронные стримы

**Факты:**
- `IAsyncEnumerable<T>` + `await foreach` — потоковая передача данных
- Компилятор использует `IAsyncEnumerator<T>` с ValueTask-based MoveNextAsync
- Поддержка CancellationToken через `WithCancellation()`

```csharp
// Producer — асинхронный генератор данных
public async IAsyncEnumerable<Order> GetOrdersAsync(
    [EnumeratorCancellation] CancellationToken ct = default)
{
    var offset = 0;
    while (true)
    {
        ct.ThrowIfCancellationRequested();
        var batch = await _db.Orders
            .OrderBy(o => o.Id)
            .Skip(offset)
            .Take(100)
            .ToListAsync(ct);
        
        if (batch.Count == 0) yield break;
        
        foreach (var order in batch)
            yield return order;
        
        offset += 100;
    }
}

// Consumer — потоковое чтение
await foreach (var order in GetOrdersAsync().WithCancellation(ct))
{
    await ProcessOrderAsync(order);
    
    // Можно прервать досрочно
    if (order.Priority == Priority.Critical)
        break;  // Итератор освобождается корректно
}

// LINQ-подобные операции
var highValueOrders = GetOrdersAsync()
    .WhereAwait(async o => await IsHighValueAsync(o))
    .SelectAwait(async o => await EnrichOrderAsync(o))
    .Take(50);

await foreach (var order in highValueOrders)
    Console.WriteLine(order.Id);

// ConfigureAwait для IAsyncEnumerable
await foreach (var order in GetOrdersAsync().ConfigureAwait(false))
{
    // Продолжение на ThreadPool (не захватывает контекст)
}
```

**Факт:** `[EnumeratorCancellation]` атрибут заставляет компилятор взять `CancellationToken` из вызова `.WithCancellation()`, а не из параметра метода напрямую.

**Факт:** `IAsyncEnumerable` использует `ValueTask<bool>` для `MoveNextAsync` — минимальные аллокации для синхронного случая (данные уже в буфере).

## System.Threading.Lock (.NET 9+)

```csharp
// .NET 9: новый тип Lock — замена lock(object)
// Преимущества: нельзя случайно использовать неподходящий объект
// Лучшая интеграция с анализаторами, производительность

private readonly Lock _lock = new();

public void DoWork()
{
    using (_lock.EnterScope())
    {
        // Критическая секция
        _counter++;
    }
    // Автоматический выход при Dispose
}

// Эквивалент lock {}
// lock (new Lock()) — ошибка компиляции!
```

## Чек-лист

- [ ] State Machine: struct, состояния (-1 = start, 0..N = await points, -2 = done)
- [ ] SynchronizationContext: DispatcherSynchronizationContext (UI), null (ASP.NET Core)
- [ ] ConfigureAwait(false): библиотеки (сохраняем контекст в UI), не нужно в ASP.NET Core
- [ ] async void: только event handlers, всегда try-catch
- [ ] async lazy: Lazy<Task<T>>
- [ ] ValueTask: hot path, без многократного await
- [ ] IAsyncEnumerable: await foreach, WithCancellation, [EnumeratorCancellation]
- [ ] System.Threading.Lock: .NET 9+ замена lock(object)

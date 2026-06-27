# Задачи для собеседований: Многопоточность и Async — с ответами

---

## Задача 1: Обеденные философы (Dining Philosophers)

**Условие:** N философов сидят за круглым столом. Между каждой парой — одна вилка. Философ ест двумя вилками. Избежать deadlock.

**Что проверяют:** Понимание deadlock, lock ordering, SemaphoreSlim, async coordination.

```csharp
public class DiningPhilosophers
{
    private readonly SemaphoreSlim[] _forks;
    private readonly SemaphoreSlim _table;  // Ограничение: не более N-1 одновременно
    
    public DiningPhilosophers(int count)
    {
        _forks = Enumerable.Range(0, count).Select(_ => new SemaphoreSlim(1, 1)).ToArray();
        _table = new SemaphoreSlim(count - 1, count - 1);
    }
    
    public async Task EatAsync(int philosopherId)
    {
        var leftFork = philosopherId;
        var rightFork = (philosopherId + 1) % _forks.Length;
        
        // Решение 1: Ограничение числа философов (N-1)
        await _table.WaitAsync();
        try
        {
            await _forks[leftFork].WaitAsync();
            await _forks[rightFork].WaitAsync();
            
            // Едим...
            Console.WriteLine($"Philosopher {philosopherId} is eating");
            await Task.Delay(100);
        }
        finally
        {
            _forks[leftFork].Release();
            _forks[rightFork].Release();
            _table.Release();
        }
    }
    
    // Решение 2: Упорядоченный захват (всегда сначала меньший индекс)
    public async Task EatOrderedAsync(int philosopherId)
    {
        var leftFork = philosopherId;
        var rightFork = (philosopherId + 1) % _forks.Length;
        
        var first = Math.Min(leftFork, rightFork);
        var second = Math.Max(leftFork, rightFork);
        
        await _forks[first].WaitAsync();
        await _forks[second].WaitAsync();
        
        try
        {
            Console.WriteLine($"Philosopher {philosopherId} is eating");
            await Task.Delay(100);
        }
        finally
        {
            _forks[first].Release();
            _forks[second].Release();
        }
    }
}
```

**Почему спрашивают:** Классика многопоточности. Проверяет понимание deadlock prevention (lock ordering, resource hierarchy). Senior должен знать оба решения.

---

## Задача 2: Async Producer-Consumer с Backpressure

**Условие:** Producer генерирует данные, Consumer обрабатывает. Producer не должен обгонять Consumer более чем на N элементов (backpressure). Использовать `System.Threading.Channels`.

```csharp
public class DataPipeline<TInput, TOutput>
{
    private readonly Channel<TInput> _channel;
    
    public DataPipeline(int boundedCapacity)
    {
        _channel = Channel.CreateBounded<TInput>(new BoundedChannelOptions(boundedCapacity)
        {
            FullMode = BoundedChannelFullMode.Wait  // Producer ждёт, если буфер полон
        });
    }
    
    public async Task ProduceAsync(IAsyncEnumerable<TInput> source, CancellationToken ct)
    {
        try
        {
            await foreach (var item in source.WithCancellation(ct))
                await _channel.Writer.WriteAsync(item, ct);
            
            _channel.Writer.Complete();
        }
        catch (Exception ex)
        {
            _channel.Writer.Complete(ex);
        }
    }
    
    public async IAsyncEnumerable<TOutput> ConsumeAsync(
        Func<TInput, Task<TOutput>> transform,
        [EnumeratorCancellation] CancellationToken ct = default)
    {
        await foreach (var item in _channel.Reader.ReadAllAsync(ct))
            yield return await transform(item);
    }
}

// Использование:
var pipeline = new DataPipeline<int, string>(boundedCapacity: 100);

var producer = Task.Run(() =>
    pipeline.ProduceAsync(GetNumbersAsync(), CancellationToken.None));

var consumer = Task.Run(async () =>
{
    await foreach (var result in pipeline.ConsumeAsync(async n =>
    {
        await Task.Delay(10);  // Simulate processing
        return (n * 2).ToString();
    }))
    {
        Console.WriteLine(result);
    }
});

await Task.WhenAll(producer, consumer);
```

**Почему спрашивают:** Channel — современная альтернатива BlockingCollection. Проверяет понимание async streams, backpressure, pipeline-архитектуры.

---

## Задача 3: Когда Task.WhenAll падает?

**Условие:** Объяснить поведение `Task.WhenAll` при исключениях в .NET 6+.

```csharp
// Запуск 3 задач, 2 из которых упадут
var task1 = Task.FromResult(1);
var task2 = Task.FromException<int>(new InvalidOperationException("Error 1"));
var task3 = Task.FromException<int>(new ArgumentException("Error 2"));

try
{
    await Task.WhenAll(task1, task2, task3);
}
catch (Exception ex)
{
    Console.WriteLine($"Caught: {ex.GetType().Name}: {ex.Message}");
    // .NET 6+: Caught: InvalidOperationException: Error 1
    // ВТОРОЕ исключение (ArgumentException) — в task3.Exception.InnerExceptions
    
    // До .NET 6: Caught: AggregateException
}

// Как получить ВСЕ исключения:
var allTasks = new[] { task1, task2, task3 };
try
{
    await Task.WhenAll(allTasks);
}
catch
{
    var exceptions = allTasks
        .Where(t => t.IsFaulted)
        .SelectMany(t => t.Exception?.InnerExceptions ?? Enumerable.Empty<Exception>())
        .ToList();
    // exceptions: [InvalidOperationException, ArgumentException]
}

// Task.WhenAll НЕ останавливается при первом исключении — ВСЕ задачи выполняются
// Исключения агрегируются, но await пробрасывает только первое (.NET 6+)
```

**Почему спрашивают:** Проверяет понимание обработки ошибок в TPL, разницу между .NET Framework и .NET 6+, практические паттерны сбора всех исключений.

---

## Задача 4: Параллельная обработка с ограничением конкуренции

**Условие:** Обработать 1000 URL параллельно, но не более 5 одновременных запросов.

```csharp
// Решение 1: SemaphoreSlim (простое и эффективное)
public async Task<List<string>> DownloadWithLimitAsync(string[] urls, int maxConcurrency = 5)
{
    var semaphore = new SemaphoreSlim(maxConcurrency);
    var httpClient = new HttpClient();
    
    var tasks = urls.Select(async url =>
    {
        await semaphore.WaitAsync();
        try
        {
            return await httpClient.GetStringAsync(url);
        }
        finally
        {
            semaphore.Release();
        }
    });
    
    return (await Task.WhenAll(tasks)).ToList();
}

// Решение 2: Parallel.ForEachAsync (.NET 6+)
public async Task<List<string>> DownloadParallelAsync(string[] urls, int maxConcurrency = 5)
{
    var httpClient = new HttpClient();
    var results = new ConcurrentBag<string>();
    
    await Parallel.ForEachAsync(urls,
        new ParallelOptions { MaxDegreeOfParallelism = maxConcurrency },
        async (url, ct) =>
        {
            var content = await httpClient.GetStringAsync(url, ct);
            results.Add(content);
        });
    
    return results.ToList();
}

// Решение 3: TPL Dataflow (для сложных пайплайнов)
public async Task<List<string>> DownloadDataflowAsync(string[] urls, int maxConcurrency = 5)
{
    var httpClient = new HttpClient();
    var block = new TransformBlock<string, string>(
        async url => await httpClient.GetStringAsync(url),
        new ExecutionDataflowBlockOptions { MaxDegreeOfParallelism = maxConcurrency });
    
    var results = new List<string>();
    var collector = new ActionBlock<string>(result => { lock (results) results.Add(result); });
    
    block.LinkTo(collector, new DataflowLinkOptions { PropagateCompletion = true });
    
    foreach (var url in urls)
        await block.SendAsync(url);
    block.Complete();
    await collector.Completion;
    
    return results;
}
```

**Почему спрашивают:** Проверяет знание ТРЁХ подходов к ограничению параллелизма. `Parallel.ForEachAsync` — самый лаконичный (.NET 6+), `SemaphoreSlim` — самый гибкий, Dataflow — для сложных конвейеров.

---

## Задача 5: Конкурентный словарь с TTL (Time-To-Live)

**Условие:** Реализовать кэш, где каждая запись живёт N секунд. После истечения — автоматически удаляется при попытке чтения.

```csharp
public class TtlCache<TKey, TValue> where TKey : notnull
{
    private readonly ConcurrentDictionary<TKey, CacheEntry> _cache = new();
    private readonly TimeSpan _ttl;
    
    public TtlCache(TimeSpan ttl) => _ttl = ttl;
    
    public void Set(TKey key, TValue value)
    {
        _cache[key] = new CacheEntry(value, DateTime.UtcNow.Add(_ttl));
    }
    
    public bool TryGet(TKey key, out TValue? value)
    {
        value = default;
        
        while (_cache.TryGetValue(key, out var entry))
        {
            if (entry.ExpiresAt > DateTime.UtcNow)
            {
                value = entry.Value;
                return true;
            }
            
            // Entry expired — try to remove atomically
            _cache.TryRemove(KeyValuePair.Create(key, entry));
        }
        
        return false;
    }
    
    // Фоновое удаление просроченных записей
    public async Task StartCleanupAsync(CancellationToken ct, TimeSpan interval)
    {
        while (!ct.IsCancellationRequested)
        {
            await Task.Delay(interval, ct);
            var now = DateTime.UtcNow;
            
            foreach (var kvp in _cache)
            {
                if (kvp.Value.ExpiresAt <= now)
                    _cache.TryRemove(kvp);
            }
        }
    }
    
    private record CacheEntry(TValue Value, DateTime ExpiresAt);
}
```

**Почему спрашивают:** Комбинация `ConcurrentDictionary` + бизнес-логика истечения. Проверяет понимание того, что `TryRemove(KeyValuePair)` — атомарное условное удаление (CAS-семантика).

---

## Задача 6: Deadlock-детектор

**Условие:** Реализовать простой deadlock detector: метод `Wait(int resourceId, int timeoutMs)` ждёт ресурс, метод `Release(int resourceId)` освобождает. Если обнаружен deadlock — бросить исключение.

```csharp
public class DeadlockDetector
{
    private readonly Dictionary<int, int> _heldBy = new();  // resourceId → threadId
    private readonly Dictionary<int, HashSet<int>> _waitingFor = new();  // threadId → resourceIds
    private readonly object _lock = new();
    
    public bool Wait(int resourceId, int threadId)
    {
        lock (_lock)
        {
            // Если ресурс занят другим потоком
            if (_heldBy.TryGetValue(resourceId, out var holderId) && holderId != threadId)
            {
                // Добавляем threadId в ожидающие
                if (!_waitingFor.ContainsKey(threadId))
                    _waitingFor[threadId] = new HashSet<int>();
                _waitingFor[threadId].Add(resourceId);
                
                // Проверяем deadlock: есть ли цикл?
                if (DetectCycle(threadId))
                    throw new DeadlockException($"Deadlock detected involving thread {threadId}");
                
                return false;  // Не получили ресурс
            }
            
            _heldBy[resourceId] = threadId;
            _waitingFor.Remove(threadId);
            return true;
        }
    }
    
    private bool DetectCycle(int startThreadId)
    {
        var visited = new HashSet<int>();
        var stack = new Stack<int>();
        stack.Push(startThreadId);
        
        while (stack.Count > 0)
        {
            var threadId = stack.Pop();
            if (!visited.Add(threadId)) continue;
            
            if (!_waitingFor.TryGetValue(threadId, out var resources))
                continue;
            
            foreach (var resourceId in resources)
            {
                if (_heldBy.TryGetValue(resourceId, out var holderId))
                {
                    if (holderId == startThreadId)
                        return true;  // Цикл найден!
                    stack.Push(holderId);
                }
            }
        }
        
        return false;
    }
}
```

**Почему спрашивают:** Проверяет понимание графов, deadlock detection algorithm (cycle detection), lock-based управление ресурсами.

---

## Задача 7: Когда ConfigureAwait(false) важен?

**Условие:** Объяснить, в каком случае этот код вызывает deadlock на .NET Framework, но не на .NET Core.

```csharp
// UI поток (WPF/WinForms)
public string GetData()
{
    return GetDataAsync().Result;  // Deadlock на .NET Framework!
}

public async Task<string> GetDataAsync()
{
    var result = await _httpClient.GetStringAsync("https://api.example.com");
    return result;
}
```

**Ответ:**

```
На .NET Framework:
1. UI поток вызывает GetData() → .Result блокирует UI поток
2. GetDataAsync() начинает HTTP запрос (IOCP, не блокирует)
3. HTTP завершается → await пытается вернуться на UI поток (SynchronizationContext)
4. Но UI поток заблокирован .Result → DEADLOCK!

На .NET Core:
1. ASP.NET Core: SynchronizationContext.Current == null
2. await продолжается на ThreadPool → deadlock'а нет
3. WPF на .NET Core: deadlock ВСЁ ЕЩЁ есть (DispatcherSynchronizationContext)

Решение:
public async Task<string> GetDataAsync()
{
    var result = await _httpClient.GetStringAsync("...")
        .ConfigureAwait(false);  // Не захватываем контекст
    return result;
}
```

**Почему спрашивают:** Проверяет глубокое понимание SynchronizationContext, разницу между .NET Framework и modern .NET, почему `ConfigureAwait(false)` всё ещё важен для UI приложений.

---

## Чек-лист

- [ ] Dining Philosophers: lock ordering, ограничение N-1
- [ ] Producer-Consumer с Channel и backpressure
- [ ] Task.WhenAll исключения: .NET 6+ vs Framework, сбор всех ошибок
- [ ] Ограничение параллелизма: SemaphoreSlim, Parallel.ForEachAsync, Dataflow
- [ ] TTL Cache: ConcurrentDictionary + CAS-удаление
- [ ] Deadlock Detector: cycle detection в графе ресурсов
- [ ] ConfigureAwait(false): когда вызывает deadlock, когда нет
- [ ] Async streams (IAsyncEnumerable) + Channel для pipeline

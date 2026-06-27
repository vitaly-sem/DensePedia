# Parallel и PLINQ — факты + примеры

---

## Parallel class

**Факты:**
- **Блокирует** вызывающий поток до завершения (в отличие от Task)
- Не используйте в ASP.NET request pipeline (блокирует request thread)
- Идеально для: batch processing, desktop UI (на фоновом потоке), консоль/сервисы

### Parallel.For

```csharp
// Простой Parallel.For
Parallel.For(0, 100, i =>
{
    Compute(i);
});

// С параллельными опциями
var options = new ParallelOptions
{
    MaxDegreeOfParallelism = Environment.ProcessorCount,  // Явное ограничение
    CancellationToken = cancellationToken
};

Parallel.For(0, 1000, options, i =>
{
    ct.ThrowIfCancellationRequested();
    var result = ExpensiveCompute(i);
    Interlocked.Add(ref total, result);
});

// ParallelLoopState — Break/Stop
Parallel.For(0, 100_000, (i, state) =>
{
    if (state.ShouldExitCurrentIteration)
        return;
    
    if (FoundResult(i))
    {
        state.Break();  // Останавливает ПЕРЕД i (все < i завершаются)
        // state.Stop();  // Останавливает НЕМЕДЛЕННО (что успело — то успело)
    }
});
```

**Факт:** `Parallel.For` с `MaxDegreeOfParallelism = 1` — выполняет последовательно, но с overhead параллельного планировщика.

### Parallel.ForEach

```csharp
// Параллельная обработка коллекции
Parallel.ForEach(items, new ParallelOptions { MaxDegreeOfParallelism = 4 }, item =>
{
    Process(item);
});

// С Partition — для оптимальной обработки больших массивов
// (каждый partition обрабатывается одним потоком, без синхронизации)
var array = new int[10_000_000];
var rangePartitioner = Partitioner.Create(0, array.Length);

Parallel.ForEach(rangePartitioner, range =>
{
    for (int i = range.Item1; i < range.Item2; i++)
    {
        array[i] = ExpensiveCompute(i);
    }
});
```

**Факт:** `Partitioner.Create` делит массив на range'ы — каждый поток обрабатывает свой range **без синхронизации**. Идеально для больших массивов.

### Parallel.Invoke

```csharp
// Параллельный запуск нескольких методов
Parallel.Invoke(
    () => DoWork1(),
    () => DoWork2(),
    () => DoWork3()
);

// С опциями
Parallel.Invoke(new ParallelOptions { MaxDegreeOfParallelism = 2 },
    () => { Thread.Sleep(1000); Console.WriteLine("1"); },
    () => { Thread.Sleep(500);  Console.WriteLine("2"); },
    () => { Thread.Sleep(200);  Console.WriteLine("3"); });
```

---

## PLINQ (Parallel LINQ)

**Факты:**
- Parallel.ForEach на LINQ стероидах
- Автоматическое распараллеливание операций
- **OrderBy, Take, Skip** — дорогие (слияние результатов)

```csharp
// Базовая PLINQ
var result = data.AsParallel()
    .Where(x => x.IsValid)
    .Select(x => ExpensiveTransform(x))
    .ToArray();  // Обязательный терминальный оператор!

// С ограничением параллелизма
var result = data.AsParallel()
    .WithDegreeOfParallelism(4)
    .WithExecutionMode(ParallelExecutionMode.ForceParallelism)  // Всегда параллельно
    .Where(x => x.IsValid)
    .Select(x => ExpensiveTransform(x))
    .ToArray();

// Сохранение порядка (дороже)
var ordered = data.AsParallel()
    .AsOrdered()  // Результаты в том же порядке, что и source
    .Select(x => ExpensiveCompute(x))
    .ToArray();

// Отмена
var result = data.AsParallel()
    .WithCancellation(ct)
    .Select(x => ExpensiveCompute(x))
    .ToArray();

// Merge — как собирать результаты
var progressive = data.AsParallel()
    .WithMergeOptions(ParallelMergeOptions.NotBuffered)  // Выдавать по готовности
    // AutoBuffered — буфер ~64 элемента
    // FullyBuffered — все до выдачи
    .Select(x => ExpensiveCompute(x));

foreach (var item in progressive)
{
    Console.WriteLine(item);  // Появляется по мере вычисления
}
```

**Когда PLINQ проигрывает обычной LINQ:**

| Сценарий | Почему |
|---|---|
| I/O-bound операции | ThreadPool блокируется на ожидании I/O (используйте async) |
| Мало элементов (< 100) | Overhead планирования > выигрыш |
| Простые операции (арифметика) | Overhead > выигрыш |
| Небезопасные к потокам операции | Race conditions, исключения |

---

## MaxDegreeOfParallelism — сколько потоков?

```csharp
// Environment.ProcessorCount — обычно оптимально для CPU-bound
var cpuCount = Environment.ProcessorCount;  // 8 на 8-ядерном

// Для I/O-bound — больше (но осторожно)
// Для I/O — используйте async, не Parallel

// Для смешанных сценариев — ядра * 2
new ParallelOptions { MaxDegreeOfParallelism = cpuCount * 2 };

// Совсем без ограничения
new ParallelOptions { MaxDegreeOfParallelism = -1 };  // Все доступные потоки
```

**Факт:** `MaxDegreeOfParallelism = -1` (неограничен) может вызвать **resource starvation** и **oversubscription**. Редко нужно.

---

## Parallel vs Task.WhenAll

| | Parallel | Task.WhenAll |
|---|---|---|
| **Блокировка** | Блокирует вызывающий поток | Асинхронный (не блокирует) |
| **Исключения** | AggregateException | Первый exception (или AggregateException) |
| **Использование** | CPU-bound (Parallel) | CPU/I/O-bound (async) |
| **Отмена** | ParallelLoopState | CancellationToken |

```csharp
// Parallel — CPU-bound
Parallel.For(0, images.Length, i =>
{
    images[i] = ApplyFilter(images[i]);  // CPU-bound
});

// Task.WhenAll — I/O-bound
await Task.WhenAll(urls.Select(url =>
    httpClient.GetStringAsync(url)));  // I/O-bound
```

---

## Partitioning — как Parallel делит данные

```csharp
// Range partitioning (массивы, IList) — каждый поток получает range
// Chunk partitioning (IEnumerable) — динамические chunks (размер растёт)

// Кастомный partitioner — для особенностей данных
public class WorkItemPartitioner : Partitioner<WorkItem>
{
    public override bool SupportsDynamicPartitions => true;
    
    public override IEnumerable<WorkItem> GetPartitions(int partitionCount)
    {
        // Распределение по приоритету
        var partitions = Enumerable.Range(0, partitionCount)
            .Select(_ => new List<WorkItem>()).ToArray();
        
        int index = 0;
        foreach (var item in _source.OrderByDescending(x => x.Priority))
        {
            partitions[index % partitionCount].Add(item);
            index++;
        }
        
        return partitions.Select(list => list.AsEnumerable()).ToArray();
    }
}
```

---

## Чек-лист

- [ ] Parallel.For/ForEach/Invoke — блокирует поток, CPU-bound
- [ ] MaxDegreeOfParallelism — ограничение, -1 = без ограничения
- [ ] ParallelLoopState: Break (после i) vs Stop (немедленно)
- [ ] Partitioner.Create — range partitioning для массивов
- [ ] PLINQ: AsParallel, WithDegreeOfParallelism, AsOrdered
- [ ] PLINQ MergeOptions: NotBuffered vs AutoBuffered vs FullyBuffered
- [ ] Parallel vs Task.WhenAll: CPU-bound vs I/O-bound

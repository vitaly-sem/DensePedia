# Concurrent Collections в .NET — факты + примеры

---

## Сравнение коллекций

| Коллекция | Thread-Safe | Lock-Free | Порядок | Bounded |
|---|---|---|---|---|
| `ConcurrentDictionary` | ✅ | ✅ (partial) | Нет | Нет |
| `ConcurrentQueue` | ✅ | ✅ (lock-free, SList) | FIFO | Нет |
| `ConcurrentStack` | ✅ | ✅ (lock-free) | LIFO | Нет |
| `ConcurrentBag` | ✅ | ✅ (lock-free) | Нет (per-thread) | Нет |
| `BlockingCollection` | ✅ (wrapper) | ❌ (использует IProducerConsumerCollection) | Зависит | ✅ |
| `Channel` | ✅ | ✅ (lock-free) | FIFO | ✅ |
| `ImmutableArray/Dictionary` | ✅ (by design) | N/A | Да | Нет |

---

## ConcurrentDictionary

**Факты:**
- Partition-based locking (не одна блокировка на весь dictionary)
- Lock-free для чтения (read можно без блокировки)
- `Count`, `IsEmpty` — lock-free (приблизительные)

```csharp
private readonly ConcurrentDictionary<string, CacheEntry> _cache = new();

// 1. GetOrAdd — атомарно
// Гарантирует однократный вызов factory (если valueFactory простой)
var entry = _cache.GetOrAdd(key, k => ExpensiveCreate(k));

// ⚠️ valueFactory может выполниться несколько раз при contention
// Решение: Lazy<T> внутри
var entry = _cache.GetOrAdd(key, _ => new Lazy<CacheEntry>(() => ExpensiveCreate(key)));

// 2. AddOrUpdate — атомарно
_cache.AddOrUpdate(key,
    addValue: new CacheEntry(data),           // Если ключа нет
    updateValueFactory: (k, old) =>           // Если ключ есть
    {
        old.Update(data);
        return old;
    });

// 3. TryUpdate — CAS для значения
var currentValue = _cache[key];
var newValue = currentValue with { Count = currentValue.Count + 1 };
_cache.TryUpdate(key, newValue, currentValue);  // Обновить, только если не изменилось

// 4. TryRemove
_cache.TryRemove(key, out var removed);

// 5. GetOrAdd с очисткой
_cache.GetOrAdd(key, _ => new CacheEntry())
      .Increment();  // Модификация объекта внутри — безопасно, т.к. ConcurrentDictionary не блокирует объекты
```

**Факт:** `Count` — **не моментальный**. Из-за lock-free операций может быть неточным.

**Факт:** `GetOrAdd` с дорогой `valueFactory` — `valueFactory` может быть вызван несколько раз при конкуренции. Добавьте блокировку внутри factory, если создание ресурса дорогое.

---

## ConcurrentQueue

```csharp
// Lock-free, FIFO, на основе singly-linked list
private readonly ConcurrentQueue<WorkItem> _queue = new();

// Producer
_queue.Enqueue(new WorkItem(data));

// Consumer
while (_queue.TryDequeue(out var item))
{
    Process(item);
}

// Peek (без удаления)
if (_queue.TryPeek(out var next))
    Console.WriteLine($"Next: {next}");

// Очистка
_queue.Clear();  // .NET 5+

// Итерация (snapshot на момент начала)
foreach (var item in _queue)
    Console.WriteLine(item);
```

**Факт:** `TryDequeue` возвращает `false`, если очередь пуста — **не бросает исключение**.

**Факт:** Итерация по `ConcurrentQueue` создаёт **snapshot** — безопасно, но может быть неактуальным в реальном времени.

---

## Channel — современная альтернатива

**Факты:**
- Встроен в .NET Core 3.1+ (`System.Threading.Channels`)
- **Lock-free** (SingleProducerSingleConsumer — полностью)
- **Async-native** (ReadAsync, WriteAsync)
- Поддержка backpressure (BoundedCapacity)
- Гораздо быстрее `BlockingCollection`

```csharp
// Bounded channel — с backpressure
var bounded = Channel.CreateBounded<Order>(new BoundedChannelOptions(100)
{
    FullMode = BoundedChannelFullMode.Wait,           // Producer ждёт
    // DropOldest — удалить старый элемент
    // DropNewest — удалить новый элемент
    // DropWrite — пропустить запись
    SingleWriter = false,   // Множественные продюсеры
    SingleReader = false    // Множественные консьюмеры
});

// Unbounded channel
var unbounded = Channel.CreateUnbounded<int>(new UnboundedChannelOptions
{
    SingleReader = true,
    SingleWriter = true
});

// Producer
await bounded.Writer.WriteAsync(order);
bounded.Writer.TryWrite(order);  // Без ожидания

// Consumer (async)
await foreach (var order in bounded.Reader.ReadAllAsync())
{
    Process(order);
}

// Consumer (один элемент)
var item = await bounded.Reader.ReadAsync();

// Завершение
bounded.Writer.Complete();

// Producer-Consumer pipeline
Channel<Order> channel = Channel.CreateBounded<Order>(100);

// Producer Task
_ = Task.Run(async () =>
{
    foreach (var order in GetOrders())
        await channel.Writer.WriteAsync(order);
    channel.Writer.Complete();
});

// Consumer Task
_ = Task.Run(async () =>
{
    await foreach (var order in channel.Reader.ReadAllAsync())
        await ProcessOrderAsync(order);
});
```

**Бенчмарк (Channel vs BlockingCollection):**

| Сценарий | BlockingCollection | Channel |
|---|---|---|
| 1M items (SPSC) | ~500ms | ~80ms |
| 1M items (MPMC) | ~700ms | ~150ms |
| Memory/op | ~200 bytes | ~80 bytes |

**Факт:** Channel использует **примитивы синхронизации** без блокировки потоков (ManualResetValueTaskSource). BlockingCollection блокирует потоки на `Take()`.

---

## BlockingCollection

```csharp
// BlockingCollection — wrapper вокруг IProducerConsumerCollection
// Может быть Bounded (ограниченный размер)

// Producer
var collection = new BlockingCollection<int>(boundedCapacity: 100);

Task.Run(() =>
{
    for (int i = 0; i < 100; i++)
    {
        collection.Add(i);  // Блокируется, если collection полна
    }
    collection.CompleteAdding();
});

// Consumer (блокирующий)
foreach (var item in collection.GetConsumingEnumerable())
{
    Console.WriteLine(item);  // Блокируется, пока нет данных
}

// Consumer (с CancellationToken)
try
{
    foreach (var item in collection.GetConsumingEnumerable(cancellationToken))
    {
        Process(item);
    }
}
catch (OperationCanceledException) { }
```

---

## Immutable Collections

```csharp
using System.Collections.Immutable;

// Immutable — thread-safe по определению (нельзя изменить)

ImmutableArray<int> arr = ImmutableArray<int>.Empty
    .Add(1)     // Возвращает НОВЫЙ immutable array
    .Add(2)
    .Add(3);

// arr всё ещё пустой! (immutable — не меняется)
// var arr2 = arr.Add(1) — так правильно

ImmutableDictionary<string, int> dict = ImmutableDictionary<string, int>.Empty
    .Add("one", 1)   // Новый dictionary
    .Add("two", 2);

// Для частых изменений — Builder
var builder = dict.ToBuilder();
builder["three"] = 3;
builder.Remove("one");
dict = builder.ToImmutable();  // Заморозка

ImmutableList<int> list = ImmutableList<int>.Empty;
```

**Факт:** Immutable коллекции используют **persistent data structures** — изменения создают новый экземпляр с shared неизменяемой частью. Memory overhead ниже, чем копирование всего.

**Факт:** `ImmutableArray<T>` — struct, не class. Нет аллокации для пустой коллекции.

---

## Чек-лист

- [ ] ConcurrentDictionary: GetOrAdd, AddOrUpdate, TryUpdate
- [ ] ConcurrentQueue: TryDequeue, FIFO
- [ ] Channel: bounded/unbounded, ReadAllAsync, WriteAsync, backpressure
- [ ] BlockingCollection: GetConsumingEnumerable, boundedCapacity
- [ ] Immutable: ImmutableArray, ImmutableDictionary, Builder
- [ ] Когда Channel vs BlockingCollection (Channel — async-native и быстрее)

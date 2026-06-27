# Синхронизация в .NET — факты + примеры

---

## lock (Monitor)

**Факты:**
- `lock(obj)` = `Monitor.Enter(obj)` + `Monitor.Exit(obj)` в try-finally
- `lock(this)` — **антипаттерн** (публичный объект может быть заблокирован извне)
- `lock(typeof(T))` — **антипаттерн** (typeof — публичный объект)
- `lock(string)` — **антипаттерн** (строки интернируются)
- Всегда используйте приватный `object _lock = new()`

```csharp
private readonly object _lock = new();
private int _counter;

public void Increment()
{
    lock (_lock)  // Monitor.Enter
    {
        _counter++;
    }  // Monitor.Exit (даже при исключении)
}

// ⚠️ Не используйте lock в async методах!
public async Task IncrementAsync()
{
    lock (_lock)  // ❌ CS1992 — The 'await' operator cannot be used in a lock body
    {
        await Task.Delay(100);  // Ошибка компиляции
    }
}

// Решение: SemaphoreSlim
private readonly SemaphoreSlim _semaphore = new(1, 1);

public async Task IncrementAsync()
{
    await _semaphore.WaitAsync();
    try
    {
        await Task.Delay(100);
        _counter++;
    }
    finally
    {
        _semaphore.Release();
    }
}
```

**Факт:** `Monitor.Enter` может использовать **spin-wait** перед context switch (адаптивно, для коротких критических секций).

**Факт:** В .NET 9 `Monitor.Lock` — альтернатива `lock` с меньшим overhead для читаемого кода.

---

## SemaphoreSlim — async-совместимый семафор

```csharp
// Ограничение параллелизма: 3 одновременных
private readonly SemaphoreSlim _gate = new(3, 3);

public async Task<Data> CallApiAsync(string url)
{
    await _gate.WaitAsync();  // Асинхронное ожидание (не блокирует поток)
    try
    {
        return await _httpClient.GetFromJsonAsync<Data>(url);
    }
    finally
    {
        _gate.Release();  // Обязательно в finally!
    }
}

// Wait синхронный (блокирует поток)
public void CallApiSync(string url)
{
    _gate.Wait();  // Блокирует текущий поток
    try { /* ... */ }
    finally { _gate.Release(); }
}

// CurrentCount — сколько осталось свободных слотов
if (_gate.CurrentCount > 0)
{
    // Быстрый путь — без ожидания
}
```

**Факт:** `SemaphoreSlim` легче `Semaphore` (который wrapping Win32 объекта). `SemaphoreSlim` — управляемый, быстрее, async-совместимый.

---

## ReaderWriterLockSlim

**Факты:**
- **Множественные читатели** (одновременно)
- **Эксклюзивный писатель** (один, без читателей)
- Идеально для: кэш, конфигурация, редко обновляемые данные

```csharp
private readonly ReaderWriterLockSlim _rwLock = new(
    LockRecursionPolicy.NoRecursion);  // RecursionPolicy.NoRecursion! (быстрее)
private Dictionary<string, CacheItem> _cache = new();

public CacheItem? Get(string key)
{
    _rwLock.EnterReadLock();
    try
    {
        return _cache.TryGetValue(key, out var item) ? item : null;
    }
    finally
    {
        _rwLock.ExitReadLock();
    }
}

public void Set(string key, CacheItem item)
{
    _rwLock.EnterWriteLock();
    try
    {
        _cache[key] = item;
    }
    finally
    {
        _rwLock.ExitWriteLock();
    }
}

// UpgradeableReadLock — обновление с чтения на запись (без гонки)
public CacheItem GetOrAdd(string key, Func<CacheItem> factory)
{
    _rwLock.EnterUpgradeableReadLock();
    try
    {
        if (_cache.TryGetValue(key, out var existing))
            return existing;
        
        _rwLock.EnterWriteLock();
        try
        {
            _cache[key] = factory();
            return _cache[key];
        }
        finally
        {
            _rwLock.ExitWriteLock();
        }
    }
    finally
    {
        _rwLock.ExitUpgradeableReadLock();
    }
}
```

**Факт:** `LockRecursionPolicy.SupportsRecursion` — медленнее, может вызвать deadlock. Используйте `NoRecursion`.

**Факт:** `ReaderWriterLockSlim` в 5-10x быстрее `ReaderWriterLock` (устарел).

---

## Interlocked — атомарные операции

**Факты:**
- Все операции **без lock** (CPU-level atomicity)
- Работает только с `int`, `long`, `double`, `object`, `T` (reference type)
- Для сложных сценариев — `Interlocked.CompareExchange<T>` (lock-free структуры)

```csharp
private int _counter;
private long _total;
private int _flags;

// Инкремент/декремент
Interlocked.Increment(ref _counter);      // _counter++
Interlocked.Decrement(ref _counter);      // _counter--
Interlocked.Add(ref _total, 100);         // _total += 100

// Обмен
var old = Interlocked.Exchange(ref _counter, 42);  // _counter = 42, return old

// CompareExchange — CAS (Compare-And-Swap)
var original = Interlocked.CompareExchange(ref _counter, newValue: 100, comparand: 42);
// Если _counter == 42 → _counter = 100, return 42
// Если _counter != 42 → ничего, return текущее значение

// Lock-free стек (Treiber stack)
public class LockFreeStack<T>
{
    private volatile Node? _head;
    
    public void Push(T item)
    {
        var node = new Node(item);
        do
        {
            node.Next = _head;
        }
        while (Interlocked.CompareExchange(ref _head, node, node.Next) != node.Next);
        // CAS: если _head == node.Next, то _head = node
    }
    
    public bool TryPop(out T? item)
    {
        Node? current;
        do
        {
            current = _head;
            if (current == null) { item = default; return false; }
        }
        while (Interlocked.CompareExchange(ref _head, current.Next, current) != current);
        
        item = current.Value;
        return true;
    }
    
    private class Node(T item) { public readonly T Value = item; public Node? Next; }
}

// Memory Barrier через Interlocked.MemoryBarrier()
Interlocked.MemoryBarrier();  // Process-wide memory barrier (дорого!)
```

**Факт:** `Interlocked.CompareExchange` — основа всех lock-free структур в .NET. Используется внутри `ConcurrentDictionary`, `ConcurrentQueue`.

---

## SpinLock — для экспертов

```csharp
// SpinLock — активное ожидание (CPU spin)
// Только для наносекундных критических секций!
// Риск: waste CPU, thread starvation

private SpinLock _spinLock = new(false);  // false = no owner tracking

public void CriticalSection()
{
    bool lockTaken = false;
    try
    {
        _spinLock.Enter(ref lockTaken);
        // Короткая операция (< 10-20 инструкций)
        _counter++;
    }
    finally
    {
        if (lockTaken) _spinLock.Exit();
    }
}
```

**Факт:** `SpinLock` в 10-100x быстрее `lock` для коротких секций, но в 1000x медленнее для длинных (сжигает CPU).

---

## Barrier — синхронизация группы потоков

```csharp
// Barrier — все потоки ждут друг друга на определённом этапе
var barrier = new Barrier(
    participantCount: 3,  // 3 потока
    postPhaseAction: _ => Console.WriteLine("All threads finished phase"));

// Запуск 3 потоков
for (int i = 0; i < 3; i++)
{
    var threadId = i;
    Task.Run(() =>
    {
        for (int phase = 0; phase < 3; phase++)
        {
            Thread.Sleep(Random.Shared.Next(100, 500));
            Console.WriteLine($"Thread {threadId} phase {phase} done");
            barrier.SignalAndWait();  // Ждём всех
        }
        barrier.RemoveParticipant();  // Убираем себя
    });
}
```

---

## CountdownEvent

```csharp
// CountdownEvent — обратный отсчёт
var countdown = new CountdownEvent(3);  // Ждём 3 сигнала

Task.Run(() => { DoWork1(); countdown.Signal(); });
Task.Run(() => { DoWork2(); countdown.Signal(); });
Task.Run(() => { DoWork3(); countdown.Signal(); });

countdown.Wait();  // Блокирует до 3 сигналов
Console.WriteLine("All done");
```

---

## Чек-лист

- [ ] lock: приватный объект, не async
- [ ] SemaphoreSlim: async WaitAsync, Release в finally
- [ ] ReaderWriterLockSlim: EnterReadLock/EnterWriteLock, UpgradeableReadLock
- [ ] Interlocked: Increment, Add, Exchange, CompareExchange (CAS)
- [ ] SpinLock: только для коротких секций, false (no tracking)
- [ ] Barrier: SignalAndWait, participant count
- [ ] CountdownEvent: Signal(), Wait()
- [ ] lock-free: Treiber stack через CompareExchange

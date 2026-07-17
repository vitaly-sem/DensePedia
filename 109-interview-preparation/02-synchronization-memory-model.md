# Синхронизация и модель памяти .NET

---

## Примитивы синхронизации — иерархия

```
Уровень абстракции (высокий → низкий):

SemaphoreSlim / AsyncLock    ← async-совместимые
    ↓
lock (Monitor)               ← основной примитив
    ↓
ReaderWriterLockSlim         ← read-heavy сценарии
    ↓
Mutex                        ← межпроцессный
    ↓
ManualResetEvent / AutoResetEvent  ← сигнализация
    ↓
Interlocked (CAS)            ← атомарные операции
    ↓
volatile / MemoryBarrier     ← модель памяти
```

---

## Monitor (lock) — как работает

**Факты:**
- `lock(obj)` компилируется в `Monitor.Enter(obj, ref lockTaken)` + `try/finally { Monitor.Exit(obj) }`
- Каждый объект в .NET имеет **sync block** (отрицательный индекс в sync block table)
- При отсутствии конкуренции — **thin lock** (хранится в заголовке объекта)
- При конкуренции — **fat lock** (запись в sync block table)
- `lock` — **реентерабельный**: тот же поток может войти повторно

```csharp
// Компилятор превращает:
lock (_syncRoot)
{
    DoWork();
}

// В:
bool lockTaken = false;
try
{
    Monitor.Enter(_syncRoot, ref lockTaken);
    DoWork();
}
finally
{
    if (lockTaken) Monitor.Exit(_syncRoot);
}
```

### Проблемы lock

```
Q: Почему нельзя lock(this) или lock(typeof(MyClass))?
A: 
  - lock(this): внешний код может захватить тот же объект → deadlock
  - lock(typeof(MyClass)): тип доступен глобально → deadlock
  
  Всегда используйте private readonly object _syncRoot = new();
```

```
Q: Что такое deadlock на lock?
A: Два потока захватывают два lock'а в разном порядке:

   Thread A: lock(a) → lock(b)
   Thread B: lock(b) → lock(a)
   
   A ждёт B освободить b, B ждёт A освободить a → deadlock.

   Решение: всегда захватывать lock'и в одном порядке.
```

### Monitor vs lock

```csharp
// Monitor даёт больше контроля:
if (Monitor.TryEnter(_syncRoot, TimeSpan.FromSeconds(1)))
{
    try { DoWork(); }
    finally { Monitor.Exit(_syncRoot); }
}

// Pulse/Wait — сигнализация (condition variable)
lock (_syncRoot)
{
    while (!_ready)
        Monitor.Wait(_syncRoot);   // Освобождает lock и ждёт Pulse
    Consume();
}

// В другом потоке:
lock (_syncRoot)
{
    _ready = true;
    Monitor.Pulse(_syncRoot);  // Будит ОДИН ожидающий поток
}
```

---

## SemaphoreSlim — ограничение параллелизма

**Факты:**
- `SemaphoreSlim` — легковесный, async-совместимый
- `Semaphore` — системный, работает между процессами
- Считает "слоты": `Wait()` уменьшает, `Release()` увеличивает
- `WaitAsync()` — асинхронное ожидание (не блокирует поток!)

```csharp
// Ограничение параллелизма: не более 10 одновременных запросов
var semaphore = new SemaphoreSlim(10);

async Task ProcessAsync(Item item)
{
    await semaphore.WaitAsync();
    try
    {
        await CallExternalApiAsync(item);
    }
    finally
    {
        semaphore.Release();
    }
}
```

```
Q: Чем SemaphoreSlim лучше Parallel.ForEach с MaxDegreeOfParallelism?
A: SemaphoreSlim можно использовать асинхронно (WaitAsync), 
   не блокируя потоки ThreadPool. Parallel.ForEach блокирует
   потоки, что приводит к ThreadPool starvation при async-нагрузке.
```

---

## ReaderWriterLockSlim — read-heavy оптимизация

**Факты:**
- Множество потоков могут читать одновременно
- Только один поток может писать
- При записи блокируются все читатели
- Поддерживает рекурсивное владение (настраивается)
- `EnterUpgradeableReadLock` — чтение с возможностью апгрейда до записи

```csharp
var rwLock = new ReaderWriterLockSlim();

// Много читателей — параллельно
rwLock.EnterReadLock();
try { return _cache.TryGetValue(key, out var v) ? v : LoadFromDb(key); }
finally { rwLock.ExitReadLock(); }

// Один писатель — эксклюзивно
rwLock.EnterWriteLock();
try { _cache[key] = newValue; }
finally { rwLock.ExitWriteLock(); }
```

---

## Interlocked — атомарные операции без блокировок

**Факты:**
- Используют **CAS** (Compare-And-Swap) на уровне процессора (`lock cmpxchg` на x86)
- Не блокируют потоки — истинно lock-free
- Работают только с `int`, `long`, `float`, `double`, `object` (ссылки)
- `Interlocked.CompareExchange` — основа всех lock-free алгоритмов

```csharp
// Атомарный инкремент (lock-free)
int count = 0;
Interlocked.Increment(ref count);  // Атомарно: count++

// CAS: если current == 0, заменить на 1
int original = Interlocked.CompareExchange(ref count, 1, 0);
// original == предыдущее значение count

// Атомарное чтение/запись (для 64-бит на 32-бит системах)
long val = Interlocked.Read(ref _longField);
Interlocked.Exchange(ref _field, newValue);
```

### Lock-free счётчик (полный пример для собеседования)

```csharp
public class LockFreeCounter
{
    private int _value;

    public int Increment() => Interlocked.Increment(ref _value);
    public int Decrement() => Interlocked.Decrement(ref _value);
    public int Read() => Interlocked.CompareExchange(ref _value, 0, 0);

    // CAS + спин-ожидание для условного обновления
    public void Add(int delta)
    {
        int original, updated;
        do
        {
            original = _value;
            updated = original + delta;
        }
        while (Interlocked.CompareExchange(ref _value, updated, original) != original);
    }
}
```

### Lock-free стек (Treiber stack)

```csharp
public class LockFreeStack<T>
{
    private Node _head; // volatile не нужен — Interlocked даёт барьеры

    private class Node
    {
        public T Value;
        public Node Next;
        public Node(T value) => Value = value;
    }

    public void Push(T value)
    {
        var node = new Node(value);
        Node original;
        do
        {
            original = _head;
            node.Next = original;
        }
        while (Interlocked.CompareExchange(ref _head, node, original) != original);
    }

    public bool TryPop(out T value)
    {
        Node original;
        do
        {
            original = _head;
            if (original == null) { value = default; return false; }
        }
        while (Interlocked.CompareExchange(ref _head, original.Next, original) != original);

        value = original.Value;
        return true;
    }
}
```

---

## Модель памяти .NET

### Ключевые концепции

```
Проблема: Компилятор, JIT и процессор переупорядочивают инструкции.
         Поток 2 может увидеть запись потоком 1 НЕ в том порядке.

Пример (Store/Load переупорядочивание):

int a = 0, b = 0;
// Thread 1:                // Thread 2:
a = 1;                      if (b == 1)
b = 1;                          Console.WriteLine(a); // Может быть 0!
```

### volatile

**Факты:**
- Запрещает кеширование в регистрах процессора — каждое чтение из памяти
- Запрещает переупорядочивание: все записи до volatile-записи видны ПОСЛЕ неё
- В C# `volatile` для полей генерирует `Volatile.Read`/`Volatile.Write` с acquire/release семантикой

```csharp
// Без volatile: поток может бесконечно читать закешированное значение
private volatile bool _isCompleted;

void Thread1() { _isCompleted = true; }  // Release semantics
void Thread2() { while (!_isCompleted) ; }  // Acquire semantics — увидит обновление
```

### Memory Barriers (MemoryBarrier)

```csharp
// Полный барьер (full fence): все операции до барьера видны до операций после
Thread.MemoryBarrier();  // mfence на x86

// На практике используйте Interlocked или volatile вместо явных барьеров
// Interlocked.CompareExchange — неявный полный барьер (full fence)
// Volatile.Read — acquire (load + half-fence)
// Volatile.Write — release (half-fence + store)
```

### ABA Problem (lock-free)

```
Проблема ABA:
1. Поток 1 читает значение A из ячейки
2. Поток 2 меняет A → B
3. Поток 2 меняет B → A
4. Поток 1 делает CAS: видит A → считает что ничего не изменилось → ОШИБКА!

Решение: tagged pointer или счётчик версий.

В .NET: Interlocked.CompareExchange с object проверяет ссылки —
         GC не даст переиспользовать ту же ссылку для другого объекта,
         поэтому ABA маловероятна для ссылочных типов.
```

### ThreadLocal\<T\> и AsyncLocal\<T\>

```csharp
// ThreadLocal — значение привязано к потоку ОС
var threadLocal = new ThreadLocal<int>(() => Thread.CurrentThread.ManagedThreadId);

// AsyncLocal — значение "перетекает" через async/await (ExecutionContext)
var asyncLocal = new AsyncLocal<int>();
asyncLocal.Value = 42;           // Установлено в этом контексте
await Task.Delay(100);
Console.WriteLine(asyncLocal.Value);  // Всё ещё 42!
```

```
Q: Когда использовать ThreadLocal vs AsyncLocal?
A: ThreadLocal — для данных, привязанных к физическому потоку 
   (например, пул объектов на поток). AsyncLocal — для данных,
   которые должны следовать за логическим контекстом через async/await
   (например, CorrelationId, токен отмены).
```

---

## Типичные проблемы многопоточности — сводка

| Проблема | Описание | Решение |
|----------|----------|---------|
| **Race Condition** | Результат зависит от порядка выполнения потоков | lock, Interlocked, ConcurrentCollection |
| **Deadlock** | Два потока ждут друг друга | Порядок lock'ов, таймауты, TryEnter |
| **Livelock** | Потоки активны, но не продвигаются | Random backoff, приоритеты |
| **Starvation** | Поток не получает доступ к ресурсу | Fair lock, SemaphoreSlim с очередью |
| **False Sharing** | Два потока пишут в разные переменные на одной cache line | Padding (`[StructLayout(LayoutKind.Sequential, Size=64)]`) |
| **ThreadPool Starvation** | Все потоки ThreadPool заблокированы синхронным ожиданием | async/await вместо sync-wait |
| **ABA Problem** | CAS видит "то же" значение после A→B→A | Tagged pointer, счётчик версий |

---

## Чек-лист для собеседования: синхронизация

- [ ] Объяснить разницу между lock, Monitor, Mutex, SemaphoreSlim
- [ ] Что такое CAS и как работает Interlocked.CompareExchange?
- [ ] Реализовать lock-free стек или счётчик
- [ ] Объяснить volatile: что именно он запрещает?
- [ ] Что такое ABA problem и как её решить?
- [ ] Чем ThreadLocal отличается от AsyncLocal?
- [ ] Что такое false sharing и как его избежать?
- [ ] Какие операции являются атомарными в .NET? (Чтение/запись 32-бит, ссылки; Interlocked для 64-бит)

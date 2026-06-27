# Memory Model и низкоуровневая синхронизация

---

## .NET Memory Model — основы

**Факты:**
- **Memory model** определяет, как потоки видят изменения друг друга
- Компилятор, JIT и CPU могут **переупорядочивать** инструкции для оптимизации
- Без барьеров — нет гарантий видимости изменений между потоками

**Пример — проблема:**

```csharp
// Thread 1
_data = 42;
_ready = true;

// Thread 2
if (_ready)
    Console.WriteLine(_data);  // Может написать 0!
```

**Почему:**
1. CPU может переупорядочить: `_ready = true` выполнится до `_data = 42`
2. Кэш CPU: Thread 2 не видит изменения Thread 1 (каждый на своём ядре)
3. JIT может удалить `_data = 42` как "бесполезное" (если не видит чтения)

---

## volatile

**Факты:**
- `volatile` — **запрещает переупорядочивание** чтений/записей
- Все записи до `volatile write` видны после `volatile read`
- **Не делает операции атомарными** (инкремент остаётся неатомарным)
- Работает с: `bool`, `byte`, `short`, `int`, `float`, `char`, `IntPtr`, `UIntPtr`
- Не работает с: `long`, `double` (на 32-bit), `decimal`, `struct`

```csharp
private volatile bool _isRunning;

public void Stop()
{
    _isRunning = false;  // volatile write → release semantics
}

public void Run()
{
    while (_isRunning)   // volatile read → acquire semantics
    {
        // Работает
    }
    Console.WriteLine("Stopped");
}

// volatile не делает:
private volatile int _counter;

public void Increment()
{
    _counter++;  // ❌ НЕ атомарно!
    // read-modify-write — три операции, не одна
}

// Решение: Interlocked.Increment
Interlocked.Increment(ref _counter);
```

**volatile semantics:**
- **Volatile read** (acquire) — все чтения/записи ПОСЛЕ не перемещаются до него
- **Volatile write** (release) — все чтения/записи ДО не перемещаются после него

---

## MemoryBarrier — явные барьеры

```csharp
// Полный барьер — все чтения/записи до, все после
Thread.MemoryBarrier();

// Барьер с Release/Acquire семантикой (более тонкий контроль)
var data = Volatile.Read(ref _field);        // acquire semantics
Volatile.Write(ref _field, value);           // release semantics

// Пример — ручная реализация volatile через барьеры
private int _value;

public int Value
{
    get
    {
        Thread.MemoryBarrier();   // acquire
        return _value;
    }
    set
    {
        _value = value;
        Thread.MemoryBarrier();   // release
    }
}
```

**Факт:** `volatile` — самый лёгкий барьер (конкретные CPU инструкции: `mfence`/`lock`). `Thread.MemoryBarrier()` — полный барьер (дороже).

**Факт:** x86/x64 имеет **strong memory model** (acquire/release по умолчанию для обычных записей). ARM — **weak model** (больше переупорядочивания, нужны барьеры чаще).

---

## Double-Checked Locking (DCL)

**Факты:**
- **Без volatile — DCL сломан!** (из-за переупорядочивания)
- Используйте `Lazy<T>` вместо ручного DCL

```csharp
// ✅ Правильный Lazy<T>
private static readonly Lazy<ExpensiveResource> _resource =
    new(() => new ExpensiveResource(), LazyThreadSafetyMode.ExecutionAndPublication);

public static ExpensiveResource Instance => _resource.Value;

// Или:
private static readonly Lazy<ExpensiveResource> _resource2 =
    new(() => new ExpensiveResource(), isThreadSafe: true);

// ⚠️ Ручной DCL (только для понимания)
private static volatile ExpensiveResource? _instance;
private static readonly object _lock = new();

public static ExpensiveResource Instance
{
    get
    {
        if (_instance == null)           // 1-я проверка (без блокировки)
        {
            lock (_lock)
            {
                if (_instance == null)   // 2-я проверка (с блокировкой)
                {
                    var temp = new ExpensiveResource();  // Создаём локально
                    Thread.MemoryBarrier();              // Release barrier
                    _instance = temp;                    // Публикуем
                }
            }
        }
        return _instance;
    }
}
```

**Факт:** Без `volatile` в DCL — один поток может видеть `_instance != null`, но объект ещё не полностью сконструирован (переупорядочивание конструктора).

**Факт:** `Lazy<T>` с `ExecutionAndPublication` — один поток инициализирует, остальные ждут. `PublicationOnly` — несколько потоков могут инициализировать, winner'а публикует.

---

## Lock-Free структуры — опасности

### ABA Problem

```csharp
// ABA проблема — классическая lock-free ловушка
// Thread 1: читает _head → Node A
// Thread 2: Pop A, Push B, Push A (тот же адрес!)
// Thread 1: CAS(_head, A, C) — успешно, но _head изменился!
// Решение: TaggedReference (Double-Word CAS)

// В .NET — нет встроенного DWCAS.
// Решение: AtomicReference<T> или Interlocked.CompareExchange с TaggedReference

// TaggedReference для ABA
internal record struct TaggedReference<T>(T Reference, int Tag) where T : class;
```

### Hazard Pointers / Epoch-based Reclamation

**Факт:** Lock-free структуры требуют **safe memory reclamation** — нельзя удалить объект, пока другой поток может его читать. Решения:

```csharp
// .NET Runtime использует Epoch-based reclamation для своих lock-free структур
// GC — преимущество .NET: нельзя получить dangling pointer (managed references)
// Но ABA остаётся проблемой
```

---

## Interlocked — барьеры внутри

```csharp
// Все Interlocked.* операции содержат полный барьер памяти
// Atomicity + Visibility

private long _value;

public void Add(long delta)
{
    Interlocked.Add(ref _value, delta);
    // Чтение _value другим потоком — гарантированно увидит изменение
}

public long Read()
{
    return Interlocked.Read(ref _value);  // Атомарное чтение long (на 32-bit)
}
```

---

## Оверхед примитивов синхронизации (от fast к slow)

| Примитив | Относительная стоимость | Описание |
|---|---|---|
| `volatile read` | 1x | Acquire barrier |
| `Interlocked.Increment` | ~3x | Locked CPU instruction |
| `lock` (Monitor) | ~30x | Context switch при contention |
| `SemaphoreSlim.WaitAsync` | ~50x | Async + context switch |
| `Thread.MemoryBarrier()` | ~5x | Full barrier |
| `Thread.Sleep(0)` | ~100x | Yield to scheduler |

**Факт:** При отсутствии contention, `lock` (Monitor.Enter) — **очень быстрый** (spin-wait перед context switch). Контекстный переключается только при реальной конкуренции.

**Факт:** `Interlocked.CompareExchange` в цикле (lock-free) может быть быстрее `lock` только при низкой конкуренции. При высокой — `lock` предпочтительнее.

---

## Чек-лист

- [ ] Memory Model: переупорядочивание, видимость, кэши CPU
- [ ] volatile: acquire/release semantics, не атомарность
- [ ] MemoryBarrier: полный/acquire/release
- [ ] Double-Checked Locking: volatile обязателен, Lazy<T> проще
- [ ] ABA problem: TaggedReference
- [ ] Hazard Pointers: GC в .NET даёт преимущество
- [ ] Overhead: volatile < Interlocked < lock < SemaphoreSlim

# GC Internals — как работает сборщик мусора

---

## Generational Hypothesis

**Факты:**
- **Weak generational hypothesis:** большинство объектов умирают молодыми
- **Strong generational hypothesis:** старые объекты редко ссылаются на молодые
- На этом построена архитектура .NET GC

```
Allocation → Gen 0 (быстро, ~1ms GC)
    ↓ если пережил Gen 0 collection
Gen 1 (промежуточная, ~10ms GC)
    ↓ если пережил Gen 1 collection
Gen 2 (Full GC, ~100ms-1s, stop-the-world)
```

**Факт:** Все новые объекты попадают в **Gen 0**. Размер Gen 0 настраивается dynamically на основе allocation rate.

**Факт:** Gen 2 collection — самая дорогая, так как GC должен просканировать **все живые объекты** всех поколений.

---

## Три фазы GC

### 1. Mark Phase (Маркировка)

```
Manager: все объекты из roots (стек, статики, CPU registers) — живые
BFS/DFS: обход графа объектов от корней
- Объекты, до которых GC дошёл → живые
- Остальные → мёртвые (can be collected)
```

```csharp
// Roots = корни GC:
// 1. Stack variables (локальные переменные потоков)
// 2. Static fields
// 3. GC handles (GCHandle, weak references)
// 4. CPU registers (reference types in registers)
// 5. Finalization queue
// 6. Interop (COM wrappers)
```

**Факт:** Mark phase использует **работу потоков** (workstation: один поток, server: один поток на ядро).

**Факт:** Большое количество живых объектов в Gen 2 → долгая маркировка (scan всех объектов).

### 2. Relocate Phase (Перемещение)

```
- GC вычисляет новые адреса для живых объектов
- Запись в SyncBlock: старый адрес → новый адрес
- Компактизация: живые объекты сдвигаются в начало сегмента
- LOH не компактируется по умолчанию (можно принудительно)
```

### 3. Compact Phase (Сжатие)

```
- Физическое перемещение объектов на новые адреса
- Обновление всех ссылок на объекты (через SyncBlock)
- После: память дефрагментирована, все живые объекты последовательно
```

**Факт:** Compact phase — самая тяжёлая, так как нужно перемещать **реальные объекты** в памяти и обновлять все ссылки (стек, регистры).

---

## GC Segments

**Факты:**
- **SOH (Small Object Heap)** — объекты < 85 000 bytes. Gen 0, 1, 2.
- **LOH (Large Object Heap)** — объекты ≥ 85 000 bytes. Gen 2 только.
- **POH (Pinned Object Heap)** — pinned объекты (.NET 5+). Gen 2.
- Каждый сегмент — **непрерывный блок памяти** (commit/reserve)

```csharp
// Размер сегмента:
// Workstation GC: 16 MB (SOH) + 16 MB (LOH)
// Server GC:      ~200 MB на ядро (SOH) + ~200 MB (LOH)

// Проверка сегментов через GC.GetGCMemoryInfo()
var info = GC.GetGCMemoryInfo();
Console.WriteLine($"Total available memory: {info.TotalAvailableMemoryBytes / 1024 / 1024} MB");
Console.WriteLine($"Heap size: {info.HeapSizeBytes / 1024 / 1024} MB");
Console.WriteLine($"Fragmented bytes: {info.FragmentedBytes}");
```

**Факт:** Когда сегмент заполнен, GC запрашивает новый сегмент у ОС. Старый сегмент освобождается, если все объекты в нём мертвы.

---

## Finalization

**Факты:**
- Объекты с `~Destructor()` или переопределённым `Finalize()` → **finalizable**
- Финализация **удваивает** время жизни объекта (сначала Survive, потом Finalize, потом Free)
- Объект переживает минимум 1 GC (перемещается в Gen 1/LOH)
- `GC.ReRegisterForFinalize` — повторная финализация (избегать)

```csharp
public class ManagedResource : IDisposable
{
    private IntPtr _nativePtr;  // Native resource
    
    ~ManagedResource()  // Finalizer
    {
        // Вызывается на отдельном потоке Finalizer
        // GC не гарантирует порядок финализации!
        CleanupNative(_nativePtr);
    }
    
    public void Dispose()
    {
        CleanupNative(_nativePtr);
        GC.SuppressFinalize(this);  // Отменяем вызов Finalizer
    }
}
```

**Жизненный цикл finalizable объекта:**

```
1. Выделение памяти → помечается как finalizable
2. Gen 0 GC: объект жив → перемещается в finalization queue (freachable)
3. Finalizer thread: вызывает Finalize() → удаляет из queue
4. Следующий GC: объект уже не в queue → может быть собран
```

**Факт:** Если Finalizer завис (deadlock), память **утекает** — объекты не освобождаются.

**Факт:** `GC.WaitForPendingFinalizers()` — блокирует поток, пока Finalizer thread не обработает очередь.

---

## GC Handles

**Факты:**
- `GCHandle` — управляемая ссылка, которую GC учитывает
- **Normal** — сильная ссылка (как static field)
- **Weak** — слабая ссылка (GC может собрать)
- **Pinned** — объект не перемещается (для P/Invoke)
- **WeakTrackResurrection** — слабая ссылка, учитывает финализацию

```csharp
// GCHandle types
public enum GCHandleType
{
    Weak = 0,           // GC может собрать объект
    WeakTrackResurrection = 1,  // Даже если объект в finalization queue
    Normal = 2,         // Strong reference
    Pinned = 3          // Strong + not movable
}

// Пример — Weak Event через GCHandle
public class WeakEventHandler<T> where T : class
{
    private readonly GCHandle _targetHandle;  // Weak reference
    private readonly Action<T> _handler;
    
    public WeakEventHandler(T target, Action<T> handler)
    {
        _targetHandle = GCHandle.Alloc(target, GCHandleType.Weak);
        _handler = handler;
    }
    
    public void Invoke()
    {
        if (_targetHandle.Target is T target)
            _handler(target);
    }
    
    public void Free() => _targetHandle.Free();
}
```

**Факт:** `GCHandle.Free()` обязателен для Pinned и Normal. Weak — GC сам обработает.

---

## GC Notifications

```csharp
// GC Notification — мониторинг приближающегося Full GC
// Только для Server GC

public class GcMonitor
{
    private readonly Thread _monitorThread;
    
    public GcMonitor()
    {
        GC.RegisterForFullGCNotification(10, 10);  // threshold, generation
        _monitorThread = new Thread(MonitorGc);
        _monitorThread.Start();
    }
    
    private void MonitorGc()
    {
        while (true)
        {
            var status = GC.WaitForFullGCApproach(TimeSpan.FromMilliseconds(500));
            if (status == GCNotificationStatus.Succeeded)
            {
                Log.Warning("Full GC approaching");
                ReduceMemoryPressure();  // Сбросить кэш
                
                GC.WaitForFullGCComplete();
                Log.Information("Full GC completed");
            }
        }
    }
}
```

---

## GC.GetGCMemoryInfo() — детальная диагностика

```csharp
// .NET 7+ — полная картина памяти
var info = GC.GetGCMemoryInfo();

Console.WriteLine($"High memory load threshold: {info.HighMemoryLoadThresholdBytes}");
Console.WriteLine($"Total available memory:     {info.TotalAvailableMemoryBytes}");
Console.WriteLine($"Memory info:                {info.MemoryInfo}");
Console.WriteLine($"Finalization pending:       {info.FinalizationPendingCount}");
Console.WriteLine($"Compacted:                  {info.Compacted}");
Console.WriteLine($"Concurrent:                 {info.Concurrent}");
Console.WriteLine($"Generation:                 {info.Generation}");
Console.WriteLine($"Pause duration:             {info.PauseDurationPercentage}%");
Console.WriteLine($"POH size:                   {info.PinnedObjectHeapSizeBytes}");
Console.WriteLine($"Fragmented:                 {info.FragmentedBytes}");
Console.WriteLine($"Heap size:                  {info.HeapSizeBytes}");
```

---

## Чек-лист

- [ ] Generational hypothesis: молодые умирают быстро
- [ ] 3 фазы: Mark → Relocate → Compact
- [ ] SOH (< 85KB) vs LOH (≥ 85KB) vs POH (pinned)
- [ ] Finalization: freachable queue, Finalizer thread, ~2x GC cycles
- [ ] GCHandle: Normal, Weak, Pinned, WeakTrackResurrection
- [ ] GC.GetGCMemoryInfo() — диагностика
- [ ] GC.RegisterForFullGCNotification — pre-GC callback

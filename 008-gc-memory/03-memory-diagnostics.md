# Диагностика памяти и дампы

---

## dotnet-dump — сбор и анализ дампов

**Факты:**
- **dotnet-dump** — кроссплатформенный сбор дампов (Linux, Windows, macOS)
- Не требует Debugging Diagnostic Tool (Windows)
- Анализ через SOS (Son of Strike)

```bash
# Установка
dotnet tool install --global dotnet-dump

# Сбор дампа процесса
dotnet dump collect --process-id 1234

# Сбор дампа с типом
dotnet dump collect -p 1234 -t full       # Полный дамп
dotnet dump collect -p 1234 -t mini       # Минимальный
dotnet dump collect -p 1234 -t heap       # Только heap

# Анализ дампа (интерактивно)
dotnet dump analyze dump_20240315.dmp
```

### Команды SOS в dotnet-dump analyze

```
# Анализ дампа в dotnet dump analyze:
> help                          # Список команд

> print-exceptions              # Все исключения в дампе
> clrstack                      # Stack trace
> clrthreads                    # Все managed threads
> threads                       # Все потоки (native + managed)

> dumpheap -stat                # Статистика heap по типам
> dumpheap -type string         # Все строки в heap
> dumpheap -min 85000           # Все LOH объекты (> 85KB)

> gcroot 0x12345678             # Root для объекта (кто держит ссылку)

> dumpobj 0x12345678            # Информация об объекте
> do 0x12345678                 # Сокращение

> dumpstack -ee                 # Managed stack (exception exploration)
> eeheap -gc                    # GC heap details (segments)
> syncblk                       # SyncBlock table (monitor locks)
> finalizequeue                 # Objects pending finalization
```

**Факт:** `dumpheap -stat` — первая команда при анализе утечки. Показывает, какие типы потребляют больше всего памяти.

**Факт:** `gcroot` — поиск, кто держит ссылку на объект (почему GC не собирает).

---

## Процесс расследования утечки памяти

```csharp
// Шаг 1: Обнаружение — мониторинг
var memoryBefore = GC.GetTotalMemory(false);
// ... работа ...
var memoryAfter = GC.GetTotalMemory(false);
var growth = memoryAfter - memoryBefore;

if (growth > 100 * 1024 * 1024)  // > 100 MB growth
    Log.Warning($"Memory growth detected: {growth / 1024 / 1024} MB");

// Шаг 2: dotnet-counters
// dotnet counters monitor --process-id 1234

// Шаг 3: Сбор дампа
// dotnet dump collect --process-id 1234

// Шаг 4: Анализ в дампе
// > dumpheap -stat
// > gcroot <адрес_подозрительного_объекта>
```

**Типичные причины утечек:**

| Причина | SOS команда | Решение |
|---|---|---|
| Static events | `gcroot` на subscriber | WeakEventManager |
| Timer references | `dumpheap -type Timer` | Dispose таймеров |
| Large cache | `dumpheap -stat` → `dumpheap -type Dictionary` | MemoryCache, eviction |
| P/Invoke handles | `dumpheap -type SafeHandle` | Dispose wrappers |
| ThreadPool | `threads` → `clrthreads` | ThreadPool starvation |
| Finalization queue | `finalizequeue` | Реализовать IDisposable |

---

## PerfView — ETW анализ (Windows)

**Факты:**
- **PerfView** — мощный ETW-анализатор от Microsoft
- Анализ: GC events, CPU samples, JIT, exceptions, contention
- **Windows-only** (Linux: dotnet-trace + SpeedScope)

```bash
# PerfView commands (XPerf-style):
PerfView.exe collect -GCOnly -DataFile:gc.etl -Process:MyApp

# Collect with CPU sampling + GC
PerfView.exe collect -GC:WithSnapshot -DataFile:full.etl
```

**Что искать в PerfView:**

```
GC Stats → GC Heap Alloc (MB/sec) → High allocation rate
GC Stats → GC Pause (%) → > 15-20% problem
GC Stats → Gen 2 GC Count → Частые Full GC
Heap Snapshot → % Gen2 Fragmentation → LOH issues
```

---

## ETW Events — программный мониторинг

```csharp
// Подписка на GC Events через EventListener
public class GcEventListener : EventListener
{
    protected override void OnEventSourceCreated(EventSource eventSource)
    {
        if (eventSource.Name == "Microsoft-Windows-DotNETRuntime")
        {
            // GC events (0x0001) + GC Heap (0x0010) 
            EnableEvents(eventSource, EventLevel.Verbose, (EventKeywords)0x0011);
        }
    }
    
    protected override void OnEventWritten(EventWrittenEventArgs eventData)
    {
        switch (eventData.EventName)
        {
            case "GCStart":
                var generation = (int)eventData.Payload[0];
                var reason = (GCR reason)eventData.Payload[2];
                break;
                
            case "GCEnd":
                break;
                
            case "GCHeapStats":
                var gen0Size = (long)eventData.Payload[0];
                var gen1Size = (long)eventData.Payload[1];
                var gen2Size = (long)eventData.Payload[2];
                var lohSize = (long)eventData.Payload[3];
                break;
        }
    }
}

// Или через EventCounters (проще)
public class GcMonitor
{
    private readonly Dictionary<string, double> _counters = new();
    
    public GcMonitor()
    {
        var source = new EventSource("System.Runtime");
        var handler = new GcEventListener(source);
    }
}
```

---

## dotnet-gcdump — heap snapshot

```bash
# Установка
dotnet tool install --global dotnet-gcdump

# Сбор heap snapshot
dotnet gcdump collect -p 1234 -o heap.gcdump

# Открыть в Visual Studio / PerfView / dotnet-gcdump analyze
dotnet gcdump analyze heap.gcdump
# Визуализация: https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-gcdump
```

---

## Memory Fail Points — OutOfMemoryException

**Факты:**
- `OutOfMemoryException` — не всегда значит, что память кончилась
- Часто: фрагментация LOH, виртуальная память исчерпана (32-bit), GC budget превышен
- 64-bit: OOM реже, но LOH фрагментация может привести

```csharp
// Обработка OOM
try
{
    var largeArray = new byte[int.MaxValue / 2];
}
catch (OutOfMemoryException)
{
    GCSettings.LargeObjectHeapCompactionMode = GCLargeObjectHeapCompactionMode.CompactOnce;
    GC.Collect(2, GCCollectionMode.Forced);
    
    // Попробовать снова
    var largeArray = new byte[int.MaxValue / 2];
}

// MemoryFailPoint — проверка доступной памяти ДО аллокации
public static void AllocateLargeBuffer(int sizeInMB)
{
    using var memCheck = new MemoryFailPoint(sizeInMB);
    // Если исключение не брошено — памяти достаточно
    var buffer = new byte[sizeInMB * 1024 * 1024];
}
```

---

## Чек-лист

- [ ] dotnet-dump: collect, analyze, SOS commands
- [ ] SOS: dumpheap, gcroot, clrstack, clrthreads, finalizequeue
- [ ] Типичные утечки: события, таймеры, кэш, P/Invoke
- [ ] PerfView: GC Stats, Heap Snapshot, ETW events
- [ ] ETW: GcEventListener, EventCounters
- [ ] dotnet-gcdump: heap snapshots
- [ ] MemoryFailPoint: проверка до аллокации
- [ ] OOM: фрагментация LOH, virtual memory

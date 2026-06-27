# GC Tuning для разных сценариев

---

## ASP.NET Core — рекомендации

**Факты:**
- ASP.NET Core использует **Server GC** по умолчанию — правильно
- Основная цель: **throughput** (высокая пропускная способность)
- Паузы до 200ms приемлемы (HTTP request может ждать)

```json
// runtimeconfig.json — ASP.NET Core (рекомендуется)
{
  "runtimeOptions": {
    "configProperties": {
      "System.GC.Server": true,
      "System.GC.Concurrent": true,
      "System.GC.HeapHardLimit": 500_000_000,      // 500 MB max heap
      "System.GC.HeapHardLimitPercent": 0,          // Disable %-based
      "System.GC.NoAffinitize": false                // Affinitize to cores
    }
  }
}

// Для контейнеров с memory limit:
// GC автоматически использует cgroup limit как TotalAvailableMemory
// Но heap hard limit лучше задать явно (80% от container limit)
```

**Настройка для high-throughput API:**

```csharp
// Программная настройка в Program.cs
AppContext.SetSwitch("System.GC.Server", true);
AppContext.SetSwitch("System.GC.Concurrent", true);

// Ручной GC после request (НЕ ДЕЛАЙТЕ этого — убивает производительность)
// GC.Collect() в ASP.NET Core — антипаттерн!
```

### Что НЕЛЬЗЯ делать в ASP.NET Core

```csharp
// ❌ Никогда
GC.Collect();                                    // Stop-the-world в середине request
GC.WaitForPendingFinalizers();                   // Блокировка
Thread.Sleep(1000);                              // Занимает thread pool
Task.Delay(1000).Wait();                         // Deadlock risk
```

---

## Микросервисы — настройка

| Тип сервиса | GC Mode | Heap Limit | Latency Mode |
|---|---|---|---|
| **HTTP API** | Server GC | 80% container limit | Default |
| **gRPC** | Server GC | 70% container limit | SustainedLowLatency |
| **Background worker** | Server GC | 90% container limit | Default |
| **Event consumer** | Server GC | 85% container limit | Default |

```json
// Микросервис с gRPC (нужна низкая latency)
{
  "runtimeOptions": {
    "configProperties": {
      "System.GC.Server": true,
      "System.GC.Concurrent": true,
      "System.GC.HeapHardLimit": 300_000_000,
      "System.GC.HeapCount": 2,       // Если CPU limit = 2
      "System.GC.NoAffinitize": true   // В K8s лучше без affinity
    }
  }
}
```

---

## Desktop (WPF/WinForms) — настройка

**Факт:** Desktop использует **Workstation GC** + **Background** — минимум пауз UI.

```json
{
  "runtimeOptions": {
    "configProperties": {
      "System.GC.Server": false,           // Workstation для UI
      "System.GC.Concurrent": true,        // Background
      "System.GC.RetainVM": false          // Не держать VM после удаления
    }
  }
}
```

```csharp
// WPF — Low Latency во время анимации
public class AnimationScope : IDisposable
{
    private readonly GCLatencyMode _previous;
    
    public AnimationScope()
    {
        _previous = GCSettings.LatencyMode;
        GCSettings.LatencyMode = GCLatencyMode.LowLatency;
    }
    
    public void Dispose()
    {
        GCSettings.LatencyMode = _previous;
    }
}

// Использование
using (new AnimationScope())
{
    // Анимация без GC пауз
    await Task.Delay(1000);
}
```

---

## Real-time / Low Latency

**Факт:** Для real-time (игры, audio/video processing, trading) — `NoGCRegion`.

```csharp
// NoGCRegion — полное отключение GC
public void ProcessAudioFrame(Span<byte> audioData)
{
    var budget = EstimateFrameProcessingBudget();  // ~50 MB
    
    if (GC.TryStartNoGCRegion(budget, disallowFullBlockingGC: true))
    {
        try
        {
            // Real-time processing — нет GC пауз
            Process(audioData);
        }
        finally
        {
            GC.EndNoGCRegion();  // Возобновляет GC
        }
    }
    else
    {
        // Fallback — если budget превышен
        ProcessWithGc(audioData);
    }
}

// Для financial trading — SustainedLowLatency
GCSettings.LatencyMode = GCLatencyMode.SustainedLowLatency;
// Gen 2 GC только при нехватке памяти
```

---

## NUMA (Non-Uniform Memory Access)

**Факты:**
- NUMA — процессоры с разной скоростью доступа к разным банкам памяти
- Server GC на NUMA машинах — **GCHeap per NUMA node**
- `GCHeapCount` — должно быть кратно числу NUMA nodes

```json
// 2 NUMA nodes по 8 cores = 16 logical processors
{
  "runtimeOptions": {
    "configProperties": {
      "System.GC.Server": true,
      "System.GC.HeapCount": 8,        // 8 heaps (4 per NUMA node)
      "System.GC.NoAffinitize": false   // Affinitize к ядрам
    }
  }
}

// Проверка NUMA
Console.WriteLine($"NUMA nodes: {System.Runtime.GCSettings.IsServerGC}");
// NUMA доступен только с Server GC
```

---

## Memory Budget — HeapHardLimit

**Факты:**
- `HeapHardLimit` — **абсолютный** лимит heap (в байтах)
- `HeapHardLimitPercent` — **относительный** лимит (% от total memory)
- При превышении: GC собирает Gen 2 (Full GC) агрессивнее
- Если не помогает → OutOfMemoryException

```json
{
  "runtimeOptions": {
    "configProperties": {
      // Вариант 1: абсолютный (рекомендуется для контейнеров)
      "System.GC.HeapHardLimit": 500_000_000,   // 500 MB
      
      // Вариант 2: процентный (для VM без memory limit)
      "System.GC.HeapHardLimitPercent": 75       // 75% от доступной
    }
  }
}

// Проверка
var info = GC.GetGCMemoryInfo();
Console.WriteLine($"Heap hard limit: {info.HighMemoryLoadThresholdBytes / 1024 / 1024} MB");
```

---

## GC Performance Counters — мониторинг

```csharp
// Ключевые метрики для мониторинга
var counters = new Dictionary<string, Func<object>>
{
    ["gc-gen-0-count"] = () => GC.CollectionCount(0),
    ["gc-gen-1-count"] = () => GC.CollectionCount(1),
    ["gc-gen-2-count"] = () => GC.CollectionCount(2),
    ["gc-total-memory"] = () => GC.GetTotalMemory(false),
    ["gc-total-allocated"] = () => GC.GetTotalAllocatedBytes(),
};

// В Prometheus / OpenTelemetry
// dotnet-counters monitor --process-id 1234 System.Runtime

// Тревоги (alerts):
// - Gen 2 GC > 10/min → Full GC слишком часто
// - GC time > 15% → heap too small or too many allocations
// - Memory > 80% hard limit → scale up
// - LOH fragmentation > 20% → consider compaction
```

---

## GC и регионы (.NET 8+)

**Факт:** .NET 8+ поддерживает **GC Regions** — альтернатива сегментам для уменьшения latency:

```
Традиционные сегменты:
[ SOH Segment Gen 0/1/2 ][ LOH Segment ][ POH Segment ]

GC Regions (.NET 8+):
[ Region ][ Region ][ Region ][ Region ] ...
Каждый регион ~4 MB, содержит объекты одного поколения
Преимущество: можно освободить регион целиком (быстрее)
```

**Факт:** GC Regions — экспериментальная фича в .NET 8. Включается через `DOTNET_GCRegionSize`.

---

## Чек-лист

- [ ] ASP.NET Core: Server GC, no manual GC.Collect, HeapHardLimit
- [ ] Микросервисы: per-service tuning, container-aware
- [ ] Desktop: Workstation GC, Low Latency для анимаций
- [ ] Real-time: NoGCRegion, SustainedLowLatency
- [ ] NUMA: per-node heap allocation
- [ ] HeapHardLimit: абсолютный vs процентный
- [ ] GC Regions (.NET 8+): регионы вместо сегментов
- [ ] Мониторинг: Gen 2 count, GC time, LOH fragmentation

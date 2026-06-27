# GC Mode и конфигурация

---

## Workstation vs Server GC

| Параметр | Workstation GC | Server GC |
|---|---|---|
| **Потоки GC** | 1 thread | 1 thread на ядро (logical CPU) |
| **Потоки приложения** | Все потоки приостанавливаются | Каждый GC thread на своём ядре |
| **Размер сегмента** | 16 MB (SOH) / 16 MB (LOH) | ~200 MB на ядро (SOH/LOH) |
| **Latency** | Низкая (быстрее возобновление) | Выше (ждут все GC threads) |
| **Throughput** | Ниже | Выше (параллельный Mark/Compact) |
| **Память** | Меньше (1 сегмент) | Больше (N сегментов на N ядер) |
| **Когда** | Desktop apps, клиентские | Server apps, ASP.NET Core |

**Факт:** Server GC **по умолчанию** для ASP.NET Core. Workstation — для Desktop (WinForms, WPF).

### Настройка в .csproj

```xml
<PropertyGroup>
  <!-- Server GC (рекомендуется для серверов) -->
  <ServerGarbageCollection>true</ServerGarbageCollection>
  
  <!-- Concurrent GC (background) -->
  <ConcurrentGarbageCollection>true</ConcurrentGarbageCollection>
</PropertyGroup>
```

### Настройка через runtimeconfig.json

```json
{
  "runtimeOptions": {
    "configProperties": {
      "System.GC.Server": true,
      "System.GC.Concurrent": true,
      "System.GC.HeapCount": 4,
      "System.GC.NoAffinitize": false
    }
  }
}
```

---

## Background vs Non-Concurrent GC

**Факты:**
- **Background GC** (concurrent) — GC работает параллельно с приложением (Gen 0/1 только)
- **Non-Concurrent GC** — все потоки приостанавливаются на всё время GC (STW)
- Background GC **доступен** с .NET 4.0 (Workstation) / .NET 4.5 (Server)

```
Background GC (рекомендуется):
Gen 0: background collection (приложение работает)
Gen 1: background collection (приложение работает)
Gen 2: foreground collection (STW — все потоки стоят)

Non-Concurrent GC:
Gen 0/1/2: STW — все потоки стоят на всё время
```

| Mode | Gen 0/1 | Gen 2 | Latency |
|---|---|---|---|
| **Background** (default) | Parallel, app runs | Foreground (STW) | Low |
| **Non-Concurrent** | STW | STW | High (long pauses) |

**Факт:** Background GC уменьшает паузы, но увеличивает общее CPU usage (GC работает параллельно).

**Факт:** `GCSettings.LatencyMode` управляет режимом:

```csharp
// Low Latency — минимизирует Gen 2 GC (но может вызвать OOM)
GCSettings.LatencyMode = GCLatencyMode.LowLatency;
// GC не делает Gen 2 collection, пока не закончится память

// SustainedLowLatency — Gen 2 только при нехватке памяти
GCSettings.LatencyMode = GCLatencyMode.SustainedLowLatency;

// NoGCRegion — полное отключение GC (⚠️ опасно)
// Для real-time сценариев
bool success = GC.TryStartNoGCRegion(100 * 1024 * 1024);  // 100 MB budget
try
{
    // Код без GC (если не превышен budget)
}
finally
{
    GC.EndNoGCRegion();
}
```

---

## DATAS (Dynamic Adaptation To Application Size)

**Факты:**
- **DATAS** — динамическая настройка GC под приложение
- **Allocation budget** (Gen 0 size) адаптируется под allocation rate
- Чем быстрее аллокация → тем больше Gen 0 → реже GC (меньше overhead)

```
High allocation rate → большой Gen 0 budget → реже GC (но дольше каждое)
Low allocation rate  → малый Gen 0 budget  → чаще GC (но короче каждое)
```

---

## GCHeapCount и Affinitization

```csharp
// GC.HeapCount — ограничение числа GC heaps
// По умолчанию: число logical processors (Server GC)
// Настройка для Server GC с 4 heaps:

// runtimeconfig.json
{
  "runtimeOptions": {
    "configProperties": {
      "System.GC.HeapCount": 4
    }
  }
}

// GCBalance — балансировка нагрузки между heaps
// System.GC.Balance: true (default)

// NoAffinitize — не привязывать GC threads к ядрам
// System.GC.NoAffinitize: true
```

**Факт:** В контейнерах (Docker/K8s) `GCHeapCount` должен соответствовать CPU limit.

**Факт:** Если CPU limit = 4, но GC создаёт 8 heaps (hyperthreading), возможна конкуренция за CPU.

---

## GC в контейнерах

```csharp
// В Docker/K8s — GC автоматически определяет лимиты через cgroups
// GC.GetGCMemoryInfo().TotalAvailableMemoryBytes — память доступная контейнеру

// Но heap count нужно настраивать:
// Если CPU limit = 2 → GC создаст 2 heaps
// Если не указано → создаст по числу logical processors (может быть больше limit)

// Настройка для контейнера:
{
  "runtimeOptions": {
    "configProperties": {
      "System.GC.Server": true,
      "System.GC.HeapCount": 2,
      "System.GC.Concurrent": true
    }
  }
}

// Проверка GC режима в коде:
Console.WriteLine($"Server GC: {GCSettings.IsServerGC}");
Console.WriteLine($"Latency mode: {GCSettings.LatencyMode}");
```

---

## Large Object Heap — детали

**Факты:**
- **Порог:** ≥ 85 000 bytes (85 KB)
- LOH сегмент выделяется **отдельно** от SOH
- По умолчанию LOH **не компактируется** (фрагментация!)
- С .NET 4.5.1: принудительная компактизация
- .NET 8+: LOH может компактироваться автоматически при фрагментации > X%

```csharp
// Принудительная компактизация LOH
GCSettings.LargeObjectHeapCompactionMode = GCLargeObjectHeapCompactionMode.CompactOnce;
GC.Collect(2, GCCollectionMode.Forced);
// После этого LOH дефрагментирован (но GC пауза длиннее)

// Проверка фрагментации
var info = GC.GetGCMemoryInfo();
var lohFragmentation = (double)info.FragmentedBytes / info.HeapSizeBytes * 100;
if (lohFragmentation > 20)
{
    Log.Warning("LOH fragmentation > 20%, consider compaction");
}
```

**Факт:** Массивы `byte[85_000]` и `string` длиной > 42 500 символов (2 байта на char) попадают в LOH.

---

## POH (Pinned Object Heap) — .NET 5+

**Факты:**
- Специальный сегмент для pinned объектов
- До .NET 5: pinned объекты фрагментировали SOH
- С .NET 5: `POH` — pinned объекты в отдельном сегменте (не фрагментируют обычную кучу)
- POH не компактируется (pinned не двигаются, но зато не мешают SOH)

```csharp
// Автоматически: объекты с GCHandleType.Pinned → POH
// Явно: GC.AllocateArray<byte>(1000, pinned: true)

// Память POH:
var info = GC.GetGCMemoryInfo();
Console.WriteLine($"POH size: {info.PinnedObjectHeapSizeBytes / 1024} KB");
```

---

## Чек-лист

- [ ] Workstation vs Server GC: 1 GC thread vs per-core
- [ ] Background GC: concurrent Gen 0/1, STW Gen 2
- [ ] DATAS: adaptive Gen 0 budget
- [ ] Latency modes: LowLatency, SustainedLowLatency, NoGCRegion
- [ ] GCHeapCount: настройка для контейнеров
- [ ] LOH: > 85KB, фрагментация, принудительная компактизация
- [ ] POH: pinned objects (.NET 5+), отдельный сегмент
- [ ] Container: GC auto-detects cgroups, но HeapCount нужно задать

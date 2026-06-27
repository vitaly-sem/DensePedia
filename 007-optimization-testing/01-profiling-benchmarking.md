# Профилирование и бенчмарки

---

## BenchmarkDotNet

**Факты:**
- Стандарт для бенчмарков в .NET
- Запускает каждый метод несколько раз (warmup + iterations)
- Предотвращает оптимизации (вроде dead code elimination)
- Выдаёт статистику: Mean, Error, StdDev, GC collections

```csharp
[MemoryDiagnoser]          // GC collections per iteration
[RankColumn]               // Ranking by performance
[Orderer(SummaryOrderPolicy.FastestToSlowest)]
public class StringBenchmarks
{
    private const string Input = "Hello, World!";
    private static readonly byte[] Data = new byte[1000];
    
    [Benchmark(Baseline = true)]
    public bool ContainsSubstring()
        => Input.Contains("World");
    
    [Benchmark]
    public bool ManualLoop()
    {
        for (int i = 0; i <= Input.Length - 5; i++)
            if (Input[i..(i + 5)] == "World")
                return true;
        return false;
    }
    
    [Benchmark]
    public byte[] ArrayCopy() => Data[..500];
    
    [Benchmark]
    public Span<byte> SpanSlice() => Data.AsSpan()[..500];  // Zero allocation!
}

// Results:
// | Method            | Mean      | Allocated |
// |------------------ |----------:|----------:|
// | SpanSlice         |  0.002 ns |        0B |  ← 0 alloc
// | ArrayCopy         | 12.500 ns |      504B |
// | ContainsSubstring |  4.800 ns |        0B |
```

**Факт:** `[Benchmark(Baseline = true)]` — сравнивает остальные методы относительно этого (Ratio column).

**Факт:** `[MemoryDiagnoser]` — показывает Gen0/Gen1/Gen2 collections и allocated bytes.

**Факт:** BenchmarkDotNet запускает каждый метод **отдельным процессом** (изоляция от JIT warmup).

---

## dotnet-trace — трассировка в production

```bash
# Установка
dotnet tool install --global dotnet-trace

# Сбор трассировки запущенного процесса
dotnet trace collect --process-id 1234 --profile gc-verbose

# Конвертация в SpeedScope / PerfView
dotnet trace convert trace.nettrace --format speedscope
```

---

## PerfView — детальный анализ

**Факты:**
- Инструмент от Microsoft для ETW (Event Tracing for Windows)
- Анализ: CPU, GC, JIT, exceptions, contention, P/Invoke
- **Windows-only** (Linux: `dotnet-trace` + `speedscope`)

---

## dotnet-counters — мониторинг в реальном времени

```bash
dotnet tool install --global dotnet-counters

# Мониторинг
dotnet counters monitor --process-id 1234

# Результат:
# [System.Runtime]
#     cpu-usage (%)                                   25
#     working-set (MB)                               150
#     gc-heap-size (MB)                              80
#     gen-0-gc-count                                  5
#     gen-1-gc-count                                  2
#     gen-2-gc-count                                  1
#     time-in-gc (%)                                  3
#     exceptions-count                                0
#     threadpool-thread-count                         8
#     monitor-lock-contention-count                   0
```

---

## Профилирование кода — что искать

| Метрика | Норма | Тревога |
|---|---|---|
| CPU usage | < 80% | > 90% (CPU-bound) |
| GC time | < 5% | > 20% (аллокации) |
| Gen 2 GC | 0-1/час | > 10/час (LOH fragmentation) |
| Thread contention | 0 | > 100/min (lock contention) |
| Exception rate | 0 | > 100/min (try-catch в цикле) |
| P/Invoke calls | Low | High (managed→native overhead) |

---

## Чек-лист

- [ ] BenchmarkDotNet: MemoryDiagnoser, Baseline, RankColumn
- [ ] dotnet-trace: трассировка ETW, конвертация в speedscope
- [ ] PerfView: ETW анализ (Windows)
- [ ] dotnet-counters: real-time мониторинг
- [ ] Метрики: CPU, GC, contention, exceptions

# Оптимизация производительности кода

---

## Allocation-free — избегаем аллокаций

**Факты:**
- Каждая аллокация → GC pressure → пауза
- Hot path (частый код) должен быть **allocation-free**

```csharp
// ❌ Аллокация в hot path
public string ProcessItem(string item)
{
    return item.Trim().ToUpper();  // Trim + ToUpper = 2 string allocations
}

// ✅ ReadOnlySpan — zero allocation
public string ProcessItem(string item)
{
    var span = item.AsSpan().Trim();  // No allocation
    return string.Concat(span);       // 1 allocation (неизбежно)
}

// ❌ LINQ аллокации
var sum = items.Where(x => x > 0).Sum(x => x);  // Iterator allocations

// ✅ Loop — no allocations
int sum = 0;
foreach (var x in items)
    if (x > 0) sum += x;
```

---

## Inlining и горячие пути

**Факт:** JIT может **inline** маленькие методы (< 32 IL bytes). Атрибут `[MethodImpl]` управляет этим.

```csharp
// Запретить inlining (для profiling)
[MethodImpl(MethodImplOptions.NoInlining)]
public void DontInlineMe() { }

// Принудительное inlining (если JIT решил не inline)
[MethodImpl(MethodImplOptions.AggressiveInlining)]
public int FastAdd(int a, int b) => a + b;  // Будет встроен в caller
```

**Факт:** Inlining не работает для: виртуальных методов, больших методов, рекурсии, exception handling.

---

## SIMD — векторные инструкции

```csharp
// Vector<T> — SIMD (Single Instruction Multiple Data)
// Доступно на CPU с SSE/AVX

public static float SumSquared(Vector<float>[] vectors)
{
    float sum = 0;
    for (int i = 0; i < vectors.Length; i++)
    {
        // Одна инструкция → 4/8 float операций
        sum += Vector.Dot(vectors[i], vectors[i]);
    }
    return sum;
}

// Hardware intrinsics (более низкий уровень)
// Vector128, Vector256, Vector512 (AVX-512)
if (Avx2.IsSupported)
{
    // Используем AVX2 инструкции
    var vec = Avx2.LoadVector256(pointer);
}

// Проверка поддержки
Console.WriteLine(Vector.IsHardwareAccelerated);  // true/false
Console.WriteLine(Vector<byte>.Count);            // 16/32 bytes per operation
```

---

## String — оптимизация

```csharp
// ❌ Конкатенация в цикле
string result = "";
for (int i = 0; i < 1000; i++)
    result += i;  // O(n²) — 1000 allocations!

// ✅ StringBuilder
var sb = new StringBuilder(4000);
for (int i = 0; i < 1000; i++)
    sb.Append(i);
string result = sb.ToString();

// ✅ String.Create — для известного формата
string result = string.Create(10, (1, 2, 3), (span, state) =>
{
    var (a, b, c) = state;
    span[0] = (char)('0' + a);
    span[1] = ',';
    span[2] = (char)('0' + b);
    // ...
});

// ✅ string.Concat — оптимизирован для <= 4 аргументов
string result = string.Concat(a, b, c);  // Единственная аллокация
```

---

## async overhead

**Факт:** `async Task` метод без реального await — overhead ~100 bytes:

```csharp
// ❌ async without await — state machine создаётся зря
public async Task<int> GetValueAsync()
{
    return 42;  // Всё равно создаётся state machine
}

// ✅ ValueTask — без аллокации если синхронно
public ValueTask<int> GetValueAsync()
{
    return new ValueTask<int>(42);  // Zero allocation
}

// ❌ await Task.Run для быстрой операции
await Task.Run(() => Compute());  // ThreadPool + Task overhead
// ✅ Direct call
Compute();
```

---

## Caching — стратегии

```csharp
// 1. IMemoryCache — in-memory
public class DataCache
{
    private readonly IMemoryCache _cache;
    
    public async Task<Data> GetAsync(int id)
    {
        return await _cache.GetOrCreateAsync($"data-{id}", async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5);
            entry.SlidingExpiration = TimeSpan.FromMinutes(1);  // Сброс при доступе
            entry.Size = 1;  // Для ограничения размера
            return await _repository.GetAsync(id);
        });
    }
}

// 2. IDistributedCache — Redis
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = "localhost:6379";
});

// 3. Second-level cache (EF Core) — через interceptor
public class CachingQueryInterceptor : IQueryInterceptor
{
    public async Task<T> ExecuteAsync<T>(IQueryable<T> query)
    {
        var key = query.ToQueryString();  // SQL as cache key
        // Check cache → Execute → Cache
    }
}
```

---

## Чек-лист

- [ ] Allocation-free: Span, struct, array pool
- [ ] Inlining: AggressiveInlining / NoInlining
- [ ] SIMD: Vector<T>, Avx2, hardware intrinsics
- [ ] String: StringBuilder, String.Create, Concat
- [ ] async overhead: ValueTask для синхронного пути
- [ ] Caching: IMemoryCache, IDistributedCache, EF Second-level

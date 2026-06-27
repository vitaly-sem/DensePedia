# Оптимизация памяти и GC

---

## GC — три поколения

**Факты:**
- **Gen 0** — короткоживущие (микросекунды-секунды). Собирается часто.
- **Gen 1** — пережившие Gen 0. Буфер между Gen 0 и Gen 2.
- **Gen 2** — долгоживущие (статические, синглтоны, кэш). Собирается редко.
- **Large Object Heap (LOH)** — объекты > 85 KB (массивы byte[], string[]). **Gen 2 collection**.

```
Allocation → Gen 0 (быстро)
    ↓ если пережил GC
Gen 1 (редко)
    ↓ если пережил ещё
Gen 2 (Full GC — дорого!)
```

**Факт:** Gen 0 collection — ~1ms. Gen 2 Full collection — ~100ms-1s (stop-the-world).

**Факт:** `GC.GetTotalMemory(false)` — текущее использование. `GC.CollectionCount(0)` — сколько раз собиралось Gen 0.

---

## LOH (Large Object Heap) — проблемы

**Факты:**
- Объекты > 85 KB (86500 bytes) → LOH
- **Не компактируется** (до .NET 4.5.1) — фрагментация
- С .NET 4.5.1 — LOH compaction принудительно `GCSettings.LargeObjectHeapCompactionMode`
- LOH collection = Gen 2 GC (дорого)

```csharp
// ❌ LOH allocation от временных буферов
byte[] buffer = new byte[100_000];  // LOH!
// После использования — Gen 2 collection для освобождения

// ✅ ArrayPool — нет LOH
byte[] buffer = ArrayPool<byte>.Shared.Rent(100_000);
ArrayPool<byte>.Shared.Return(buffer);

// LOH compaction (если нужна дефрагментация)
GCSettings.LargeObjectHeapCompactionMode = GCLargeObjectHeapCompactionMode.CompactOnce;
GC.Collect(2, GCCollectionMode.Forced);
```

---

## Struct vs Class — memory/performance

| | Struct | Class |
|---|---|---|
| **Allocation** | Stack (inline) или часть объекта | Heap (GC-managed) |
| **GC pressure** | Нет (если не boxed) | Да |
| **Passing** | By value (copy) | By reference (4/8 bytes) |
| **Equality** | Value equality (может быть медленно) | Reference equality |
| **Inheritance** | Нет (`sealed` by default) | Да |
| **Null** | Nullable<T> или default | null |
| **When** | ≤ 16 bytes, short-lived, immutable | > 16 bytes, polymorphic, shared |

```csharp
// Struct — для маленьких, immutable value types
public readonly struct Point
{
    public int X { get; }
    public int Y { get; }
    
    public Point(int x, int y) => (X, Y) = (x, y);
}

// ❌ Struct > 16 bytes — может быть медленнее class (копирование)
public struct LargeStruct  // 32 bytes!
{
    public int A, B, C, D, E, F, G, H;  // 32 bytes on 64-bit
}
```

**Факт:** `readonly struct` — компилятор может избежать defensive copy при передаче через `in` или `ref`.

**Факт:** `ref struct` (Span) — живёт только на стеке, не может быть в heap.

---

## Object Pooling

```csharp
// ObjectPool<T> — переиспользование объектов (Microsoft.Extensions.ObjectPool)
public class StringBuilderPool
{
    private static readonly ObjectPool<StringBuilder> _pool = 
        new DefaultObjectPool<StringBuilder>(new StringBuilderPooledObjectPolicy());
    
    public static string Build(Action<StringBuilder> build)
    {
        var sb = _pool.Get();
        try
        {
            build(sb);
            return sb.ToString();
        }
        finally
        {
            sb.Clear();
            _pool.Return(sb);
        }
    }
}

// Кастомный пул для дорогих объектов
public class ConnectionPool
{
    private readonly ConcurrentBag<DbConnection> _connections = new();
    
    public DbConnection Rent()
        => _connections.TryTake(out var conn) ? conn : CreateConnection();
    
    public void Return(DbConnection conn)
    {
        if (conn.State == ConnectionState.Open)
            _connections.Add(conn);
        else
            conn.Dispose();  // Не возвращаем сломанные
    }
}
```

---

## Stackalloc и Buffer pooling

```csharp
// 1. stackalloc — для маленьких временных буферов (на стеке)
Span<byte> tempBuffer = stackalloc byte[256];  // < 1KB — оптимально

// 2. ArrayPool — для больших временных буферов
var pool = ArrayPool<byte>.Shared;
byte[] buffer = pool.Rent(4096);  // Из пула
try
{
    // Используем buffer
}
finally
{
    pool.Return(buffer);  // Обратно в пул
}
```

**Факт:** `ArrayPool<T>.Shared` — глобальный пул с сегментами. `ArrayPool<T>.Create()` — отдельный.

**Факт:** `Rent()` может вернуть массив **больше** запрошенного размера (до 2x). Используйте `buffer.AsSpan(0, requestedSize)`.

---

## Memory Leaks Detection

```csharp
// 1. Event subscription — WeakEventManager (см. WPF раздел)
// 2. Static collections — забыли очистить ConcurrentDictionary
// 3. Event handlers на static events
// 4. Timer callback — Timer держит ссылку
// 5. Lambda capture — скрытая ссылка

// Проверка через WeakReference
var obj = new MyClass();
var wr = new WeakReference(obj);
obj = null;
GC.Collect();
GC.WaitForPendingFinalizers();
Console.WriteLine(wr.IsAlive ? "LEAK!" : "Collected");
```

---

## Чек-лист

- [ ] GC Generations: Gen 0 (1ms), Gen 1, Gen 2 (10-100ms)
- [ ] LOH: > 85KB, фрагментация, ArrayPool
- [ ] Struct vs Class: value type, < 16 bytes, readonly
- [ ] ObjectPool<T>: переиспользование дорогих объектов
- [ ] stackalloc + ArrayPool: временные буферы
- [ ] Memory leaks: события, статики, таймеры, лямбды

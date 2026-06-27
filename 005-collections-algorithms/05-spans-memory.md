# Span, Memory и низкоуровневая работа с памятью

---

## Span\<T\> — stack-only reference type

**Факты:**
- `Span<T>` — **ref struct** (только на стеке, не в heap, не в полях класса)
- Представление **непрерывного блока памяти** любой природы (array, native, stackalloc)
- Zero-overhead абстракция над памятью
- **Slice** без аллокации (изменение view)

```csharp
// Span может указывать на:
// 1. Массив
int[] array = [1, 2, 3, 4, 5];
Span<int> span = array.AsSpan();

// 2. Часть массива (без копирования!)
Span<int> slice = span[1..^1];  // { 2, 3, 4 }

// 3. Native memory (Marshal.AllocHGlobal)
IntPtr ptr = Marshal.AllocHGlobal(100);
Span<byte> nativeSpan = new Span<byte>(ptr.ToPointer(), 100);

// 4. Stackalloc (очень быстро)
Span<byte> stackSpan = stackalloc byte[256];  // На стеке, не в heap
```

**Факт:** `Span<T>` позволяет избежать аллокаций при работе с подмассивами:

```csharp
// ❌ Аллокация — создание нового массива
var firstHalf = array.Take(array.Length / 2).ToArray();

// ✅ Zero allocation — Span view
Span<int> span = array.AsSpan();
var firstHalf = span[..(span.Length / 2)];
```

**Факт:** Span — **ref struct**, поэтому не может быть в:
- Поля класса
- Async методы (state machine — класс)
- Lambda/captured variables
- Boxing (нельзя привести к object)

```csharp
// ❌ Нельзя
class Foo { Span<int> _span; }           // Ref struct in class field
async Task Bar() { Span<int> s = ...; }  // Span in async method
Func<int> f = () => { Span<int> s; };    // Span in lambda

// ✅ Можно
ref struct Foo { Span<int> _span; }      // Ref struct in ref struct
Span<int> s = ...;
foreach (var item in s) { }              // Iteration
```

---

## Memory\<T\> — heap-compatible аналог Span

**Факты:**
- `Memory<T>` — **struct** (может быть в heap, в async, в полях класса)
- `ReadOnlyMemory<T>` — read-only версия
- `.Span` — получить Span (мгновенно, O(1))

```csharp
// Memory — можно в async методах
public async Task ProcessAsync(Memory<byte> buffer)
{
    // Получаем Span (только синхронно — Span ref struct)
    Span<byte> span = buffer.Span;
    
    // Работаем с данными
    Process(span);
    
    // Memory передаётся через await безопасно
    await Task.Yield();
    
    // Снова получаем Span
    Process(buffer.Span);
}

// Memory — можно в полях класса
public class BufferPool
{
    private Memory<byte> _buffer;
    
    public Memory<byte> Rent(int size) { ... }
}
```

---

## ReadOnlySequence\<T\> — не-непрерывная память

**Факты:**
- Для работы с **сегментированной памятью** (например, HTTP/2 frames, pipe)
- Состоит из **цепочки ReadOnlyMemory<T>** сегментов
- Используется в: `System.IO.Pipelines`, Kestrel, SignalR

```csharp
// Pipelines — чтение из сокета без копирования
public async Task ProcessPipeAsync(PipeReader reader)
{
    while (true)
    {
        ReadResult result = await reader.ReadAsync();
        ReadOnlySequence<byte> buffer = result.Buffer;
        
        // Обработка последовательности
        foreach (var segment in buffer)
        {
            ProcessSegment(segment.Span);
        }
        
        // Сообщаем, сколько прочитали
        reader.AdvanceTo(buffer.End);
    }
}
```

---

## stackalloc — аллокация на стеке

**Факты:**
- Аллокация на **стеке**, не в managed heap (нет GC pressure)
- Автоматически освобождается при выходе из метода
- Стек ограничен (~1 MB), не для больших данных
- **Очень быстро** — просто сдвиг stack pointer

```csharp
// stackalloc — только в unsafe или Span контексте
Span<byte> buffer = stackalloc byte[256];  // На стеке

// Для больших данных — ArrayPool<T>
byte[] pooled = ArrayPool<byte>.Shared.Rent(4096);  // Из пула
try
{
    // Используем
}
finally
{
    ArrayPool<byte>.Shared.Return(pooled);
}

// Бенчмарк: stackalloc 256 bytes vs new byte[256]
// stackalloc: ~1 ns, 0 GC
// new byte[256]: ~30 ns, 1 GC alloc
```

**Факт:** `stackalloc` в Span контексте **не требует** `unsafe`:

```csharp
// Требует unsafe
unsafe { int* ptr = stackalloc int[10]; }

// Не требует unsafe (Span-safe контекст)
Span<int> span = stackalloc int[10];  // .NET 6+
```

---

## Unsafe Code — когда нужно

```csharp
// Unsafe — для максимальной производительности
// Использовать ТОЛЬКО если доказано, что safe код медленнее

private static unsafe void FastCopy(byte[] source, byte[] dest, int count)
{
    fixed (byte* pSource = source, pDest = dest)
    {
        // SIMD-оптимизированное копирование
        Buffer.MemoryCopy(pSource, pDest, dest.Length, count);
    }
}

// Unsafe class (System.Runtime.CompilerServices)
// Для низкоуровневых манипуляций
ref int GetReference(int[] array)
{
    return ref Unsafe.Add(ref MemoryMarshal.GetArrayDataReference(array), 10);
}
```

---

## MemoryMarshal — низкоуровневые преобразования

```csharp
// 1. Cast — reinterpret байтов как struct (без копирования)
ReadOnlySpan<byte> bytes = GetBytes();
ReadOnlySpan<MyStruct> structs = MemoryMarshal.Cast<byte, MyStruct>(bytes);

// 2. Получить reference на элемент массива
ref int first = ref MemoryMarshal.GetArrayDataReference(array);

// 3. CreateFromPinnedArray — Memory из pinned array
var memory = MemoryMarshal.CreateFromPinnedArray(array, 0, array.Length);

// 4. AsBytes — любой Span<T> → Span<byte>
Span<int> ints = stackalloc int[10];
Span<byte> bytes = MemoryMarshal.AsBytes(ints);
```

---

## Чек-лист

- [ ] Span<T>: ref struct, zero-overhead slice, stack-only
- [ ] Memory<T>: heap-compatible, можно в async/fields
- [ ] ReadOnlySequence<T>: segmented memory, Pipelines
- [ ] stackalloc: stack allocation, no GC, ограничен 1MB
- [ ] ArrayPool<T>: pooling для временных буферов
- [ ] MemoryMarshal: Cast, GetArrayDataReference
- [ ] Unsafe: только при доказанной необходимости

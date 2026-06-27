# Native Memory и P/Invoke

---

## SafeHandle — управление native ресурсами

**Факты:**
- `SafeHandle` — managed wrapper для native handles
- **Наследуется** от `SafeHandleZeroOrMinusOneIsInvalid`
- Автоматическая финализация при GC
- Защита от **handle recycling** (атака через повторное использование handle)

```csharp
// Кастомный SafeHandle
public class FileHandle : SafeHandleZeroOrMinusOneIsInvalid
{
    private FileHandle() : base(ownsHandle: true) { }
    
    protected override bool ReleaseHandle()
    {
        // Вызывается Finalizer thread'ом
        return NativeMethods.CloseHandle(handle);
    }
}

// Использование в managed коде
public class FileReader : IDisposable
{
    private readonly FileHandle _handle;
    
    public FileReader(string path)
    {
        _handle = NativeMethods.CreateFile(path);
    }
    
    public void Dispose()
    {
        _handle.Dispose();
        GC.SuppressFinalize(this);
    }
}
```

**Факт:** `SafeHandle` реализует **critical finalization** — гарантированно вызывается даже при `AppDomain.Unload`.

**Факт:** **Не используйте** `IntPtr` для native handles — используйте `SafeHandle`. `IntPtr` утекает, если исключение произошло до ручного `CloseHandle`.

---

## GC.AddMemoryPressure — сообщаем GC о native памяти

**Факт:** GC не знает о native памяти. Если managed объект мал, но держит большой native buffer — GC будет собираться редко, native память утекает.

```csharp
public class NativeBitmap : IDisposable
{
    private IntPtr _data;
    private readonly int _size;
    
    public NativeBitmap(int width, int height)
    {
        _size = width * height * 4;  // 4 bytes per pixel
        _data = Marshal.AllocHGlobal(_size);
        
        // Сообщаем GC о native аллокации
        GC.AddMemoryPressure(_size);
    }
    
    public void Dispose()
    {
        if (_data != IntPtr.Zero)
        {
            Marshal.FreeHGlobal(_data);
            _data = IntPtr.Zero;
            
            // Сообщаем GC об освобождении
            GC.RemoveMemoryPressure(_size);
        }
    }
}

// Вариант с SafeHandle + AddMemoryPressure
public class NativeBufferHandle : SafeHandleZeroOrMinusOneIsInvalid
{
    private readonly int _size;
    
    public NativeBufferHandle(int size) : base(true)
    {
        _size = size;
        SetHandle(Marshal.AllocHGlobal(size));
        GC.AddMemoryPressure(size);  // Important!
    }
    
    protected override bool ReleaseHandle()
    {
        GC.RemoveMemoryPressure(_size);  // Important!
        Marshal.FreeHGlobal(handle);
        return true;
    }
}
```

**Факт:** `GC.AddMemoryPressure` увеличивает GC budget, заставляя GC собираться чаще. Без него — OOM при большом потреблении native памяти.

---

## Pinned Objects и GCHandle

**Факты:**
- **Pinned object** — GC не может переместить (для P/Invoke)
- До .NET 5: pinned объекты **фрагментировали SOH** (Gen 2 растёт)
- С .NET 5: POH (Pinned Object Heap) — отдельный сегмент

```csharp
// 1. P/Invoke с marshaling (автоматический pin)
[DllImport("kernel32.dll")]
private static extern void CopyMemory(IntPtr dest, IntPtr src, int count);

public void Copy(byte[] source, byte[] destination)
{
    // Маршалер автоматически пинит массивы на время вызова
    Marshal.Copy(source, 0, IntPtr.Zero, source.Length);
}

// 2. Ручной pin через GCHandle (долгий pin)
public class BufferWrapper : IDisposable
{
    private readonly byte[] _buffer;
    private GCHandle _handle;
    
    public BufferWrapper(int size)
    {
        _buffer = GC.AllocateArray<byte>(size, pinned: true);  // .NET 5+ — POH!
        _handle = GCHandle.Alloc(_buffer, GCHandleType.Pinned);
    }
    
    public IntPtr Pointer => _handle.AddrOfPinnedObject();
    
    public void Dispose()
    {
        if (_handle.IsAllocated)
            _handle.Free();
    }
}

// 3. GC.AllocateArray<T>(pinned: true) — современный способ
var buffer = GC.AllocateArray<byte>(4096, pinned: true);  // На POH
// ... P/Invoke с buffer ...
// GC сам освободит pinned (на POH не фрагментирует SOH)
```

**Факт:** `fixed` statement — временный pin (stack-only):

```csharp
public unsafe void Process(byte[] buffer)
{
    fixed (byte* ptr = buffer)
    {
        // ptr valid only in this scope
        NativeMethods.Process(ptr, buffer.Length);
    }
    // После фигурной скобки — объект больше не pinned
}
```

---

## NativeMemory — выделение без GC (NET 6+)

**Факты:**
- `NativeMemory` — аллокация/освобождение native памяти без managed объектов
- Не создаёт managed объекты — нет GC pressure
- **Требует** `unsafe`

```csharp
public unsafe class NativeBuffer : IDisposable
{
    private void* _ptr;
    private readonly int _size;
    
    public NativeBuffer(int size)
    {
        _size = size;
        _ptr = NativeMemory.Alloc((nuint)size);  // Нет managed объекта
        NativeMemory.Clear(_ptr, (nuint)size);   // Zero-init
    }
    
    public Span<byte> AsSpan() => new Span<byte>(_ptr, _size);
    
    public void Dispose()
    {
        if (_ptr != null)
        {
            NativeMemory.Free(_ptr);  // Быстрое освобождение
            _ptr = null;
        }
    }
}

// NativeMemory.AlignedAlloc — для выравнивания (SIMD, AVX)
var aligned = NativeMemory.AlignedAlloc(1024, 64);  // 64-byte aligned
```

**Факт:** `NativeMemory` значительно быстрее `Marshal.AllocHGlobal`, так как использует `malloc`/`aligned_alloc` напрямую.

---

## Marshal — P/Invoke marshaling

```csharp
// Маршалинг struct
[StructLayout(LayoutKind.Sequential, CharSet = CharSet.Unicode)]
public struct NativePoint
{
    public int X;
    public int Y;
    [MarshalAs(UnmanagedType.ByValTStr, SizeConst = 256)]
    public string Name;
}

[DllImport("native.dll", CallingConvention = CallingConvention.Cdecl)]
private static extern int ProcessPoint(ref NativePoint point);

// Маршалинг строк
[DllImport("native.dll", CharSet = CharSet.Unicode)]
private static extern IntPtr CreateString([MarshalAs(UnmanagedType.LPWStr)] string input);

// Маршалинг callback (delegate → function pointer)
[UnmanagedFunctionPointer(CallingConvention.Cdecl)]
public delegate void NativeCallback(int status, IntPtr userData);

[DllImport("native.dll")]
private static extern void SetCallback(NativeCallback callback, IntPtr userData);
```

**Факт:** `MarshalAs` — управляет тем, как .NET marshaller конвертирует managed типы в native. Неправильный `UnmanagedType` → crash.

---

## Чек-лист

- [ ] SafeHandle: managed wrapper, critical finalization, anti-recycling
- [ ] GC.AddMemoryPressure: сообщаем GC о native памяти
- [ ] Pinned: GCHandle, fixed, GC.AllocateArray (pinned: true), POH
- [ ] NativeMemory: unsafe, без managed объектов, быстрее Marshal
- [ ] Marshal: struct layout, string marshaling, callbacks
- [ ] GC.RemoveMemoryPressure: обязательно в Dispose/Finalize
- [ ] POH (.NET 5+): pinned отдельно от SOH

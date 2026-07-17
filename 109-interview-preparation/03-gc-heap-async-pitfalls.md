# GC, стек/куча и Async-подводные камни

---

## GC — поколения и устройство кучи

### Управляемая куча (Managed Heap)

```
┌──────────────────────────────────────────────────────────────┐
│                    MANAGED HEAP                              │
├──────────┬──────────────────────┬────────────┬──────────────┤
│  Gen 0   │       Gen 1          │   Gen 2    │    LOH       │
│  (~256KB)│   (~2MB)             │  (остаток) │  (≥85000 B)  │
│  ephemeral                      │            │              │
│  segment                        │            │              │
└──────────┴──────────────────────┴────────────┴──────────────┘

POH (Pinned Object Heap) — C# 10+: объекты с фиксированным адресом
```

### Процесс сборки мусора

```
Алгоритм (Mark-Sweep-Compact для малых поколений):

1. MARK: GC обходит граф живых объектов от корней (roots):
   - Статические поля
   - Локальные переменные на стеке
   - CPU регистры
   - GC Handles (GCHandle)
   
2. SWEEP: Непомеченные объекты считаются мёртвыми — память освобождается
   
3. COMPACT: Живые объекты сдвигаются в начало сегмента
   (ТОЛЬКО для Gen 0, Gen 1, Gen 2; НЕ для LOH по умолчанию)

Gen 0 сборка (самая частая):
   - Выжившие объекты продвигаются в Gen 1
   - Gen 0 полностью очищается

Gen 1 сборка:
   - Выжившие объекты продвигаются в Gen 2
   - Запускается когда Gen 0 недостаточно

Gen 2 сборка (full GC — самая дорогая):
   - Выжившие остаются в Gen 2
   - Может сжимать LOH (только с GCSettings.LargeObjectHeapCompactionMode)
```

### Ключевые цифры для собеседования

| Порог | Значение | Примечание |
|-------|----------|------------|
| LOH threshold | 85 000 байт | Объекты ≥85KB попадают в LOH |
| Array LOH | ≥85KB (для byte[] — ≥85000 элементов с заголовком) | |
| Gen 0 budget | ~256KB (динамический, зависит от кеша CPU) | |
| Gen 1 budget | ~2MB (динамический) | |
| Double array LOH | ≥ 85000/8 ≈ 10625 элементов | |
| Object header | 8 байт (x64; sync block + method table) | |
| Minimal object size | 24 байта (x64: header + empty class) | |

---

## Stack vs Heap — что где живёт

```
СТЕК (Stack)                    КУЧА (Heap)
─────────────────────────────   ─────────────────────────────
- Value types (struct, int,     - Reference types (class,
  bool, enum, record struct)      record, interface, delegate,
- Ссылки на объекты (pointer)     array)
- Локальные переменные           - Boxed value types
- Параметры методов              - Статические поля
- Возвращаемые адреса            - Объекты с dynamic lifetime
- Span<T> (ref struct) —         - Закрытия (closures)
  ТОЛЬКО на стеке                - Лямбды, захватившие переменные
```

```csharp
// Где что живёт:
class Program
{
    static int StaticField;          // Куча (static segment)

    void Method(int param)           // param — стек
    {
        int local = 42;              // Стек
        var list = new List<int>();  // list (ссылка) — стек; List<int> (объект) — куча
        var point = new PointType(); // struct PointType — стек
        object boxed = local;        // boxed (ссылка) — стек; упакованный int — куча
    }
}
```

### Boxing/Unboxing — скрытая аллокация

```csharp
// BOXING — value type → object: аллокация в куче
int value = 42;
object boxed = value;           // BOXING: 24+ байта в куче + копирование

// Unboxing — object → value type: копирование из кучи
int unboxed = (int)boxed;       // UNBOXING: чтение из кучи

// Скрытый boxing:
var list = new ArrayList();     // ArrayList хранит object
list.Add(42);                   // BOXING: int → object
list.Add(new Point());          // BOXING: struct → object

// Dictionary с struct-ключом:
var dict = new Dictionary<Point, string>();
dict.TryGetValue(new Point(1,2), out _);  
// GetHashCode вызывается на struct — БЕЗ boxing (через constrained call)
```

---

## GC и производительность — главные советы

```
1. Избегай аллокаций в горячих путях:
   - Используй ArrayPool<T>, StringBuilder (capacity), Span<T>
   - Кешируй делегаты и Task-объекты

2. Минимизируй время жизни объектов:
   - Объекты, умирающие в Gen 0 — бесплатны (почти)
   - Объекты, доживающие до Gen 2 — дороги

3. Не пининг (pinning) надолго:
   - GCHandle.Alloc(obj, GCHandleType.Pinned) мешает GC сжимать кучу
   - fixed блок — только на минимальное время

4. LOH: избегай крупных краткоживущих массивов
   - Каждый такой массив фрагментирует LOH
   - Используй ArrayPool для буферов
```

---

## Async/await — подводные камни

### Deadlock из-за SynchronizationContext

**Самая известная проблема:**

```csharp
// КЛАССИЧЕСКИЙ DEADLOCK — в UI-приложении (WPF/WinForms) или ASP.NET (до Core)
public void OnButtonClick()
{
    // !!! НЕ ДЕЛАЙТЕ ТАК:
    var result = GetDataAsync().Result;   // БЛОКИРУЕТ UI-поток
    // Deadlock: UI-поток ждёт .Result, а continuation await'a
    //           хочет вернуться в UI-поток (SynchronizationContext)
}

async Task<string> GetDataAsync()
{
    await Task.Delay(1000);
    return "data";  // Хочет выполниться в UI-потоке, но он ЗАНЯТ
}

// ПРАВИЛЬНО:
public async void OnButtonClick()
{
    var result = await GetDataAsync().ConfigureAwait(false);
    // ИЛИ: var result = await GetDataAsync(); — но continuation в UI-потоке
}
```

### ConfigureAwait(false) — когда и зачем

```
ConfigureAwait(false): продолжение НЕ захватывает SynchronizationContext

ПРАВИЛО:
  - ВСЕГДА используйте ConfigureAwait(false) в библиотечном коде
  - НЕ используйте в UI-обработчиках (нужен контекст для обновления UI)
  - НЕ используйте в ASP.NET Core — там нет SynchronizationContext

Последствия неиспользования:
  - WPF/WinForms: deadlock при .Result/.Wait()
  - ASP.NET (Framework): deadlock, снижение пропускной способности
  - ASP.NET Core: без последствий (нет контекста)
```

### Исключения в async void

```csharp
// async void — ОПАСНО: исключение не поймать!
async void FireAndForget()
{
    await Task.Delay(100);
    throw new Exception("BOOM!"); // УБИВАЕТ процесс (в .NET Framework)
                                  // или теряется (в .NET Core)
}

// ПРАВИЛЬНО: всегда async Task, кроме event handlers
async Task DoWorkAsync()
{
    await Task.Delay(100);
    throw new Exception("BOOM!"); // Поймается в вызывающем коде
}

// Fire-and-forget с обработкой ошибок:
async Task SafeFireAndForget()
{
    try { await DoWorkAsync(); }
    catch (Exception ex) { Log.Error(ex); } // ЯВНАЯ обработка
}
```

### ThreadPool Starvation

```csharp
// ПРОБЛЕМА: синхронное ожидание на ThreadPool
Task.Run(() =>
{
    var result = SlowApiCallAsync().Result;  // БЛОКИРУЕТ поток ThreadPool!
}).Wait();

// Вместо этого — async all the way:
var result = await SlowApiCallAsync();  // Поток возвращается в пул во время ожидания

// ПРОБЛЕМА: много долгих Task.Run()
Parallel.For(0, 1000, i =>
{
    var result = DoSyncWork();  // Блокирует потоки ThreadPool
});

// Для CPU-bound работы лучше:
Parallel.For(0, 1000, i => DoSyncWork());  // OK — CPU-bound
// Для I/O-bound: ТОЛЬКО async/await, без Task.Run()
```

---

## Типичные вопросы на собеседовании

### Q1: Где живут поля struct внутри class?

```csharp
class Container
{
    int x;            // Куча (поле объекта)
    Point _point;     // Куча (часть объекта Container)
    List<int> _list;  // Куча (ссылка + сам List)
}

struct Point { public int X, Y; }

void Method()
{
    var c = new Container();    // c — стек (ссылка), Container — куча
    var p = new Point();        // p — стек (struct полностью на стеке)
    var arr = new Point[10];    // arr — стек (ссылка), Point[10] — куча
                                // Все 10 Point — куча (часть массива)
}
```

### Q2: Почему Span\<T\> только на стеке?

```
Span<T> — ref struct, может существовать ТОЛЬКО на стеке.
Причина: Span содержит managed pointer (ref T _reference).
Managed pointer не может жить на куче — GC не знает как его обновлять
при сжатии кучи.

Ограничения:
  - Не может быть полем класса
  - Не может быть в async-методе (async state machine — класс!)
  - Не может быть в лямбде/замыкании
  - Не может быть в generic с class-ограничением
```

### Q3: Объясните разницу value type vs reference type

```
┌──────────────────────┬──────────────────────────┐
│     VALUE TYPE       │     REFERENCE TYPE       │
├──────────────────────┼──────────────────────────┤
│ struct, enum, int,   │ class, record, interface,│
│ bool, record struct  │ delegate, array, string  │
├──────────────────────┼──────────────────────────┤
│ На стеке (или часть  │ Ссылка на стеке,         │
│ объемлющего объекта) │ объект в куче            │
├──────────────────────┼──────────────────────────┤
│ Копируется при       │ Копируется ссылка        │
│ присваивании         │ (оба указывают на 1 obj) │
├──────────────────────┼──────────────────────────┤
│ Не может быть null   │ Может быть null          │
│ (без Nullable<T>)    │                          │
├──────────────────────┼──────────────────────────┤
│ Не участвует в GC    │ Участвует в GC           │
│ (нет заголовка)      │ (есть object header)     │
├──────────────────────┼──────────────────────────┤
│ Наследование:        │ Наследование: да          │
│ только интерфейсы    │ (кроме sealed)           │
└──────────────────────┴──────────────────────────┘
```

---

## Чек-лист: GC и async

- [ ] Как работают поколения GC? Что такое Gen 0/1/2/LOH/POH?
- [ ] Что попадает в LOH? Почему LOH не сжимается по умолчанию?
- [ ] Где живут value types vs reference types?
- [ ] Что такое boxing и почему он дорогой?
- [ ] Почему `Span<T>` — `ref struct` и не может быть в куче?
- [ ] Классический deadlock: `.Result` на UI-потоке + async. Как избежать?
- [ ] `ConfigureAwait(false)` — когда использовать, когда нет?
- [ ] Почему `async void` опасен?
- [ ] Что такое ThreadPool starvation и как его избежать?
- [ ] Чем отличается `Task.Run` от `await`?

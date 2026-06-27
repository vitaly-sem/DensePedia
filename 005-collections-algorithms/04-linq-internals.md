# LINQ — внутреннее устройство

---

## Deferred Execution (отложенное выполнение)

**Факты:**
- LINQ методы **не выполняются немедленно** — они строят expression tree или iterator
- Выполнение происходит при **итерации** (foreach, .ToList(), .Count(), .First())
- Deferred vs Immediate:

```csharp
IEnumerable<int> query = numbers.Where(n => n > 5).Select(n => n * 2);
// ↑ Ничего не выполнилось — построен iterator chain

List<int> result = query.ToList();
// ↑ Выполнение: numbers итерируется, Where фильтрует, Select преобразует

// Immediate execution operators:
// ToList(), ToArray(), ToDictionary(), ToHashSet()
// Count(), First(), Last(), Single(), Any(), All()
```

**Факт:** Deferred execution позволяет работать с **бесконечными последовательностями**:

```csharp
// Бесконечная последовательность + deferred = работает
IEnumerable<int> Fibonacci()
{
    int a = 0, b = 1;
    while (true)
    {
        yield return a;
        (a, b) = (b, a + b);
    }
}

var evenFib = Fibonacci().Where(x => x % 2 == 0).Take(10).ToList();
// ↓
// Берёт только первые 10 чётных чисел Фибоначчи
// Никогда не вычисляет 11-е
```

---

## Streaming vs Buffering

**Факты:**
- **Streaming operators** — обрабатывают элемент и передают дальше (Where, Select, Take)
- **Buffering operators** — требуют всю последовательность (OrderBy, GroupBy, Reverse)

```csharp
// Streaming — O(1) memory
// Where, Select, Take, Skip, TakeWhile, Distinct (частично)
var stream = source.Where(x => x > 0).Select(x => x * 2);

// Buffering — O(n) memory (нужна вся последовательность)
// OrderBy, ThenBy, GroupBy, Reverse, Cast (some)
var buffered = source.OrderBy(x => x).GroupBy(x => x % 2);
// ↑ OrderBy кэширует все элементы → сортирует → выдаёт
// GroupBy кэширует все элементы → группирует → выдаёт
```

**Факт:** `Distinct()` использует `HashSet` внутри — streaming (не требует всю последовательность), но O(n) memory.

**Факт:** `Take()` — streaming, но может читать из source больше, чем нужно (batch size для I/O оптимизации).

---

## Iterator Implementation (yield return)

```csharp
// Компилятор превращает yield return в state machine (как async)
public static IEnumerable<T> Where<T>(this IEnumerable<T> source, Func<T, bool> predicate)
{
    foreach (var item in source)
    {
        if (predicate(item))
            yield return item;
    }
}

// Компилятор генерирует:
// 1. Класс, реализующий IEnumerator<T> + IEnumerable<T>
// 2. State machine с состояниями: -1 (start), 0 (running), -2 (done)
// 3. MoveNext() — выполняет код до следующего yield return
// 4. Current — текущий yield return значение
```

**Факт:** Каждый `yield return` добавляет **состояние** в state machine. Цепочка из 10 LINQ операторов = 10 state machines.

---

## IQueryable — Expression Trees

**Факты:**
- `IQueryable<T>` строит **Expression Tree**, не делегаты
- `IEnumerable<T>` — делегаты (Func<>, Action<>)
- `IQueryable<T>` нужен для: EF Core, NHibernate, MongoDB LINQ provider

```csharp
// IEnumerable — выполняется в памяти (client-side)
IEnumerable<Order> enumerable = context.Orders.Where(o => o.Amount > 100);
// var func = (Order o) => o.Amount > 100; → IL code

// IQueryable — expression tree (server-side SQL)
IQueryable<Order> queryable = context.Orders.Where(o => o.Amount > 100);
// Expression: o => o.Amount > 100 → Expression<Func<Order, bool>> → SQL WHERE

// Разница: IQueryable компилируется в SQL на сервере
// IEnumerable загружает все данные и фильтрует в памяти

// ❌ Bad — загружает ВСЕ Orders в память
var bad = context.Orders.Where(o => o.Amount > 100).AsEnumerable()
    .OrderBy(o => SomeClientSideFunction(o)).ToList();

// ✅ Good — SQL выполнит фильтр, только результат в память
var good = context.Orders
    .Where(o => o.Amount > 100)
    .OrderBy(o => o.Date)
    .ToList();
```

**Факт:** `AsEnumerable()` ломает IQueryable chain — всё после него выполняется в памяти.

**Факт:** Expression Trees нельзя использовать: ref/out parameters, optional parameters, unsafe code.

---

## Custom LINQ Operators

```csharp
// Кастомные операторы — расширение IEnumerable<T>
public static class LinqExtensions
{
    // ForEach — нет встроенного (только List<T>.ForEach)
    public static void ForEach<T>(this IEnumerable<T> source, Action<T> action)
    {
        foreach (var item in source)
            action(item);
    }
    
    // DistinctBy — появился в .NET 6, но можно написать самому
    public static IEnumerable<T> DistinctBy<T, TKey>(
        this IEnumerable<T> source, Func<T, TKey> keySelector)
    {
        var seen = new HashSet<TKey>();
        foreach (var item in source)
            if (seen.Add(keySelector(item)))
                yield return item;
    }
    
    // Batch — разделение на chunks
    public static IEnumerable<IEnumerable<T>> Batch<T>(
        this IEnumerable<T> source, int size)
    {
        using var enumerator = source.GetEnumerator();
        while (enumerator.MoveNext())
            yield return YieldBatch(enumerator, size);
    }
    
    private static IEnumerable<T> YieldBatch<T>(IEnumerator<T> enumerator, int size)
    {
        yield return enumerator.Current;
        for (int i = 1; i < size && enumerator.MoveNext(); i++)
            yield return enumerator.Current;
    }
    
    // Where with index
    public static IEnumerable<T> Where<T>(
        this IEnumerable<T> source, Func<T, int, bool> predicate)
    {
        int index = 0;
        foreach (var item in source)
            if (predicate(item, index++))
                yield return item;
    }
}
```

---

## LINQ Performance Gotchas

```csharp
// 1. Множественная итерация — источник итерируется каждый раз
var filtered = collection.Where(x => x > 10);
var count = filtered.Count();       // Iterates
var first = filtered.First();        // Iterates again!
var list = filtered.ToList();        // Iterates again!

// ✅ Решение: материализовать один раз
var list = collection.Where(x => x > 10).ToList();
var count = list.Count;              // No iteration
var first = list[0];

// 2. OrderBy().First() — O(n log n) сортировка
var max = collection.OrderByDescending(x => x.Value).First();
// ✅ Лучше: MaxBy() — O(n)
var max = collection.MaxBy(x => x.Value);  // .NET 6+

// 3. Count() на IEnumerable (не ICollection)
// Если source — не ICollection, Count() итерирует всю коллекцию
if (collection.Count() > 0)  // O(n)!
// ✅ Better: Any() — O(1) (проверяет первый элемент)
if (collection.Any())  // O(1)

// 4. Where().ToList().Where().ToList() — множественные итерации
var result = collection.Where(x => x > 0).ToList()
    .Where(x => x % 2 == 0).ToList();
// ✅ Лучше: один проход
var result = collection.Where(x => x > 0 && x % 2 == 0).ToList();

// 5. Contains в цикле — O(n²)
var results = items.Where(item => lookup.Contains(item.Id)).ToList();
// ✅ Лучше: lookup в HashSet
var lookupSet = lookup.ToHashSet();
var results = items.Where(item => lookupSet.Contains(item.Id)).ToList();
```

---

## Чек-лист

- [ ] Deferred vs Immediate execution
- [ ] Streaming vs Buffering operators (memory impact)
- [ ] Iterator implementation (yield return state machine)
- [ ] IQueryable vs IEnumerable: Expression Trees vs Delegates
- [ ] AsEnumerable() breaks IQueryable chain
- [ ] Multiple iteration problem
- [ ] LINQ performance: Count vs Any, OrderBy.First vs MaxBy

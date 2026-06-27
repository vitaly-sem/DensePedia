# Задачи для собеседований: Алгоритмы и коллекции — с ответами

---

## Задача 1: Поиск дубликата в массиве

**Условие:** Дан массив из N+1 целых чисел, где каждое число от 1 до N. Ровно одно число повторяется. Найти его за O(n) времени и O(1) памяти, не модифицируя массив.

**Что проверяют:** Алгоритмическое мышление, понимание linked list cycle detection (Floyd's algorithm), умение сводить задачу к известной.

```csharp
// Решение: Floyd's Tortoise and Hare (Cycle Detection)
public int FindDuplicate(int[] nums)
{
    // Phase 1: Find intersection point
    int slow = nums[0];
    int fast = nums[0];
    
    do
    {
        slow = nums[slow];          // 1 step
        fast = nums[nums[fast]];    // 2 steps
    } while (slow != fast);
    
    // Phase 2: Find entrance to cycle (= duplicate)
    slow = nums[0];
    while (slow != fast)
    {
        slow = nums[slow];
        fast = nums[fast];
    }
    
    return slow;
}

// Пример: [3, 1, 3, 4, 2]
// Индексы: 0→3, 1→1, 2→3, 3→4, 4→2
// Граф: 0→3→4→2→3 (цикл начинается в 3) → ответ 3
```

**Объяснение:** Массив можно представить как связный список, где `nums[i]` — это следующий индекс. Поскольку одно число повторяется, в графе образуется цикл. Вход в цикл = дубликат. Floyd's algorithm находит его за O(n).

**Почему спрашивают:** Проверяет способность увидеть неочевидную аналогию (массив → граф → цикл). Junior пытается отсортировать или использовать HashSet (O(n) память), Senior видит Floyd.

---

## Задача 2: LRU Cache

**Условие:** Реализовать Least Recently Used (LRU) кэш с методами `Get(key)` и `Put(key, value)`. Оба за O(1). При превышении capacity удалять наименее используемый элемент.

```csharp
public class LRUCache
{
    private readonly int _capacity;
    private readonly Dictionary<int, LinkedListNode<(int key, int value)>> _map;
    private readonly LinkedList<(int key, int value)> _list;
    
    public LRUCache(int capacity)
    {
        _capacity = capacity;
        _map = new Dictionary<int, LinkedListNode<(int, int)>>(capacity);
        _list = new LinkedList<(int, int)>();
    }
    
    public int Get(int key)
    {
        if (!_map.TryGetValue(key, out var node))
            return -1;
        
        // Move to front = most recently used
        _list.Remove(node);
        _list.AddFirst(node);
        return node.Value.value;
    }
    
    public void Put(int key, int value)
    {
        if (_map.TryGetValue(key, out var existing))
        {
            // Update existing
            _list.Remove(existing);
        }
        else if (_map.Count >= _capacity)
        {
            // Evict least recently used (back of list)
            var lru = _list.Last!;
            _map.Remove(lru.Value.key);
            _list.RemoveLast();
        }
        
        var newNode = _list.AddFirst((key, value));
        _map[key] = newNode;
    }
}

// Использование:
var cache = new LRUCache(2);
cache.Put(1, 1);
cache.Put(2, 2);
cache.Get(1);       // 1 — теперь 2 наименее используемый
cache.Put(3, 3);    // Удаляет 2 (LRU)
cache.Get(2);       // -1 (удалён)
```

**Почему спрашивают:** Проверяет понимание структур данных (Dictionary + LinkedList), дизайн систем (кэширование), O(1) операции. Часто на System Design интервью.

---

## Задача 3: Пул объектов (Object Pool)

**Условие:** Реализовать thread-safe пул объектов. Методы: `Rent()` (получить объект) и `Return(T obj)` (вернуть). Если объектов нет — создать новый. Максимум пула — N.

```csharp
public class ObjectPool<T> where T : class, new()
{
    private readonly int _maxSize;
    private readonly ConcurrentBag<T> _pool = new();
    private int _count;
    
    public ObjectPool(int maxSize = 10) => _maxSize = maxSize;
    
    public T Rent()
    {
        if (_pool.TryTake(out var item))
        {
            Interlocked.Decrement(ref _count);
            return item;
        }
        return new T();
    }
    
    public void Return(T item)
    {
        if (Interlocked.Increment(ref _count) <= _maxSize)
            _pool.Add(item);
        else
            Interlocked.Decrement(ref _count);  // Откат — пул полон
    }
}
```

**Альтернатива — через `ArrayPool<T>` (built-in):**

```csharp
// .NET встроенный ArrayPool
byte[] buffer = ArrayPool<byte>.Shared.Rent(1024);
try
{
    // Используем buffer[0..1023]
}
finally
{
    ArrayPool<byte>.Shared.Return(buffer);  // Обязательно вернуть!
}
```

**Почему спрашивают:** Проверяет понимание аллокаций, GC pressure, thread-safety, паттернов. На практике — покажите что знаете `ArrayPool<T>` и `ObjectPool<T>` из Microsoft.Extensions.ObjectPool.

---

## Задача 4: Lock-free счётчик хитов

**Условие:** Реализовать потокобезопасный счётчик, который логирует каждые N запросов (сбрасывая счётчик) без блокировок.

```csharp
public class HitCounter
{
    private int _count;
    private readonly int _reportThreshold;
    private readonly Action<int> _reportAction;
    
    public HitCounter(int reportThreshold, Action<int> reportAction)
    {
        _reportThreshold = reportThreshold;
        _reportAction = reportAction;
    }
    
    public void Increment()
    {
        var newCount = Interlocked.Increment(ref _count);
        if (newCount >= _reportThreshold)
        {
            // Только один поток сбрасывает (CAS)
            var oldCount = Interlocked.Exchange(ref _count, 0);
            if (oldCount >= _reportThreshold)  // Гарантия: только первый поток логирует
                _reportAction(oldCount);
        }
    }
}
```

**Почему спрашивают:** `Interlocked` — фундамент lock-free программирования в .NET. Проверяет понимание атомарных операций и race conditions.

---

## Задача 5: Пересечение двух interval-списков

**Условие:** Даны два списка интервалов, отсортированных по началу. Найти их пересечение за O(n+m).

```csharp
public record Interval(int Start, int End);

public List<Interval> Intersect(List<Interval> a, List<Interval> b)
{
    var result = new List<Interval>();
    int i = 0, j = 0;
    
    while (i < a.Count && j < b.Count)
    {
        // Проверяем пересечение
        int start = Math.Max(a[i].Start, b[j].Start);
        int end = Math.Min(a[i].End, b[j].End);
        
        if (start <= end)
            result.Add(new Interval(start, end));
        
        // Двигаем тот, который закончился раньше
        if (a[i].End < b[j].End)
            i++;
        else
            j++;
    }
    
    return result;
}

// Тест
var a = new List<Interval> { new(0, 2), new(5, 10), new(13, 23), new(24, 25) };
var b = new List<Interval> { new(1, 5), new(8, 12), new(15, 24), new(25, 26) };
var result = Intersect(a, b);
// [[1,2], [5,5], [8,10], [15,23], [24,24], [25,25]]
```

**Почему спрашивают:** Проверяет умение работать с двумя указателями (two-pointer technique), обработку краевых случаев.

---

## Задача 6: Преобразование Dictionary с группировкой

**Условие:** Дан `List<Employee>`. Сгруппировать по департаменту и для каждого департамента посчитать: количество сотрудников, среднюю зарплату, максимальную зарплату, список имён.

```csharp
public record Employee(string Name, string Department, decimal Salary);

public record DepartmentStats(
    string Department,
    int Count,
    decimal AvgSalary,
    decimal MaxSalary,
    List<string> Names);

public List<DepartmentStats> AnalyzeByDepartment(List<Employee> employees)
{
    return employees
        .GroupBy(e => e.Department)
        .Select(g => new DepartmentStats(
            Department: g.Key,
            Count: g.Count(),
            AvgSalary: Math.Round(g.Average(e => e.Salary), 2),
            MaxSalary: g.Max(e => e.Salary),
            Names: g.OrderBy(e => e.Name).Select(e => e.Name).ToList()
        ))
        .OrderByDescending(s => s.AvgSalary)
        .ToList();
}
```

**Альтернатива — через `Aggregate` (один проход):**

```csharp
public Dictionary<string, DepartmentStats> AnalyzeOnePass(List<Employee> employees)
{
    return employees.Aggregate(
        new Dictionary<string, DepartmentStats>(),
        (acc, e) =>
        {
            if (acc.TryGetValue(e.Department, out var stats))
            {
                acc[e.Department] = stats with
                {
                    Count = stats.Count + 1,
                    MaxSalary = Math.Max(stats.MaxSalary, e.Salary),
                    AvgSalary = ((stats.AvgSalary * stats.Count) + e.Salary) / (stats.Count + 1),
                    Names = stats.Names.Append(e.Name).OrderBy(n => n).ToList()
                };
            }
            else
            {
                acc[e.Department] = new DepartmentStats(
                    e.Department, 1, e.Salary, e.Salary, [e.Name]);
            }
            return acc;
        });
}
```

**Почему спрашивают:** LINQ — core skill для .NET. Проверяет знание GroupBy, Aggregate, Select. Второе решение показывает понимание сложности (один проход O(n) vs O(n log n)).

---

## Задача 7: Event Bus / Message Broker (упрощённый)

**Условие:** Реализовать простой In-Memory Event Bus с подпиской по типу события.

```csharp
public interface IEvent { }

public record OrderPlaced(Guid OrderId, decimal Total) : IEvent;
public record PaymentReceived(Guid OrderId, decimal Amount) : IEvent;

public class SimpleEventBus
{
    private readonly Dictionary<Type, List<Delegate>> _handlers = new();
    private readonly object _lock = new();
    
    public void Subscribe<T>(Action<T> handler) where T : IEvent
    {
        lock (_lock)
        {
            var type = typeof(T);
            if (!_handlers.ContainsKey(type))
                _handlers[type] = new List<Delegate>();
            _handlers[type].Add(handler);
        }
    }
    
    public void Publish<T>(T @event) where T : IEvent
    {
        List<Delegate> handlers;
        lock (_lock)
        {
            if (!_handlers.TryGetValue(typeof(T), out handlers!))
                return;
            handlers = handlers.ToList();
        }
        
        foreach (var handler in handlers)
        {
            if (handler is Action<T> action)
                action(@event);
        }
    }
}

// Использование:
var bus = new SimpleEventBus();
bus.Subscribe<OrderPlaced>(e => Console.WriteLine($"Order {e.OrderId} placed, total: {e.Total}"));
bus.Subscribe<OrderPlaced>(e => Console.WriteLine($"Second handler: {e.OrderId}"));
bus.Publish(new OrderPlaced(Guid.NewGuid(), 100m));
```

**Почему спрашивают:** Проверяет понимание паттернов Pub/Sub, MediatR-подобных библиотек, type safety с generics, thread-safety.

---

## Задача 8: Rate Limiter Token Bucket

**Условие:** Реализовать Token Bucket rate limiter: N токенов, пополняется со скоростью R токенов/сек. Метод `bool TryConsume()` — true если токен доступен.

```csharp
public class TokenBucket
{
    private readonly int _maxTokens;
    private readonly double _refillRate;  // Токенов в секунду
    private double _currentTokens;
    private DateTime _lastRefill;
    private readonly object _lock = new();
    
    public TokenBucket(int maxTokens, double refillRatePerSecond)
    {
        _maxTokens = maxTokens;
        _refillRate = refillRatePerSecond;
        _currentTokens = maxTokens;
        _lastRefill = DateTime.UtcNow;
    }
    
    public bool TryConsume(int tokens = 1)
    {
        lock (_lock)
        {
            Refill();
            if (_currentTokens >= tokens)
            {
                _currentTokens -= tokens;
                return true;
            }
            return false;
        }
    }
    
    private void Refill()
    {
        var now = DateTime.UtcNow;
        var elapsed = (now - _lastRefill).TotalSeconds;
        _currentTokens = Math.Min(_maxTokens, _currentTokens + elapsed * _refillRate);
        _lastRefill = now;
    }
}

// Использование:
var bucket = new TokenBucket(maxTokens: 100, refillRatePerSecond: 10);
if (bucket.TryConsume())
    Console.WriteLine("Request allowed");
// Позволяет burst до 100, затем 10 в секунду
```

**Почему спрашивают:** Rate limiting — core pattern для API. Проверяет понимание алгоритмов, time-based расчётов, thread-safety.

---

## Чек-лист

- [ ] Floyd's Cycle Detection (задача о дубликате)
- [ ] LRU Cache (Dictionary + LinkedList, O(1) get/put)
- [ ] Object Pool (ArrayPool, ConcurrentBag, Interlocked)
- [ ] Lock-free счётчик (Interlocked.Increment/Exchange, CAS)
- [ ] Two-pointer technique (пересечение интервалов)
- [ ] LINQ GroupBy + Aggregate (анализ данных)
- [ ] Event Bus (Pub/Sub, generics, type safety)
- [ ] Token Bucket (rate limiting, time-based refill)

# .NET Collections — глубокое погружение для собеседования

---

## List\<T\> — внутреннее устройство

**Факты:**
- Внутри — динамический массив `T[]`
- `Capacity` — размер внутреннего массива; `Count` — число элементов
- Growth factor: **×2** до ~2^16 элементов, затем **×1.5**
- При добавлении сверх Capacity — выделение нового массива и копирование всех элементов (O(n) amortized O(1))
- `TrimExcess()` уменьшает Capacity до Count, если Count < 90% Capacity

```csharp
// Упрощённая реализация List<T>.Add + Grow
public class SimplifiedList<T>
{
    private T[] _items = new T[4];
    private int _size;

    public void Add(T item)
    {
        if (_size == _items.Length)
            Grow();
        _items[_size++] = item;
    }

    private void Grow()
    {
        int newCapacity = _items.Length < 0x10000
            ? _items.Length * 2       // ×2 для маленьких
            : _items.Length + _items.Length / 2; // ×1.5 для больших
        T[] newItems = new T[newCapacity];
        Array.Copy(_items, newItems, _size);  // O(n)
        _items = newItems;
    }
}
```

### Сложность операций

| Операция | Средняя | Худшая | Примечание |
|----------|---------|--------|------------|
| `Add()` | O(1) | O(n) | Amortized O(1); O(n) при расширении |
| `Insert(index)` | O(n) | O(n) | Сдвиг всех элементов после index |
| `RemoveAt(index)` | O(n) | O(n) | Сдвиг всех элементов после index |
| `Remove(item)` | O(n) | O(n) | Поиск O(n) + сдвиг O(n) |
| `this[index]` | O(1) | O(1) | Прямой доступ к массиву |
| `Contains(item)` | O(n) | O(n) | Линейный поиск |
| `IndexOf(item)` | O(n) | O(n) | Линейный поиск |
| `Sort()` | O(n log n) | O(n log n) | IntroSort (QuickSort + HeapSort + InsertionSort) |

**Факт:** `List<T>.Sort()` использует **IntroSort** — гибрид QuickSort, HeapSort (если глубина рекурсии > 2*log₂n) и InsertionSort (для подмассивов < 16 элементов).

### Память и GC давление

```csharp
// List<int> с Capacity=1000 содержит:
// - Сам объект List: ~24 байта (object header + method table + fields)
// - Внутренний int[]: ~8 + 4*1000 = 4008 байт
// - Если Capacity=4 а Count=0: внутренний массив = 8 + 4*4 = 24 байта

// Проблема: после AddRange(1000) и Clear() массив всё ещё 4008 байт
list.TrimExcess(); // => уменьшает до 0 (если Count=0)
```

---

## Dictionary\<TKey, TValue\> — хеш-таблица

**Факты:**
- Внутри — массив `Entry[]` (структура с key, value, hashCode, next)
- Коллизии разрешаются методом **цепочек** (separate chaining) через поле `next` — связный список внутри того же массива
- `Buckets` — массив индексов (int[]) на первые Entry в цепочке
- Load factor ~0.72 — при достижении resize (удвоение)
- `EqualityComparer<TKey>.Default` для хеширования и сравнения

### Внутренняя структура Entry

```csharp
// Упрощённо: что внутри Dictionary
internal struct Entry<TKey, TValue>
{
    public uint hashCode;   // Хеш ключа (младшие 31 бит + 1 бит признака)
    public int next;        // Индекс следующей Entry в цепочке (или -1)
    public TKey key;
    public TValue value;
}

// Структура Dictionary (упрощённо)
public class SimplifiedDict<TKey, TValue>
{
    private int[] _buckets;      // bucket[i] = индекс первой Entry в цепочке
    private Entry[] _entries;    // Все записи, включая "свободные"
    private int _count;          // Число живых записей
    private int _freeList;       // Индекс начала списка свободных Entry
    private int _freeCount;      // Число свободных Entry (после Remove)
}
```

### Алгоритм поиска (TryGetValue)

```
TryGetValue(key):
  1. Вычислить hashCode = key.GetHashCode() & 0x7FFFFFFF
  2. Определить bucket = hashCode % buckets.Length
  3. Пройти по цепочке: i = buckets[bucket];
     while (i >= 0):
       Entry e = entries[i];
       if e.hashCode == hashCode && Equals(e.key, key)
         return e.value;
       i = e.next;
  4. return false;
```

### Сложность операций

| Операция | Средняя | Худшая | Примечание |
|----------|---------|--------|------------|
| `Add()` | O(1) | O(n) | O(n) при resize; O(n) при плохом хеше |
| `TryGetValue()` | O(1) | O(n) | O(n) при коллизиях всех ключей |
| `Remove()` | O(1) | O(n) | O(n) при коллизиях |
| `ContainsKey()` | O(1) | O(n) | Аналогично TryGetValue |
| `this[key]` get/set | O(1) | O(n) | |

**Факт:** В худшем случае (все ключи имеют одинаковый хеш) Dictionary вырождается в связный список — O(n) на все операции. Это важно на собеседовании: **никогда не используйте GetHashCode, возвращающий константу**.

### Удаление и FreeList

```csharp
// Remove НЕ сдвигает массив — помечает Entry как свободную
// Свободные Entry связываются через поле next в список freeList
dict.Remove(key1);
dict.Remove(key2);
// После двух удалений: _freeCount=2, _freeList указывает на последнюю удалённую
// При Add: сначала проверяется freeList — переиспользование свободного слота
dict.Add(key3, value3);  // Займёт слот от key2 (LIFO из freeList)
```

**Факт:** При частых Add/Remove Dictionary не дефрагментируется — `_count` уменьшается, но `_entries.Length` не меняется. Для освобождения памяти нужно создать новый Dictionary.

### Резюме для собеседования по Dictionary

```
Q: Когда Dictionary хуже List?
A: При малом числе элементов (< ~10) List с линейным поиском быстрее
   из-за отсутствия вычисления хеша и отсутствия накладных расходов

Q: Что такое коллизия и как .NET её решает?
A: Коллизия — два разных ключа дают одинаковый bucket. .NET использует
   separate chaining через поле next в структуре Entry

Q: Почему важно переопределять GetHashCode вместе с Equals?
A: Если два объекта равны через Equals, они ДОЛЖНЫ возвращать одинаковый
   GetHashCode. Иначе Dictionary не найдёт ключ.
```

---

## HashSet\<T\> — множество

**Факты:**
- Внутренне **аналогичен Dictionary\<T, object\>** — тот же массив Entry[]
- Value всегда специальный объект-маркер
- Те же характеристики сложности
- `Add()` возвращает `bool` (true если элемент был добавлен)
- `IntersectWith`, `UnionWith`, `ExceptWith` — O(n) операции над множествами

```csharp
// HashSet — это Dictionary без Value
// Slot = Entry с полями hashCode, next, value (T)
// Value хранит сам элемент

var set = new HashSet<int>();
set.Add(1);  // true
set.Add(1);  // false — уже есть
```

---

## SortedDictionary\<TKey, TValue\> и SortedSet\<T\>

**Факты:**
- Внутри — **красно-чёрное дерево** (Red-Black Tree)
- Гарантирует O(log n) на вставку, удаление, поиск
- Элементы всегда отсортированы по ключу
- Использует `IComparer<T>` вместо `IEqualityComparer<T>`

| Операция | SortedDictionary | Dictionary |
|----------|:---:|:---:|
| Insert | O(log n) | O(1)* |
| Search | O(log n) | O(1)* |
| Delete | O(log n) | O(1)* |
| Min/Max | O(log n) | O(n) |
| Range query | O(log n + k) | O(n) |

**Когда использовать:**
- Нужен перебор в отсортированном порядке
- Нужны операции диапазона (Floor, Ceiling, Range)
- Размер коллекции большой и O(log n) приемлемо

```csharp
var sorted = new SortedSet<int> { 5, 2, 8, 1 };
foreach (var x in sorted) Console.Write(x + " "); // 1 2 5 8 — всегда отсортирован

// Операции, которых нет в HashSet:
sorted.GetViewBetween(2, 5);  // {2, 5} — подмножество за O(log n)
sorted.Min;                   // 1 — O(log n)
sorted.Max;                   // 8 — O(log n)
```

---

## LinkedList\<T\> — двусвязный список

**Факты:**
- Каждый узел (`LinkedListNode<T>`) содержит ссылки на Next и Previous
- Вставка/удаление в середине — O(1) **если есть ссылка на узел**
- Поиск по индексу/значению — O(n)
- Нет прямого доступа по индексу (`ElementAt` — O(n))

```csharp
var list = new LinkedList<int>();
var node = list.AddLast(42);    // O(1)
list.AddAfter(node, 100);       // O(1) — вставка после известного узла
list.Remove(node);              // O(1) — удаление известного узла

// НО: поиск узла — O(n)
var found = list.Find(42);      // O(n)
```

| Операция | List\<T\> | LinkedList\<T\> |
|----------|:---:|:---:|
| Add (конец) | O(1)* | O(1) |
| Insert (середина) | O(n) | O(1)** |
| Remove (середина) | O(n) | O(1)** |
| Index access | O(1) | O(n) |
| Cache locality | Отличная | Плохая |

\* Amortized; \** При наличии ссылки на LinkedListNode

---

## Потокобезопасные коллекции

### ConcurrentDictionary\<TKey, TValue\>

**Факты:**
- Внутри — массив **lock-стрипов** (tables + locks), каждый защищает свою группу buckets
- Read-операции **lock-free** (чтение через Volatile.Read)
- Write-операции используют fine-grained locking (только свой bucket)
- `GetOrAdd` и `AddOrUpdate` — атомарные фабрики
- Не использует DictionarySlim под капотом — полностью собственная реализация

```csharp
var dict = new ConcurrentDictionary<string, int>();

// Атомарные операции:
dict.TryAdd("key", 1);                        // true если ключа не было
dict.GetOrAdd("key", k => ComputeValue(k));   // Вызов фабрики ТОЛЬКО если ключа нет
dict.AddOrUpdate("key", 1, (k, old) => old + 1); // Инкремент атомарно

// Перебор — snapshot (не блокирует, но может содержать устаревшие данные)
foreach (var kvp in dict) { /* safe, weak consistency */ }
```

**Факт:** `ConcurrentDictionary` не гарантирует линейную согласованность при итерации — вы можете увидеть состояние словаря на момент **начала** итерации, но не на момент обработки конкретного элемента.

### ConcurrentQueue\<T\>

**Факты:**
- Lock-free MPSC (multiple producer, single consumer) очередь
- Внутри — связный список сегментов (массивов фиксированного размера, по умолчанию 32 элемента)
- `Enqueue` и `TryDequeue` — lock-free через Interlocked операции
- Подходит для producer-consumer сценариев

```csharp
var q = new ConcurrentQueue<int>();
q.Enqueue(1);              // Lock-free
q.Enqueue(2);
if (q.TryDequeue(out int x))  // Lock-free
    Console.WriteLine(x);     // 1
```

### ConcurrentBag\<T\>

**Факты:**
- Локальные списки для каждого потока (ThreadLocal)
- `Add` — O(1), добавляет в локальный список текущего потока
- `TryTake` — сначала ищет в локальном списке, затем "ворует" у других потоков (work-stealing)
- **Не гарантирует FIFO порядок!**

### Immutable Collections (System.Collections.Immutable)

**Факты:**
- **Persistent data structures** — при изменении создаётся новая версия, старая неизменна
- Внутри — **AVL-деревья** или префиксные деревья (в зависимости от коллекции)
- Потокобезопасны по определению (read-only + structural sharing)
- Цена: O(log n) вместо O(1), overhead по памяти

```csharp
using System.Collections.Immutable;

var list = ImmutableList<int>.Empty;
list = list.Add(1);    // Новый список, старый не изменён
list = list.Add(2);    // Структурное разделение: новый "узел" ссылается на старый

var old = list;
list = list.Add(3);
// old всё ещё [1, 2]; list = [1, 2, 3]
```

---

## Чек-лист для собеседования: выбор коллекции

| Сценарий | Коллекция | Причина |
|----------|-----------|---------|
| Частое чтение по ключу | `Dictionary<TK,TV>` | O(1) среднее |
| Упорядоченный перебор + поиск | `SortedDictionary<TK,TV>` | O(log n), сортировка гарантирована |
| Только проверка наличия | `HashSet<T>` | Легче Dictionary, O(1) |
| Последовательное добавление в конец | `List<T>` | Amortized O(1), cache-friendly |
| Частая вставка/удаление в середине | `LinkedList<T>` | O(1) если есть ссылка на узел |
| Потокобезопасное чтение/запись | `ConcurrentDictionary<TK,TV>` | Lock-free reads + fine-grained locks |
| Producer-consumer (потокобезопасно) | `ConcurrentQueue<T>` / `Channel<T>` | Lock-free, оптимизировано |
| Многопоточный стек | `ConcurrentStack<T>` | Lock-free |
| Неизменяемые данные (функциональный стиль) | `ImmutableList<T>` / `ImmutableDictionary` | Persistent, structural sharing |
| Высокопроизводительные сценарии без GC | `ArrayPool<T>` + `Span<T>` | Allocation-free |

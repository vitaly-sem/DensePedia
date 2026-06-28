# Внутреннее устройство коллекций .NET

---

## List\<T\>

**Факты:**
- Внутри — массив `T[]` с расширением
- `Capacity` — размер внутреннего массива; `Count` — количество элементов
- При добавлении сверх Capacity — **удвоение** массива (копирование всех элементов O(n))
- Вставка в середину — O(n) (сдвиг элементов)
- Удаление с конца — O(1), с начала/середины — O(n)

```csharp
// Внутреннее устройство (упрощённо):
public class CustomList<T>
{
    private T[] _items = new T[4];  // Default capacity = 4
    private int _size;
    
    public void Add(T item)
    {
        if (_size == _items.Length)
        {
            // Growth factor = 2x (для больших списков меньше)
            var newItems = new T[_items.Length * 2];
            Array.Copy(_items, newItems, _size);
            _items = newItems;
        }
        _items[_size++] = item;
    }
    
    public void Insert(int index, T item)
    {
        if (_size == _items.Length) Grow();
        Array.Copy(_items, index, _items, index + 1, _size - index);  // Shift
        _items[index] = item;
        _size++;
    }
    
    public bool Remove(T item)
    {
        int index = IndexOf(item);
        if (index < 0) return false;
        RemoveAt(index);
        return true;
    }
    
    public void RemoveAt(int index)
    {
        _size--;
        Array.Copy(_items, index + 1, _items, index, _size - index);  // Shift left
        _items[_size] = default;  // Release reference for GC
    }
}
```

**Факт:** Growth factor = 2 для списков до ~2^16 элементов, затем 1.5 (чтобы не аллоцировать слишком много лишней памяти).

**Факт:** `TrimExcess()` — уменьшает Capacity до Count (если Count < 90% Capacity). Полезно после массового добавления.

**Факт:** `List<T>` не thread-safe. Для многопоточности — `ConcurrentQueue<T>` или lock.

---

## Dictionary\<TKey, TValue\>

**Факты:**
- Внутри — **hash table** с **open addressing** (с 2017: actual implementation uses separate chaining with buckets)
- На самом деле: массив **buckets** + массив **entries**
- Каждый entry содержит: hashCode, key, value, next (индекс следующего в цепочке)

```csharp
// Внутреннее устройство (упрощённо):
internal struct Entry<TKey, TValue>
{
    public int HashCode;    // Lower 31 bits of hash
    public int Next;        // Index of next entry in chain (-1 = end)
    public TKey Key;
    public TValue Value;
}

// При добавлении:
// 1. Вычисляем hash(key) → bucketIndex = hash % buckets.Length
// 2. Проверяем цепочку коллизий (entries[bucketIndex] → Next → ...)
// 3. Если ключ найден — replace value
// 4. Если не найден — добавляем в конец entries
// 5. Если load factor > 0.72 — resize (удвоение)

// Коллизии — resolution через цепочки (chaining)
// В .NET Core 3.0+ — используется int[] buckets (индекс первого entry в цепочке)
// Сложность: O(1) average, O(n) worst (если все ключи в один bucket)
```

**Факт:** Load factor = 0.72 (72%). При превышении — capacity удваивается + **rehash** всех элементов.

**Факт:** Хеш-функция для `string` — randomized per process (.NET Core 2.1+). Защита от HashDoS атак.

**Факт:** Порядок элементов в Dictionary **не гарантирован** и меняется при rehash. Для упорядоченного — `SortedDictionary<TKey, TValue>` или `OrderedDictionary` (.NET 9+).

**Факт:** `TryGetValue` быстрее `ContainsKey` + `[]` — один lookup вместо двух.

```csharp
// ❌ Два lookup
if (dict.ContainsKey(key))
    var value = dict[key];

// ✅ Один lookup
if (dict.TryGetValue(key, out var value))
    // use value
```

---

## HashSet\<T\>

**Факты:**
- Аналогичен Dictionary, но только ключи (без значений)
- Внутри — тот же механизм: buckets + slots
- `Add()` — возвращает `false`, если элемент уже существует
- `UnionWith`, `IntersectWith`, `ExceptWith`, `SymmetricExceptWith` — set operations O(n)

```csharp
var set = new HashSet<int> { 1, 2, 3, 4, 5 };
var other = new HashSet<int> { 4, 5, 6, 7 };

set.UnionWith(other);                  // { 1, 2, 3, 4, 5, 6, 7 }
set.IntersectWith(other);              // { 4, 5 }
set.ExceptWith(other);                 // { 1, 2, 3 }
set.SymmetricExceptWith(other);        // { 1, 2, 3, 6, 7 }

// IsSubsetOf, IsSupersetOf, Overlaps
if (set.IsSubsetOf(other)) { }
```

**Факт:** Кастомный `IEqualityComparer<T>` для HashSet — определите `GetHashCode()` правильно. Плохой hash → O(n) производительность.

---

## SortedSet\<T\> и SortedDictionary\<T\>

**Факты:**
- Внутри — **Kаждый узел** = Red-Black Tree
- Add/Remove/Contains — O(log n)
- In-order traversal = sorted order
- `Min` / `Max` — O(1) (левый/правый узел)
- `GetViewBetween(min, max)` — O(log n) возвращает поддерево (subtree view)

```csharp
var sorted = new SortedSet<int> { 5, 1, 3, 2, 4 };
// Всегда отсортировано: { 1, 2, 3, 4, 5 }

// Range query
var range = sorted.GetViewBetween(2, 4);  // { 2, 3, 4 }

// Reverse
foreach (var item in sorted.Reverse()) { }
```

**Факт:** `SortedSet<T>.GetViewBetween()` — O(log n), не копирует данные (view).

**Факт:** `SortedDictionary<TKey, TValue>` — надстройка над `SortedSet<KeyValuePair<TKey, TValue>>`.

---

## Stack\<T\> и Queue\<T\>

| | Stack (LIFO) | Queue (FIFO) |
|---|---|---|
| **Внутри** | Массив `T[]`, head pointer | Circular buffer `T[]`, head + tail |
| **Push / Enqueue** | `_items[_size++] = item` | `_items[_tail++] = item` (wrap) |
| **Pop / Dequeue** | `return _items[--_size]` | `return _items[_head++]` (wrap) |
| **Growth** | 2x как List | 2x как List |
| **Когда** | DFS, undo, parsing | BFS, producer-consumer, buffers |

```csharp
// Queue — circular buffer
// Enqueue(1): [0]  [1]  [ ]  [ ]
//              ↑head    ↑tail
// Enqueue(2): [0]  [1]  [2]  [ ]
//              ↑head         ↑tail
// Dequeue:    [0]  [1]  [2]  [ ]
//                   ↑head    ↑tail
// Enqueue(3): [3]  [1]  [2]  [ ]   ← wrap!
//              ↑tail  ↑head
```

**Факт:** Queue использует **circular buffer** — при Dequeue head движется вправо, при Enqueue tail тоже. При достижении конца — wrap вокруг (tail = 0). Это O(1) для Enqueue и Dequeue.

---

## LinkedList\<T\>

**Факты:**
- Двусвязный список: каждый node имеет Previous + Next
- Вставка/удаление — O(1) **если есть ссылка на node**
- Поиск по значению — O(n) (линейный)
- Нет индексатора `[]`
- **Memory overhead:** каждый элемент = Node<T> (object + 2 references) ≈ 24+ bytes overhead

| Операция | List\<T\> | LinkedList\<T\> |
|---|---|---|
| Добавить в конец | O(1) amortized | O(1) |
| Вставить в начало | O(n) | O(1) |
| Вставить в середину | O(n) | O(1) (с ссылкой) |
| Удалить из начала | O(n) | O(1) |
| Доступ по индексу | O(1) | O(n) |
| Memory | Меньше | Больше (Node) |

**Факт:** `LinkedList<T>` почти всегда медленнее `List<T>` на практике из-за cache locality. Массив — последовательный в памяти, связный список — рандомный доступ (cache misses).

---

## Чек-лист

- [ ] List<T>: array-based, growth factor 2x→1.5x, O(1) add, O(n) insert/remove
- [ ] Dictionary<T>: hash table, load factor 0.72, collision chaining, rehash
- [ ] HashSet<T>: set operations (Union, Intersect, Except), O(1) contains
- [ ] SortedSet/SortedDictionary: Red-Black Tree, O(log n), GetViewBetween
- [ ] Queue: circular buffer, O(1) Enqueue/Dequeue
- [ ] LinkedList: двусвязный, O(1) insert/delete с ссылкой, cache misses

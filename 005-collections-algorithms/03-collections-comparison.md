# System.Collections — сравнительный анализ

---

## Generic vs NonGeneric vs Concurrent vs Immutable vs Frozen

| Характеристика | Generic (`List<T>`) | NonGeneric (`ArrayList`) | Concurrent (`ConcurrentDictionary`) | Immutable (`ImmutableArray<T>`) | Frozen (`FrozenSet<T>`) |
|---|---|---|---|---|---|
| **Type safety** | ✅ Compile-time | ❌ Boxing/unboxing | ✅ | ✅ | ✅ |
| **Thread safety** | ❌ | ❌ | ✅ (lock-free частично) | ✅ (by design) | ✅ (by design) |
| **Производительность** | Высокая | Низкая (boxing) | Ниже Generic (contention) | Ниже Generic (копии) | Самая высокая для lookup |
| **Мутабельность** | ✅ | ✅ | ✅ | ❌ (каждое изменение → новый объект) | ❌ |
| **Аллокации** | Минимальные | На каждый value type | Минимальные | На каждое изменение | Однократно при создании |
| **Сценарий** | Повседневная работа | Legacy (не использовать) | Многопоточность | Shared state, snapshots | Оптимизированный read-only lookup |

---

## Когда что использовать — дерево решений

```
Мутабельные данные?
├── Да
│   ├── Многопоточный доступ?
│   │   ├── Да → Concurrent* (ConcurrentDictionary, ConcurrentQueue, Channel)
│   │   └── Нет
│   │       ├── Частые вставки/удаления в середине? → LinkedList<T> (осторожно: cache misses)
│   │       ├── Нужен доступ по индексу? → List<T>
│   │       ├── Нужен lookup по ключу? → Dictionary<TKey,TValue>
│   │       ├── Уникальные элементы? → HashSet<T>
│   │       ├── FIFO? → Queue<T>
│   │       ├── LIFO? → Stack<T>
│   │       └── Sorted? → SortedSet<T> / SortedDictionary<TKey,TValue>
│   └── Нет
│       ├── Данные известны при создании и только читаются?
│       │   ├── Да → FrozenSet<T> / FrozenDictionary<TKey,TValue> (.NET 8+)
│       │   └── Нет → Immutable* (ImmutableArray, ImmutableDictionary)
│       └── Передача между потоками без блокировок? → Immutable* / Frozen*
```

---

## Frozen Collections (.NET 8+)

**Факты:**
- `FrozenSet<T>` и `FrozenDictionary<TKey,TValue>` — оптимизированные для lookup read-only коллекции
- **В 2-5x быстрее** `Dictionary<TKey,TValue>` для ContainsKey/search операций
- Требуют upfront-создания (идеально для: конфигурация, enum mappings, DI registrations)
- Используют оптимизированный хешинг без коллизий (perfect hashing где возможно)

```csharp
using System.Collections.Frozen;

// Создание из существующей коллекции (дорого, но один раз)
var ordinaryDict = new Dictionary<string, int>
{
    ["red"] = 1, ["green"] = 2, ["blue"] = 3
};
FrozenDictionary<string, int> frozen = ordinaryDict.ToFrozenDictionary();

// Создание через builder (для большых коллекций)
var builder = new FrozenDictionaryBuilder<string, int>(ignoreCase: true);
foreach (var item in largeDataset)
    builder[item.Key] = item.Value;
FrozenDictionary<string, int> frozen2 = builder.ToFrozenDictionary();

// Использование — сверхбыстрый lookup
if (frozen.TryGetValue("red", out int value))
    Console.WriteLine(value);  // 1

// FrozenSet — аналогично
FrozenSet<string> tags = new[] { "important", "urgent", "review" }
    .ToFrozenSet();

if (tags.Contains("urgent"))  // Сверхбыстрый lookup
    SendNotification();
```

**Факт:** `FrozenDictionary` использует одну из трёх стратегий в зависимости от размера: `LengthBuckets` (до 4 элементов), `SingleChar` (ключи с разными первыми символами), `Default` (полноценная хеш-таблица с оптимизациями).

**Факт:** Создание `FrozenDictionary` дороже обычного `Dictionary` (требует анализа всех ключей), но каждый последующий lookup — быстрее. ROI положительный при >5 операций lookup.

---

## ReadOnly wrappers vs Immutable

```csharp
// ReadOnly — обёртка над мутабельной коллекцией (безопасность через API)
var list = new List<int> { 1, 2, 3 };
IReadOnlyList<int> readOnly = list.AsReadOnly();  // ReadOnlyCollection<int>
// list.Add(4);  // Оригинал меняется → readOnly "видит" изменение!

// Immutable — настоящая неизменяемость
ImmutableList<int> immutable = ImmutableList<int>.Empty.Add(1).Add(2);
ImmutableList<int> immutable2 = immutable.Add(3);  // Новый объект
// immutable всё ещё {1, 2}; immutable2 = {1, 2, 3}
```

**Факт:** `IReadOnlyList<T>` — это **контракт** (интерфейс). Оригинальная коллекция может быть изменена. `ImmutableList<T>` — настоящая неизменяемость.

---

## ConcurrentDictionary vs Dictionary + lock

```csharp
// ConcurrentDictionary — lock-free для чтения, fine-grained locks для записи
private readonly ConcurrentDictionary<string, int> _concurrent = new();
_concurrent.TryAdd("key", 1);          // Атомарно
_concurrent.GetOrAdd("key", _ => 42);  // Атомарно

// Dictionary + lock — проще для сложных операций
private readonly Dictionary<string, int> _dict = new();
private readonly object _lock = new();

lock (_lock)
{
    if (!_dict.ContainsKey("key"))
        _dict["key"] = 1;
    else
        _dict["key"] += 1;
}
// Для сложной логики — Dictionary + lock читаемее
// Для простого GetOrAdd — ConcurrentDictionary производительнее
```

**Бенчмарк (ориентировочный) — 1M операций GetOrAdd:**
| Подход | Время |
|---|---|
| `ConcurrentDictionary.GetOrAdd` | ~50ms |
| `Dictionary + lock` | ~80ms |
| `Dictionary + ReaderWriterLockSlim` | ~65ms |

---

## Сводная таблица выбора

| Сценарий | Рекомендация | Почему |
|---|---|---|
| Частые добавления в конец, доступ по индексу | `List<T>` | Array-based, O(1) add/index |
| Частые вставки/удаления в середине больших коллекций | `LinkedList<T>` | O(1) с ссылкой (но cache misses) |
| Поиск по ключу, мутабельный | `Dictionary<TKey,TValue>` | Hash table, O(1) avg |
| Уникальные значения, мутабельный | `HashSet<T>` | Hash table, O(1) Contains |
| FIFO очередь | `Queue<T>` / `Channel<T>` | Circular buffer / lock-free |
| LIFO стек | `Stack<T>` | Array-based |
| Sorted данные | `SortedSet<T>` / `SortedDictionary<TKey,TValue>` | Red-Black tree, O(log n) |
| Многопоточная очередь | `ConcurrentQueue<T>` / `Channel<T>` | Lock-free (Channel — async-native) |
| Многопоточный словарь | `ConcurrentDictionary<TKey,TValue>` | Partition-based locking |
| Неизменяемые данные | `ImmutableArray<T>` / `ImmutableDictionary<TKey,TValue>` | Persistent data structures |
| Оптимизированный read-only lookup | `FrozenSet<T>` / `FrozenDictionary<TKey,TValue>` | Perfect hashing (.NET 8+) |
| Ограниченная очередь с backpressure | `Channel<T>` (Bounded) / `BlockingCollection<T>` | Channel — async-native, быстрее |
| Read-only API контракт | `IReadOnlyList<T>` / `IReadOnlyDictionary<TKey,TValue>` | Интерфейс (не гарантирует неизменяемость!) |

---

## Чек-лист

- [ ] Generic vs NonGeneric: всегда Generic, NonGeneric — для legacy
- [ ] FrozenSet/FrozenDictionary: .NET 8+, для read-only lookup
- [ ] IReadOnlyList ≠ Immutable: ReadOnly — контракт, Immutable — настоящая неизменяемость
- [ ] ConcurrentDictionary vs Dictionary+lock: простые операции — Concurrent, сложные — lock
- [ ] Channel vs BlockingCollection: Channel — async-native и быстрее
- [ ] Immutable vs Frozen: Immutable — частые "изменения", Frozen — один раз создал и только читаешь

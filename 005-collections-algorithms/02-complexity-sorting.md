# Оценка сложности и сортировка

---

## Big O Notation — таблица

| Обозначение | Название | Пример |
|---|---|---|
| O(1) | Константная | Array access, hash lookup |
| O(log n) | Логарифмическая | Binary search, Red-Black tree |
| O(n) | Линейная | List iteration, linear search |
| O(n log n) | Линейно-логарифмическая | QuickSort, MergeSort, HeapSort |
| O(n²) | Квадратичная | BubbleSort, InsertionSort, nested loops |
| O(2ⁿ) | Экспоненциальная | Fibonacci naive, subset generation |
| O(n!) | Факториальная | Permutations, traveling salesman |

**Факт:** Dictionary<T>.ContainsKey() — O(1) average, O(n) worst (все ключи collided).

**Факт:** List<T>.Contains() — O(n). Если нужно часто проверять наличие — используйте HashSet<T>.

---

## Сложность операций с коллекциями

| Коллекция | Add | Insert | Remove | Search | Index |
|---|---|---|---|---|---|
| `T[]` | — | O(n) | O(n) | O(n) | O(1) |
| `List<T>` | O(1)* | O(n) | O(n) | O(n) | O(1) |
| `LinkedList<T>` | O(1) | O(1)** | O(1)** | O(n) | O(n) |
| `HashSet<T>` | O(1)* | — | O(1) | O(1) | — |
| `SortedSet<T>` | O(log n) | — | O(log n) | O(log n) | — |
| `Dictionary<T>` | O(1)* | — | O(1) | O(1) | — |
| `SortedDictionary<T>` | O(log n) | — | O(log n) | O(log n) | — |
| `Stack<T>` | O(1) | — | O(1) | O(n) | — |
| `Queue<T>` | O(1) | — | O(1) | O(n) | — |

\* amortized (может быть O(n) при resize)
\** при наличии ссылки на узел

---

## Сортировка в .NET

**Факты:**
- `List<T>.Sort()` использует **introsort** (quick sort + heap sort при глубокой рекурсии)
- `Array.Sort()` — introsort (для примитивов — специализированная)
- `LINQ .OrderBy()` — **stable sort** (quicksort стабильная версия), O(n log n)
- `List<T>.Sort()` — **не stable** (не сохраняет порядок равных элементов)

```csharp
var list = new List<int> { 5, 3, 1, 4, 2 };

// 1. In-place sort (introsort, not stable)
list.Sort();

// 2. Custom comparer
list.Sort((a, b) => b.CompareTo(a));  // Descending

// 3. LINQ OrderBy (stable, creates new collection)
var sorted = list.OrderBy(x => x).ToList();

// 4. OrderBy with ThenBy (multi-key)
var people = people
    .OrderBy(p => p.LastName)    // Primary key
    .ThenBy(p => p.FirstName)    // Secondary key
    .ToList();

// 5. Reverse
list.Reverse();  // In-place

// 6. BinarySearch (O(log n)) — ТОЛЬКО если отсортирован!
int index = list.BinarySearch(3);  // Returns index or bitwise complement
if (index < 0)
    list.Insert(~index, 3);  // Insert at correct position
```

---

## Поиск — алгоритмы

### Binary Search

```csharp
// Binary search — O(log n), требует отсортированный массив
public static int BinarySearch<T>(T[] sorted, T target) where T : IComparable<T>
{
    int left = 0, right = sorted.Length - 1;
    
    while (left <= right)
    {
        int mid = left + (right - left) / 2;  // Предотвращение overflow
        int cmp = sorted[mid].CompareTo(target);
        
        if (cmp == 0) return mid;
        if (cmp < 0) left = mid + 1;
        else right = mid - 1;
    }
    
    return ~left;  // Bitwise complement = insertion point
}

// LowerBound — first index >= target
public static int LowerBound<T>(T[] sorted, T target) where T : IComparable<T>
{
    int left = 0, right = sorted.Length;
    while (left < right)
    {
        int mid = left + (right - left) / 2;
        if (sorted[mid].CompareTo(target) < 0)
            left = mid + 1;
        else
            right = mid;
    }
    return left;
}

// UpperBound — first index > target
public static int UpperBound<T>(T[] sorted, T target) where T : IComparable<T>
    => LowerBound(sorted, target) + 1;
```

---

## Сортировка — реализация алгоритмов

### QuickSort (средний O(n log n), worst O(n²))

```csharp
public static void QuickSort<T>(T[] arr, int left, int right) where T : IComparable<T>
{
    if (left >= right) return;
    
    int pivot = Partition(arr, left, right);
    QuickSort(arr, left, pivot - 1);
    QuickSort(arr, pivot + 1, right);
}

private static int Partition<T>(T[] arr, int left, int right) where T : IComparable<T>
{
    // Pivot = median of three (left, mid, right) — предотвращает worst case
    int mid = left + (right - left) / 2;
    T pivot = MedianOfThree(arr[left], arr[mid], arr[right]);
    
    int i = left - 1;
    for (int j = left; j < right; j++)
    {
        if (arr[j].CompareTo(pivot) <= 0)
        {
            i++;
            (arr[i], arr[j]) = (arr[j], arr[i]);
        }
    }
    (arr[i + 1], arr[right]) = (arr[right], arr[i + 1]);
    return i + 1;
}
```

### MergeSort (O(n log n), stable)

```csharp
public static T[] MergeSort<T>(T[] arr) where T : IComparable<T>
{
    if (arr.Length <= 1) return arr;
    
    int mid = arr.Length / 2;
    var left = MergeSort(arr[..mid]);
    var right = MergeSort(arr[mid..]);
    
    return Merge(left, right);
}

private static T[] Merge<T>(T[] left, T[] right) where T : IComparable<T>
{
    var result = new T[left.Length + right.Length];
    int i = 0, j = 0, k = 0;
    
    while (i < left.Length && j < right.Length)
        result[k++] = left[i].CompareTo(right[j]) <= 0 ? left[i++] : right[j++];
    
    while (i < left.Length) result[k++] = left[i++];
    while (j < right.Length) result[k++] = right[j++];
    
    return result;
}
```

---

## Другие алгоритмы для интервью

### Два указателя (Two Pointers)

```csharp
// Найти пару чисел, сумма которых = target (в отсортированном массиве)
public static (int, int)? TwoSum(int[] sorted, int target)
{
    int left = 0, right = sorted.Length - 1;
    
    while (left < right)
    {
        int sum = sorted[left] + sorted[right];
        
        if (sum == target) return (left, right);
        if (sum < target) left++;
        else right--;
    }
    
    return null;
}
```

### Скользящее окно (Sliding Window)

```csharp
// Максимальная сумма подмассива длины k
public static int MaxSubarraySum(int[] arr, int k)
{
    int windowSum = 0;
    for (int i = 0; i < k; i++)
        windowSum += arr[i];
    
    int maxSum = windowSum;
    for (int i = k; i < arr.Length; i++)
    {
        windowSum = windowSum - arr[i - k] + arr[i];  // Slide window
        maxSum = Math.Max(maxSum, windowSum);
    }
    
    return maxSum;
}
```

### Рекурсия vs Итерация

```csharp
// Fibonacci — рекурсия (O(2ⁿ))
static int FibRec(int n) => n <= 1 ? n : FibRec(n - 1) + FibRec(n - 2);

// Fibonacci — мемоизация (O(n))
static int FibMemo(int n, Dictionary<int, int>? memo = null)
{
    memo ??= new();
    if (memo.ContainsKey(n)) return memo[n];
    return memo[n] = n <= 1 ? n : FibMemo(n - 1, memo) + FibMemo(n - 2, memo);
}

// Fibonacci — итерация (O(n), O(1) space)
static int FibIter(int n)
{
    if (n <= 1) return n;
    int a = 0, b = 1;
    for (int i = 2; i <= n; i++)
        (a, b) = (b, a + b);
    return b;
}
```

---

## Чек-лист

- [ ] Big O: O(1), O(log n), O(n), O(n log n), O(n²), O(2ⁿ), O(n!)
- [ ] Сложность операций всех коллекций
- [ ] Sort в .NET: introsort (not stable), LINQ OrderBy (stable)
- [ ] BinarySearch: только для отсортированных, LowerBound/UpperBound
- [ ] QuickSort: median of three, worst case O(n²)
- [ ] MergeSort: O(n log n) stable, extra memory O(n)
- [ ] Два указателя, скользящее окно
- [ ] Рекурсия vs итерация vs мемоизация

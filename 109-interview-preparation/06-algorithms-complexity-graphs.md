# Алгоритмы, сложность и графы — для собеседования

---

## Big O — база

### Классы сложности (от быстрых к медленным)

| Нотация | Название | n=100 | n=1M | Пример |
|---------|----------|-------|------|--------|
| O(1) | Константная | 1 | 1 | Доступ по индексу, хеш-таблица (средний) |
| O(log n) | Логарифмическая | 7 | 20 | Бинарный поиск, сбалансированное дерево |
| O(n) | Линейная | 100 | 1M | Линейный поиск, обход списка |
| O(n log n) | Квазилинейная | 664 | 20M | Быстрая сортировка, MergeSort |
| O(n²) | Квадратичная | 10K | 10¹² | Пузырьковая сортировка, сравнение всех пар |
| O(2ⁿ) | Экспоненциальная | 1.27×10³⁰ | ∞ | Полный перебор подмножеств |

### Разбор сложности на примерах

```csharp
// O(1) — один проход всегда
int GetFirst(int[] arr) => arr[0];

// O(n) — один проход
int Sum(int[] arr)
{
    int sum = 0;
    for (int i = 0; i < arr.Length; i++) sum += arr[i];
    return sum;
}

// O(n²) — вложенный цикл
bool HasDuplicate(int[] arr)
{
    for (int i = 0; i < arr.Length; i++)
        for (int j = i + 1; j < arr.Length; j++)
            if (arr[i] == arr[j]) return true;
    return false;
}

// O(log n) — бинарный поиск
int BinarySearch(int[] sorted, int target)
{
    int left = 0, right = sorted.Length - 1;
    while (left <= right)
    {
        int mid = left + (right - left) / 2;
        if (sorted[mid] == target) return mid;
        if (sorted[mid] < target) left = mid + 1;
        else right = mid - 1;
    }
    return -1;
}

// O(n log n) — сортировка + линейный проход
bool HasDuplicateOptimized(int[] arr)
{
    Array.Sort(arr);            // O(n log n)
    for (int i = 1; i < arr.Length; i++)  // O(n)
        if (arr[i] == arr[i - 1]) return true;
    return false;
}
// Итого: O(n log n + n) = O(n log n)
```

### Amortized Analysis

```
Amortized O(1) = операция ОБЫЧНО занимает O(1), но ИНОГДА — O(n).
Среднее по всем операциям — O(1).

Пример: List<T>.Add()
  - Обычно: O(1) — просто запись в массив
  - При расширении: O(n) — копирование всего массива
  - Расширение происходит в 2, 4, 8, 16, 32 ... элементах
  - Геометрическая прогрессия: n + n/2 + n/4 + ... = 2n операций на n вставок
  - 2n / n = 2 → O(1) amortized
```

---

## Сортировки — сравнительный анализ

### QuickSort (быстрая сортировка)

```
Алгоритм (Divide & Conquer):
  1. Выбрать pivot (опорный элемент)
  2. Разделить: элементы < pivot слева, > pivot справа
  3. Рекурсивно отсортировать левую и правую части

Сложность:
  - Best/Average: O(n log n)
  - Worst: O(n²) — когда pivot всегда min или max (уже отсортированный массив)
  
  - In-place: да (не требует доп. памяти)
  - Stable: нет
  - Используется: стандартная сортировка в большинстве языков (как часть IntroSort)
```

### MergeSort (сортировка слиянием)

```
Алгоритм (Divide & Conquer):
  1. Разделить массив пополам
  2. Рекурсивно отсортировать каждую половину
  3. Слить два отсортированных подмассива

Сложность:
  - Всегда O(n log n) — гарантированно
  - Доп. память: O(n)
  - Stable: да
  - Используется: когда нужна стабильная сортировка; внешняя сортировка
```

### HeapSort (пирамидальная сортировка)

```
Алгоритм:
  1. Построить max-heap из массива (Heapify) — O(n)
  2. n раз: извлечь максимум (корень), поместить в конец — O(n log n)

Сложность:
  - Всегда O(n log n)
  - In-place: да
  - Stable: нет
  - Используется: как fallback в IntroSort
```

### Сравнительная таблица сортировок

| Алгоритм | Средняя | Худшая | Память | Stable | In-place |
|----------|:-------:|:------:|:------:|:------:|:--------:|
| Bubble Sort | O(n²) | O(n²) | O(1) | Да | Да |
| Insertion Sort | O(n²) | O(n²) | O(1) | Да | Да |
| Selection Sort | O(n²) | O(n²) | O(1) | Нет | Да |
| **QuickSort** | O(n log n) | O(n²) | O(log n) | Нет | Да |
| **MergeSort** | O(n log n) | O(n log n) | O(n) | Да | Нет |
| **HeapSort** | O(n log n) | O(n log n) | O(1) | Нет | Да |
| TimSort | O(n log n) | O(n log n) | O(n) | Да | Нет |
| Counting Sort | O(n + k) | O(n + k) | O(n + k) | Да | Нет |
| Radix Sort | O(d·n) | O(d·n) | O(n + k) | Да | Нет |

```
Q: Какой алгоритм сортировки использует .NET Array.Sort()?
A: IntroSort — гибрид QuickSort + HeapSort + InsertionSort:
   - QuickSort с медианой-из-трёх для выбора pivot
   - HeapSort если глубина рекурсии > 2·log₂ n (защита от O(n²))
   - InsertionSort для подмассивов < 16 элементов

Q: Почему на практике QuickSort быстрее MergeSort?
A: Лучшая cache locality (in-place, последовательный доступ).
   MergeSort требует O(n) дополнительной памяти и прыгает по памяти.
```

---

## Графы — обход и алгоритмы

### Представление графов

```csharp
// 1. Матрица смежности (Adjacency Matrix)
int[,] matrix = new int[n, n];
matrix[0, 1] = 1; // ребро от 0 к 1
// Плюсы: O(1) проверка ребра
// Минусы: O(V²) памяти всегда

// 2. Список смежности (Adjacency List) — предпочтительный
var graph = new List<int>[n];
graph[0] = new List<int> { 1, 2 }; // вершина 0 → 1 и 2
// Плюсы: O(V + E) памяти
// Минусы: O(degree) проверка ребра

// 3. Рёбра с весами
var edges = new List<(int from, int to, int weight)>();
edges.Add((0, 1, 5));
```

### BFS (Breadth-First Search) — поиск в ширину

```
O(V + E), очередь (Queue)

Используется:
  - Кратчайший путь в невзвешенном графе
  - Поиск в ширину (уровень за уровнем)
  - Проверка двудольности
```

```csharp
void BFS(List<int>[] graph, int start)
{
    var visited = new bool[graph.Length];
    var queue = new Queue<int>();
    queue.Enqueue(start);
    visited[start] = true;

    while (queue.Count > 0)
    {
        int v = queue.Dequeue();
        Console.Write(v + " ");

        foreach (int neighbor in graph[v])
        {
            if (!visited[neighbor])
            {
                visited[neighbor] = true;
                queue.Enqueue(neighbor);
            }
        }
    }
}
```

### DFS (Depth-First Search) — поиск в глубину

```
O(V + E), стек (рекурсия)

Используется:
  - Поиск циклов
  - Топологическая сортировка
  - Поиск компонент связности
  - Поиск мостов и точек сочленения
```

```csharp
// DFS — рекурсивный
void DFS(List<int>[] graph, int v, bool[] visited)
{
    visited[v] = true;
    Console.Write(v + " ");

    foreach (int neighbor in graph[v])
    {
        if (!visited[neighbor])
            DFS(graph, neighbor, visited);
    }
}

// Топологическая сортировка (DFS-based)
List<int> TopologicalSort(List<int>[] graph)
{
    var visited = new bool[graph.Length];
    var order = new List<int>();

    for (int i = 0; i < graph.Length; i++)
        if (!visited[i])
            TopoDFS(i);

    order.Reverse(); // Обратный порядок завершения
    return order;

    void TopoDFS(int v)
    {
        visited[v] = true;
        foreach (int u in graph[v])
            if (!visited[u])
                TopoDFS(u);
        order.Add(v); // Добавляем ПОСЛЕ обхода всех потомков
    }
}
```

### Dijkstra — кратчайший путь во взвешенном графе

```
O((V + E) log V) с PriorityQueue

Ограничение: веса рёбер должны быть НЕОТРИЦАТЕЛЬНЫМИ
```

```csharp
int[] Dijkstra(List<(int to, int weight)>[] graph, int start)
{
    int n = graph.Length;
    var dist = new int[n];
    Array.Fill(dist, int.MaxValue);
    dist[start] = 0;

    var pq = new PriorityQueue<int, int>(); // (vertex, distance)
    pq.Enqueue(start, 0);

    while (pq.Count > 0)
    {
        int v = pq.Dequeue();

        foreach (var (to, weight) in graph[v])
        {
            int newDist = dist[v] + weight;
            if (newDist < dist[to])
            {
                dist[to] = newDist;
                pq.Enqueue(to, newDist);
            }
        }
    }
    return dist;
}
```

### Алгоритм A\*

```
A* = Dijkstra + эвристика

f(n) = g(n) + h(n)
  g(n) — известное расстояние от start до n
  h(n) — эвристика (оценка расстояния от n до goal)
  
Если h(n) — admissible (не переоценивает):

  h(n) = 0        → Dijkstra
  h(n) = точное   → прямой путь без лишних узлов
  h(n) — Manhattan distance (для сеток без диагоналей)
  h(n) — Euclidean distance (для сеток с диагоналями)
```

---

## Алгоритмические паттерны для собеседования

### Two Pointers (Два указателя)

```
Задача: найти пару чисел в отсортированном массиве, дающую сумму target.

O(n) вместо O(n²) перебора всех пар.
```

```csharp
(int, int)? TwoSum(int[] sorted, int target)
{
    int left = 0, right = sorted.Length - 1;
    while (left < right)
    {
        int sum = sorted[left] + sorted[right];
        if (sum == target) return (sorted[left], sorted[right]);
        if (sum < target) left++;
        else right--;
    }
    return null;
}
```

### Sliding Window (Скользящее окно)

```
Задача: максимальная сумма подмассива длины k.

O(n) вместо O(n·k).
```

```csharp
int MaxSumSubarray(int[] arr, int k)
{
    int windowSum = 0;
    for (int i = 0; i < k; i++) windowSum += arr[i];
    int maxSum = windowSum;

    for (int i = k; i < arr.Length; i++)
    {
        windowSum += arr[i] - arr[i - k]; // Убрали левый, добавили правый
        maxSum = Math.Max(maxSum, windowSum);
    }
    return maxSum;
}
```

### Fast & Slow Pointers (Быстрый и медленный указатели)

```
Задача: найти цикл в связном списке (Floyd's Cycle Detection).

O(n) времени, O(1) памяти.
```

```csharp
bool HasCycle(ListNode head)
{
    if (head == null) return false;
    ListNode slow = head, fast = head;

    while (fast != null && fast.Next != null)
    {
        slow = slow.Next;
        fast = fast.Next.Next;
        if (slow == fast) return true; // Встретились — есть цикл
    }
    return false;
}

// Найти точку входа в цикл:
ListNode FindCycleStart(ListNode head)
{
    ListNode slow = head, fast = head;

    // Шаг 1: найти встречу
    while (fast != null && fast.Next != null)
    {
        slow = slow.Next;
        fast = fast.Next.Next;
        if (slow == fast) break;
    }
    if (fast == null || fast.Next == null) return null; // нет цикла

    // Шаг 2: от головы и от точки встречи — встретятся на входе в цикл
    slow = head;
    while (slow != fast)
    {
        slow = slow.Next;
        fast = fast.Next;
    }
    return slow;
}
```

---

## Чек-лист: алгоритмы на собеседовании

- [ ] Big O: O(1), O(log n), O(n), O(n log n), O(n²), O(2ⁿ) — примеры каждого
- [ ] Разница между average, worst и amortized complexity
- [ ] QuickSort vs MergeSort vs HeapSort — плюсы/минусы
- [ ] Что такое IntroSort и почему он в .NET?
- [ ] BFS vs DFS: когда что использовать?
- [ ] Dijkstra: как работает, ограничения
- [ ] A\*: чем отличается от Dijkstra?
- [ ] Two Pointers, Sliding Window — распознать задачу и применить
- [ ] Dynamic Programming: top-down (memoization) vs bottom-up (tabulation)
- [ ] Хеш-таблица: как решить задачу за O(n) с Dictionary/HashSet

### Типичные задачи на DP (Dynamic Programming)

```
1. Fibonacci: F(n) = F(n-1) + F(n-2)  → O(n) с memoization
2. Knapsack (рюкзак): максимизировать ценность при ограничении веса  → O(n·W)
3. Longest Common Subsequence (LCS)  → O(n·m)
4. Edit Distance (Levenshtein)  → O(n·m)
5. Coin Change: минимальное число монет для суммы  → O(n·amount)
```

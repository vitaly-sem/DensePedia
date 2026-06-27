# Визуальное дерево (VisualTree) vs Логическое дерево (LogicalTree)

---

## Факты

**LogicalTree:**
- То, что вы пишете в XAML
- Элементы: `Window > Grid > StackPanel > Button`
- Используется для: resource lookup (`FindResource`), binding inheritance, event routing
- Содержит только логические контейнеры (Layout-элементы и Controls)

**VisualTree:**
- То, что реально отрисовывается
- Включает визуальные части контролов: `Button > ButtonChrome > ContentPresenter > TextBlock`
- Используется для: hit testing, rendering, animations, `VisualTreeHelper`
- Производительность: обход VisualTree — дорогой (вызов неуправляемого кода)

**Пример — различие:**

```xml
<!-- LogicalTree -->
<Window>
  <Grid>
    <Button Content="Click" />
  </Grid>
</Window>
```

```
LogicalTree:
  Window
    Grid
      Button

VisualTree (упрощённо):
  Window
    Border (ResizeBorder)
      AdornerDecorator
        ContentPresenter
          Grid
            ButtonChrome
              ContentPresenter
                TextBlock ("Click")
```

**Факт:** VisualTree может быть в **10-100 раз** больше LogicalTree по количеству узлов.

**Факт:** В WPF каждый `UIElement` имеет VisualTree. Это даёт гибкость стилизации ControlTemplate, но увеличивает memory footprint.

---

## Поиск элементов

```csharp
// Поиск вниз по VisualTree (найти TextBlock внутри Button)
public static T? FindVisualChild<T>(DependencyObject parent) where T : DependencyObject
{
    for (int i = 0; i < VisualTreeHelper.GetChildrenCount(parent); i++)
    {
        var child = VisualTreeHelper.GetChild(parent, i);
        if (child is T childTyped)
            return childTyped;
        
        var result = FindVisualChild<T>(child);
        if (result != null)
            return result;
    }
    return null;
}

// Breadth-first — эффективнее (не идёт глубоко без необходимости)
public static T? FindVisualChildBFS<T>(DependencyObject parent) where T : DependencyObject
{
    var queue = new Queue<DependencyObject>();
    queue.Enqueue(parent);
    
    while (queue.Count > 0)
    {
        var current = queue.Dequeue();
        for (int i = 0; i < VisualTreeHelper.GetChildrenCount(current); i++)
        {
            var child = VisualTreeHelper.GetChild(current, i);
            if (child is T childTyped)
                return childTyped;
            queue.Enqueue(child);
        }
    }
    return null;
}

// Поиск вверх по LogicalTree (найти родительский ListBoxItem)
public static T? FindLogicalParent<T>(DependencyObject child) where T : DependencyObject
{
    var parent = LogicalTreeHelper.GetParent(child);
    while (parent != null)
    {
        if (parent is T parentTyped)
            return parentTyped;
        parent = LogicalTreeHelper.GetParent(parent);
    }
    return null;
}
```

**Факт:** `FindName()` ищет только в LogicalTree. Для поиска визуальных частей (из ControlTemplate) используйте `VisualTreeHelper`.

---

## Когда обходить дерево?

| Сценарий | Дерево | Метод |
|---|---|---|
| Получить DataContext дочернего контрола | Logical | `LogicalTreeHelper.GetParent()` |
| Найти элемент в ControlTemplate | Visual | `VisualTreeHelper.GetChild()` |
| Hit-тестирование (что под курсором) | Visual | `VisualTreeHelper.HitTest()` |
| Resource resolution (`{StaticResource}`) | Logical + Visual | `FrameworkElement.FindResource()` |
| Получить все визуальные дочерние элементы | Visual | `VisualTreeHelper.GetChildrenCount()` |

**Hit Testing — продвинуто:**

```csharp
private void OnMouseDown(object sender, MouseButtonEventArgs e)
{
    var hitTestResult = VisualTreeHelper.HitTest(this, e.GetPosition(this));
    
    if (hitTestResult?.VisualHit is TextBlock tb)
        MessageBox.Show($"Clicked on text: {tb.Text}");
    
    // Geometry hit testing
    var geometry = new EllipseGeometry(e.GetPosition(this), 5, 5);
    var results = new List<DependencyObject>();
    
    VisualTreeHelper.HitTest(this, null,
        result =>
        {
            results.Add(result.VisualHit);
            return HitTestResultBehavior.Continue;
        },
        new GeometryHitTestParameters(geometry));
}
```

---

## EffectiveValues — Dependency Property System

При обходе VisualTree/LogicalTree стоит помнить, что каждый `DependencyObject` хранит **EffectiveValues array** — массив всех установленных DP. Это влияет на memory:

```csharp
// Каждый DependencyObject аллоцирует массив слотов под DP
// Если у вас 1000 элементов с кастомными DP → 1000 * (среднее число DP) слотов
```

---

## Чек-лист

- [ ] Чем отличается VisualTree от LogicalTree
- [ ] Как искать элементы в дереве (DFS vs BFS)
- [ ] Когда обход дерева дорогой (ItemCount > 1000 с ControlTemplate)
- [ ] Hit Testing с VisualTreeHelper
- [ ] Resource lookup через LogicalTree

# Производительность WPF — факты

---

## Virtualization

### Факты

**Типы виртуализации:**

| Тип | Что виртуализирует | Container |
|---|---|---|
| **UI Virtualization** | Визуальные элементы (не создаёт невидимые) | VirtualizingStackPanel |
| **Data Virtualization** | Данные (подгружает по мере скролла) | Кастомная реализация |
| **Container Recycling** | Переиспользует контейнеры | `VirtualizationMode="Recycling"` |

**VirtualizationMode:**

| Режим | Описание | Memory | CPU |
|---|---|---|---|
| `Standard` | Создаёт/удаляет контейнеры при скролле | Низкий | Высокий (GC) |
| `Recycling` | Переиспользует контейнеры (меняет DataContext) | Средний | Низкий |

```xml
<!-- Максимальная производительность для больших списков -->
<ListBox ItemsSource="{Binding LargeCollection}"
         VirtualizingPanel.IsVirtualizing="True"
         VirtualizingPanel.VirtualizationMode="Recycling"
         VirtualizingPanel.ScrollUnit="Item" />
```

**Когда виртуализация НЕ работает:**

| Ситуация | Причина | Решение |
|---|---|---|
| Внутри `ScrollViewer` | ScrollViewer даёт бесконечное пространство | Вынести ScrollViewer |
| `WrapPanel` / `StackPanel` | Эти панели не виртуализируют | Использовать VirtualizingWrapPanel (NuGet) |
| `ItemsControl` без VirtualizingStackPanel | По умолчанию StackPanel | Явно указать VirtualizingStackPanel |
| Grouping включён | GroupItem создаётся для каждой группы | Ограничить число групп |
| Variable-height элементы | VirtualizingStackPanel требует фиксированной высоты для корректного scrolling | `ScrollUnit="Pixel"`, задать высоту |

**Deferred Loading / Async Binding:**

```csharp
// Lazy loading данных по мере скролла
public class VirtualizedCollection<T> : ISupportIncrementalLoading, INotifyCollectionChanged
{
    private readonly List<T> _items = new();
    private bool _hasMoreItems = true;
    
    public bool HasMoreItems => _hasMoreItems;
    public IAsyncOperation<LoadMoreItemsResult> LoadMoreItemsAsync(uint count)
    {
        return Task.Run(async () =>
        {
            var newItems = await _service.FetchBatchAsync(_items.Count, (int)count);
            foreach (var item in newItems)
                _items.Add(item);
            _hasMoreItems = newItems.Count == count;
            CollectionChanged?.Invoke(this, new NotifyCollectionChangedEventArgs(
                NotifyCollectionChangedAction.Add, newItems));
            return new LoadMoreItemsResult { Count = (uint)newItems.Count };
        }).AsAsyncOperation();
    }
}
```

**Факт:** `DeferrableContent` (атрибут XAML `x:DeferLoadStrategy`) в .NET 5+ WPF позволяет отложить создание элементов UI до активации: `<StackPanel x:DeferLoadStrategy="Lazy">` — memory saving для редко используемых tab-панелей.

**Факт:** Container Recycling может вызвать проблемы с сохранением состояния (CheckBox selection). Используйте `IsSelected` binding вместо local state.

---

## Freezable Objects

**Факты:**
- После `Freeze()` объект становится **immutable** и **thread-safe**
- **Не генерирует** change notifications (экономия CPU/memory)
- Объекты WPF (Brushes, Pens, Transform) — **Freezable**

```csharp
var brush = new SolidColorBrush(Colors.Red);

if (brush.CanFreeze)
{
    brush.Freeze();  // Нельзя больше менять Color
}

// Можно использовать из любого потока
// Не будет вызывать InvalidateVisual при изменениях

// ❌ Ошибка после Freeze
brush.Color = Colors.Blue;  // InvalidOperationException
```

**Факт:** Встроенные WPF-ресурсы (SystemColors, SystemBrushes) уже frozen. Кастомные ресурсы `SolidColorBrush`, `LinearGradientBrush` — нет.

**Факт:** Frozen-объекты можно **кэшировать**. Один frozen brush, используемый 1000 элементов — один объект в памяти. Non-frozen — 1000 копий.

---

## Bitmap Effects vs ShaderEffects

**Факты:**

```csharp
// ❌ Устарело — rendering на CPU, slow
var blur = new BlurBitmapEffect { Radius = 5 };
button.BitmapEffect = blur;  // Deprecated в .NET 4.0

// ✅ GPU-accelerated — ShaderEffect (Pixel Shader 2.0+)
public class CustomBlurEffect : ShaderEffect
{
    private static readonly PixelShader _shader = new()
    {
        UriSource = new Uri("pack://application:,,,/Shaders/Blur.ps")
    };
    
    public static readonly DependencyProperty InputProperty =
        RegisterPixelShaderSamplerProperty("Input", typeof(CustomBlurEffect), 0);
    
    public CustomBlurEffect()
    {
        PixelShader = _shader;
    }
}
```

**Факт:** WPF 4+ использует **GPU acceleration** (Direct3D) для rendering. CPU rendering (software fallback) — только если GPU недоступен (RDP, VM без GPU).

**Факт:** `Effect` (DropShadowEffect, BlurEffect) — это GPU ShaderEffect. Используйте их вместо BitmapEffects:

```xml
<!-- ✅ DropShadowEffect — GPU, recommended -->
<Button Content="Shadow">
  <Button.Effect>
    <DropShadowEffect Color="Gray" BlurRadius="5" Opacity="0.5" />
  </Button.Effect>
</Button>
```

---

## UI Thread и Freeze

**Факты:**
- WPF имеет **два потока** для rendering: UI thread + Render thread (невидимый)
- Render thread занимается композицией (Direct3D)
- Если UI thread заблокирован — приложение «зависает», но render thread продолжает анимации

**WPF Rendering Pipeline:**

```
UI Thread                  Render Thread
   |                           |
   Measure/Arrange             |
   InvalidateVisual ───────► Composition
   InvalidateArrange           Materialize
                               Render (Direct3D)
```

**Факт:** Слишком частые `InvalidateVisual()` вызывают перерисовку на каждый вызов. Используйте `InvalidateArrange()` / `InvalidateMeasure()` если меняется только layout.

---

## Оптимизация Binding

```csharp
// 1. {Binding} без оптимизации — Reflection каждый раз
// 2. Compiled Bindings (x:DataType) — .NET 5 WPF
//    компилирует binding во время сборки

<ItemsControl ItemsSource="{x:Bind ViewModel.Items}"
              xmlns:vm="using:MyApp.ViewModels"
              x:DataType="vm:MainViewModel">
  <ItemsControl.ItemTemplate>
    <DataTemplate x:DataType="vm:ItemViewModel">
      <TextBlock Text="{x:Bind Name}" />
    </DataTemplate>
  </ItemsControl.ItemTemplate>
</ItemsControl>
```

**Факт:** Compiled bindings (`x:Bind`) в 10-50x быстрее обычных `{Binding}`. Доступны в .NET 5+ WPF/.NET Core WPF.

**Факт:** Самый быстрый binding — вообще без binding: прямой code-behind. Но это нарушает MVVM.

---

## Memory Leaks — частые причины

| Причина | Пример | Решение |
|---|---|---|
| Event subscription | `source.Event += handler` (strong ref) | WeakEventManager |
| Binding на статический ресурс | `{StaticResource}` к большому объекту | DynamicResource или очистка |
| DataContext не очищен | TabControl сохраняет VM в памяти | VirtualizingTabControl |
| BitmapImage не очищен | Картинки без `CacheOption` | `BitmapCacheOption.OnLoad` |
| CollectionView не освобождён | `CollectionViewSource.GetDefaultView` | Очистка при закрытии |

```csharp
// BitmapCacheOption — предотвратить утечку дескрипторов
var image = new BitmapImage();
image.BeginInit();
image.UriSource = new Uri(path);
image.CacheOption = BitmapCacheOption.OnLoad;  // Загрузить сразу, не держать файл
image.EndInit();
```

---

## Чек-лист

- [ ] VirtualizationMode.Recycling для списков > 100 элементов
- [ ] Freezable: Freeze() для статических ресурсов (Brushes, Transforms)
- [ ] ShaderEffects (DropShadow, Blur) вместо BitmapEffects
- [ ] Compiled bindings (x:Bind) — .NET 5+ WPF
- [ ] InvalidateVisual vs InvalidateArrange vs InvalidateMeasure
- [ ] Memory leaks: события, DataContext, BitmapImage
- [ ] UI thread не блокировать (Task + Dispatcher)

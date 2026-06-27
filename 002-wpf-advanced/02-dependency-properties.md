# Dependency Properties (DP) — углублённо

---

## Факты

**Что делает DP особенной:**
1. **Value resolution** — приоритет из 10 уровней (объяснено ниже)
2. **Change notification** — PropertyChanged callback для Binding
3. **Value inheritance** — значение из родителя (например, `FontSize`, `DataContext`)
4. **Animation support** — анимация имеет приоритет выше local value
5. **Styling** — Style setters, triggers, templates
6. **Memory** — одно static поле для свойства, а не instance поле для каждого объекта
7. **Validation & Coercion** — встроенные механизмы

---

## Приоритет значений (Dependency Property Value Precedence)

```
 1 (highest)  Property system coercion (CoerceValueCallback)
 2            Animation (ключевые кадры / From/To/By)
 3            Local value (SetValue, XAML attribute)
 4            TemplatedParent template properties
 5            Implicit style triggers
 6            Template triggers
 7            Style triggers
 8            Template setters
 9            Default style (theme style)
10  (lowest)  Default value (FrameworkPropertyMetadata.DefaultValue)
```

**Практическое следствие:**

```csharp
// Если вы установили локальное значение (приоритет 3)
button.Width = 100;

// Анимация (приоритет 2) переопределит его
var animation = new DoubleAnimation(200, TimeSpan.FromSeconds(1));
button.BeginAnimation(Button.WidthProperty, animation);

// Даже после этого:
button.Width = 150;  // Не сработает, пока анимация не остановлена!

// Сброс анимации — вернёт локальное значение
button.BeginAnimation(Button.WidthProperty, null);
// Теперь button.Width = 100 (локальное)
```

**Факт:** `SetCurrentValue` устанавливает значение на приоритете 3 (local), но **не перебивает анимацию**. Используйте для default values из VM.

```csharp
// SetCurrentValue — "подсказка" для property system
// Не переопределяет анимацию, triggers, стили
button.SetCurrentValue(Button.WidthProperty, 150.0);
```

---

## Создание кастомной DependencyProperty

```csharp
public class GaugeControl : FrameworkElement
{
    // 1. Регистрация DP (static field)
    public static readonly DependencyProperty ValueProperty =
        DependencyProperty.Register(
            name: nameof(Value),
            propertyType: typeof(double),
            ownerType: typeof(GaugeControl),
            typeMetadata: new FrameworkPropertyMetadata(
                defaultValue: 0.0,
                flags: FrameworkPropertyMetadataOptions.AffectsRender  // invalidate Arrange
                      | FrameworkPropertyMetadataOptions.AffectsMeasure // invalidate Measure
                      | FrameworkPropertyMetadataOptions.BindsTwoWayByDefault, // TwoWay binding
                propertyChangedCallback: OnValueChanged,
                coerceValueCallback: CoerceValue,
                isAnimationProhibited: false),
            validateValueCallback: ValidateValue);
    
    // 2. CLR wrapper (для C# доступа)
    public double Value
    {
        get => (double)GetValue(ValueProperty);
        set => SetValue(ValueProperty, value);
    }
    
    // 3. Property Changed — реакция на изменение
    private static void OnValueChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
    {
        var control = (GaugeControl)d;
        var oldValue = (double)e.OldValue;
        var newValue = (double)e.NewValue;
        
        control.InvalidateVisual();  // Принудительный перерендер
    }
    
    // 4. Coerce — принудительное ограничение (до validation)
    private static object CoerceValue(DependencyObject d, object baseValue)
    {
        var value = (double)baseValue;
        return Math.Clamp(value, 0.0, 100.0);  // всегда 0-100
    }
    
    // 5. Validate — проверка до установки
    private static bool ValidateValue(object value)
    {
        if (value is double d)
            return !double.IsNaN(d) && !double.IsInfinity(d);
        return false;
    }
}
```

---

## Read-Only DependencyProperty

```csharp
// Для свойств, которые устанавливаются только системой (IsMouseOver, IsSelected)
public class CustomControl : Control
{
    private static readonly DependencyPropertyKey IsCustomPropertyKey =
        DependencyProperty.RegisterReadOnly(
            name: "IsCustom",
            propertyType: typeof(bool),
            ownerType: typeof(CustomControl),
            typeMetadata: new FrameworkPropertyMetadata(false));
    
    public static readonly DependencyProperty IsCustomProperty =
        IsCustomPropertyKey.DependencyProperty;
    
    public bool IsCustom
    {
        get => (bool)GetValue(IsCustomProperty);
        private set => SetValue(IsCustomPropertyKey, value);  // Только внутри класса
    }
}
```

---

## Attached Properties

**Факты:**
- Позволяют "прикрепить" свойство к любому `DependencyObject`
- Хранятся в dictionary самого объекта (не путать с Extension methods)
- Используются для: `Grid.Row`, `Canvas.Left`, `DockPanel.Dock`

```csharp
// Создание Attached Property
public static class PanelExtensions
{
    // Регистрация как Attached
    public static readonly DependencyProperty CornerRadiusProperty =
        DependencyProperty.RegisterAttached(
            name: "CornerRadius",
            propertyType: typeof(CornerRadius),
            ownerType: typeof(PanelExtensions),
            typeMetadata: new FrameworkPropertyMetadata(
                new CornerRadius(0),
                FrameworkPropertyMetadataOptions.AffectsRender,
                OnCornerRadiusChanged));
    
    // Setters/Getters (обязательны для XAML)
    public static void SetCornerRadius(DependencyObject element, CornerRadius value)
        => element.SetValue(CornerRadiusProperty, value);
    
    public static CornerRadius GetCornerRadius(DependencyObject element)
        => (CornerRadius)element.GetValue(CornerRadiusProperty);
    
    // Обработчик — применяем к Border элемента
    private static void OnCornerRadiusChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
    {
        if (d is FrameworkElement element && element.Template != null)
        {
            // Ждём загрузки Template
            element.ApplyTemplate();
            var border = (Border)element.Template.FindName("PART_Border", element);
            if (border != null)
                border.CornerRadius = (CornerRadius)e.NewValue;
        }
    }
}
```

```xml
<!-- XAML использование -->
<Button local:PanelExtensions.CornerRadius="8,8,0,0" Content="Rounded Top" />
```

**Факт:** Attached Properties **не static extension methods** — они хранятся в EffectiveValues массиве `DependencyObject`, к которому прикреплены.

---

## Attached Property как Behaviour (эмуляция Behaviours)

```csharp
public static class DragBehavior
{
    public static readonly DependencyProperty IsDraggableProperty =
        DependencyProperty.RegisterAttached("IsDraggable", typeof(bool), typeof(DragBehavior),
            new PropertyMetadata(false, OnIsDraggableChanged));
    
    public static void SetIsDraggable(UIElement element, bool value) => element.SetValue(IsDraggableProperty, value);
    public static bool GetIsDraggable(UIElement element) => (bool)element.GetValue(IsDraggableProperty);
    
    private static void OnIsDraggableChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
    {
        if (d is UIElement element)
        {
            if ((bool)e.NewValue)
            {
                element.MouseDown += OnMouseDown;
                element.MouseMove += OnMouseMove;
                element.MouseUp += OnMouseUp;
            }
            else
            {
                element.MouseDown -= OnMouseDown;
                element.MouseMove -= OnMouseMove;
                element.MouseUp -= OnMouseUp;
            }
        }
    }
}
```

---

## Property Engine Internals

**Факт:** Каждый `DependencyProperty` имеет **static индекс** (глобально уникальный integer). EffectiveValues массив внутри `DependencyObject` хранит пары (index, value) по этому индексу.

**Факт:** Если у `DependencyObject` зарегистрировано много DP, EffectiveValues массив расширяется — это одна из причин **memory fragmentation** в WPF.

**Факт:** `ClearValue(DependencyProperty)` сбрасывает значение на **DefaultValue** (самый низкий приоритет), а не на null.

**Факт:** `DependencyProperty.Register` создаёт **один static объект** на всё AppDomain. `new PropertyMetadata(...)` создаётся один раз при первом использовании.

---

## Чек-лист

- [ ] 10 уровней приоритета DP (от coercion до default)
- [ ] `SetCurrentValue` vs `SetValue` — различие
- [ ] Read-only DP (`DependencyPropertyKey`)
- [ ] Attached Properties — регистрация, setter/getter
- [ ] Callbacks: PropertyChanged → CoerceValue → ValidateValue (порядок вызова)
- [ ] ClearValue возвращает к Default, не к null
- [ ] Memory: static DP field, instance EffectiveValues array

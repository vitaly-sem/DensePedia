# Attached Behaviours в WPF

---

## Что такое Attached Behaviour

**Факт:** Attached Behaviour — это Attached Property + обработчики событий. Позволяет добавлять поведение любому `DependencyObject` без наследования.

**Зачем:**
- **DRY** — не писать один и тот же код в каждом View
- **Отделение логики** от View (MVVM-friendly)
- **Переиспользование** поведения между разными контролами

---

## Пример: AutoFocus Behaviour

```csharp
public static class AutoFocusBehavior
{
    public static readonly DependencyProperty IsAutoFocusedProperty =
        DependencyProperty.RegisterAttached(
            "IsAutoFocused", typeof(bool), typeof(AutoFocusBehavior),
            new FrameworkPropertyMetadata(false, OnIsAutoFocusedChanged));
    
    public static void SetIsAutoFocused(DependencyObject element, bool value)
        => element.SetValue(IsAutoFocusedProperty, value);
    
    public static bool GetIsAutoFocused(DependencyObject element)
        => (bool)element.GetValue(IsAutoFocusedProperty);
    
    private static void OnIsAutoFocusedChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
    {
        if (d is UIElement element)
        {
            if ((bool)e.NewValue)
            {
                // Фокус после загрузки (View уже отрисован)
                element.Loaded += (s, _) => ((UIElement)s).Focus();
            }
        }
    }
}
```

```xml
<TextBox local:AutoFocusBehavior.IsAutoFocused="True" />
```

---

## Пример: NumericInput Behaviour

```csharp
public static class NumericInputBehavior
{
    public static readonly DependencyProperty IsNumericProperty =
        DependencyProperty.RegisterAttached("IsNumeric", typeof(bool), 
            typeof(NumericInputBehavior),
            new PropertyMetadata(false, OnIsNumericChanged));
    
    public static void SetIsNumeric(DependencyObject element, bool value)
        => element.SetValue(IsNumericProperty, value);
    
    public static bool GetIsNumeric(DependencyObject element)
        => (bool)element.GetValue(IsNumericProperty);
    
    private static void OnIsNumericChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
    {
        if (d is TextBox textBox)
        {
            textBox.PreviewTextInput += OnPreviewTextInput;
            DataObject.AddPastingHandler(textBox, OnPaste);
        }
    }
    
    private static void OnPreviewTextInput(object sender, TextCompositionEventArgs e)
    {
        var textBox = (TextBox)sender;
        var newText = textBox.Text.Remove(textBox.SelectionStart, textBox.SelectionLength)
                      + e.Text;
        
        e.Handled = !IsNumeric(newText);
    }
    
    private static void OnPaste(object sender, DataObjectPastingEventArgs e)
    {
        if (e.DataObject.GetDataPresent(DataFormats.Text))
        {
            var text = e.DataObject.GetData(DataFormats.Text)?.ToString();
            if (text != null && !IsNumeric(text))
                e.CancelCommand();
        }
    }
    
    private static bool IsNumeric(string text)
        => double.TryParse(text, NumberStyles.Any, CultureInfo.InvariantCulture, out _);
}
```

---

## Пример: Drag & Drop Behaviour

```csharp
public static class DragDropBehavior
{
    public static readonly DependencyProperty IsDropTargetProperty =
        DependencyProperty.RegisterAttached("IsDropTarget", typeof(bool),
            typeof(DragDropBehavior),
            new PropertyMetadata(false, OnIsDropTargetChanged));
    
    public static readonly DependencyProperty DropCommandProperty =
        DependencyProperty.RegisterAttached("DropCommand", typeof(ICommand),
            typeof(DragDropBehavior),
            new PropertyMetadata(null));
    
    public static void SetIsDropTarget(DependencyObject element, bool value)
        => element.SetValue(IsDropTargetProperty, value);
    
    public static bool GetIsDropTarget(DependencyObject element)
        => (bool)element.GetValue(IsDropTargetProperty);
    
    public static void SetDropCommand(DependencyObject element, ICommand value)
        => element.SetValue(DropCommandProperty, value);
    
    public static ICommand GetDropCommand(DependencyObject element)
        => (ICommand)element.GetValue(DropCommandProperty);
    
    private static void OnIsDropTargetChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
    {
        if (d is UIElement element)
        {
            element.AllowDrop = true;
            element.DragOver += OnDragOver;
            element.Drop += OnDrop;
        }
    }
    
    private static void OnDragOver(object sender, DragEventArgs e)
    {
        e.Effects = e.Data.GetDataPresent(DataFormats.FileDrop)
            ? DragDropEffects.Copy
            : DragDropEffects.None;
        e.Handled = true;
    }
    
    private static void OnDrop(object sender, DragEventArgs e)
    {
        if (sender is DependencyObject element &&
            e.Data.GetDataPresent(DataFormats.FileDrop))
        {
            var files = (string[])e.Data.GetData(DataFormats.FileDrop);
            var command = GetDropCommand(element);
            
            if (command?.CanExecute(files) == true)
                command.Execute(files);
        }
    }
}
```

```xml
<Border local:DragDropBehavior.IsDropTarget="True"
        local:DragDropBehavior.DropCommand="{Binding FileDropCommand}"
        Background="LightGray">
  <TextBlock Text="Drop files here" />
</Border>
```

---

## Interaction.Triggers (Blend SDK)

**Факт:** Вместо кастомных Attached Behaviours можно использовать `System.Windows.Interactivity` (Blend SDK):

```xml
xmlns:i="http://schemas.microsoft.com/expression/2010/interactivity"

<TextBox>
  <i:Interaction.Behaviors>
    <local:NumericInputBehavior />
  </i:Interaction.Behaviors>
</TextBox>
```

```csharp
public class NumericInputBehavior : Behavior<TextBox>
{
    protected override void OnAttached()
    {
        AssociatedObject.PreviewTextInput += OnPreviewTextInput;
    }
    
    protected override void OnDetaching()
    {
        AssociatedObject.PreviewTextInput -= OnPreviewTextInput;
    }
}
```

**Факт:** `Behavior<T>` наследуется от `System.Windows.Interactivity.Behavior` — предоставляет `AssociatedObject` (типизированный).

---

## Command Behaviour — когда нужно вызывать ICommand из события

```csharp
public static class EventCommandBehavior
{
    public static readonly DependencyProperty EventCommandProperty =
        DependencyProperty.RegisterAttached("EventCommand", typeof(ICommand),
            typeof(EventCommandBehavior),
            new PropertyMetadata(null, OnEventCommandChanged));
    
    public static readonly DependencyProperty EventCommandParameterProperty =
        DependencyProperty.RegisterAttached("EventCommandParameter", typeof(object),
            typeof(EventCommandBehavior),
            new PropertyMetadata(null));
    
    public static void SetEventCommand(DependencyObject element, ICommand value)
        => element.SetValue(EventCommandProperty, value);
    
    public static ICommand GetEventCommand(DependencyObject element)
        => (ICommand)element.GetValue(EventCommandProperty);
    
    public static void SetEventCommandParameter(DependencyObject element, object value)
        => element.SetValue(EventCommandParameterProperty, value);
    
    private static void OnEventCommandChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
    {
        if (d is Button button)
        {
            button.Click -= OnButtonClick;
            if (e.NewValue is ICommand)
                button.Click += OnButtonClick;
        }
    }
    
    private static void OnButtonClick(object sender, RoutedEventArgs e)
    {
        if (sender is DependencyObject element)
        {
            var command = GetEventCommand(element);
            var parameter = GetEventCommandParameter(element);
            
            if (command?.CanExecute(parameter) == true)
                command.Execute(parameter);
        }
    }
}
```

---

## Чек-лист

- [ ] Attached Behaviour = Attached Property + event handlers
- [ ] RegisterAttached (static setters/getters)
- [ ] Поведения: AutoFocus, NumericInput, DragDrop, Event-to-Command
- [ ] Interaction.Behaviors (Blend SDK) — Behavior<T>
- [ ] Отписка от событий в OnDetaching (предотвратить утечки)
- [ ] CommandParameter для передачи данных в команду

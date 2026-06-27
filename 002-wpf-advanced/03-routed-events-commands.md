# Routed Events и Commands

---

## Routed Events

### Три стратегии маршрутизации

| Стратегия | Описание | Примеры |
|---|---|---|
| **Bubbling** | От источника события → вверх по VisualTree | `Click`, `MouseDown`, `KeyUp` |
| **Tunneling** | От корня VisualTree → вниз к источнику | `PreviewKeyDown`, `PreviewMouseDown` |
| **Direct** | Только на источнике | `MouseEnter`, `MouseLeave`, `GotFocus` |

**Факт:** Tunneling события имеют префикс `Preview*` и происходят **до** Bubbling. Пара: `PreviewMouseDown` (tunnel) → `MouseDown` (bubble).

### Порядок обработки:

```
Root (Window)
  ├── PreviewKeyDown (Tunneling) ← перехват до дочернего
  ├── Grid
  │   ├── PreviewKeyDown (Tunneling)
  │   ├── TextBox (источник)
  │   │   └── KeyDown (Bubbling) ← начало всплытия
  │   └── KeyDown (Bubbling)
  └── KeyDown (Bubbling)
```

### Handled — остановка маршрутизации

```csharp
private void OnPreviewKeyDown(object sender, KeyEventArgs e)
{
    // Tunneling — можно перехватить до достижения целевого элемента
    if (e.Key == Key.Enter)
    {
        e.Handled = true;  // Событие НЕ дойдёт до TextBox
        // Но другие обработчики PreviewKeyDown на этом же элементе всё равно сработают
    }
}
```

**Факт:** `e.Handled = true` **не останавливает** другие обработчики на том же элементе — они все получат событие. Маршрутизация остановится для следующих элементов в цепочке.

**Факт:** Можно подписаться на handled события через `AddHandler(RoutedEvent, handler, handledEventsToo: true)`.

### Создание кастомного RoutedEvent

```csharp
public class CustomSlider : Control
{
    // Регистрация
    public static readonly RoutedEvent ValueChangedEvent =
        EventManager.RegisterRoutedEvent(
            name: "ValueChanged",
            routingStrategy: RoutingStrategy.Bubble,
            handlerType: typeof(RoutedPropertyChangedEventHandler<double>),
            ownerType: typeof(CustomSlider));
    
    // CLR event wrapper
    public event RoutedPropertyChangedEventHandler<double> ValueChanged
    {
        add => AddHandler(ValueChangedEvent, value);
        remove => RemoveHandler(ValueChangedEvent, value);
    }
    
    // Protected virtual — для override в наследниках
    protected virtual void OnValueChanged(double oldValue, double newValue)
    {
        RaiseEvent(new RoutedPropertyChangedEventArgs<double>(
            oldValue, newValue, ValueChangedEvent));
    }
    
    // Использование
    private void UpdateValue(double newValue)
    {
        var old = _currentValue;
        _currentValue = newValue;
        OnValueChanged(old, newValue);
    }
}
```

### Attached Event (редко)

```csharp
public static class DragDropExtensions
{
    public static readonly RoutedEvent DragStartedEvent =
        EventManager.RegisterRoutedEvent("DragStarted", RoutingStrategy.Bubble,
            typeof(DragStartedEventHandler), typeof(DragDropExtensions));
    
    public static void AddDragStartedHandler(DependencyObject element, DragStartedEventHandler handler)
        => ((UIElement)element).AddHandler(DragStartedEvent, handler);
    
    public static void RemoveDragStartedHandler(DependencyObject element, DragStartedEventHandler handler)
        => ((UIElement)element).RemoveHandler(DragStartedEvent, handler);
}
```

---

## Commands

### ICommand — три реализации

```csharp
// 1. RelayCommand (самый распространённый)
public class RelayCommand : ICommand
{
    private readonly Action<object?> _execute;
    private readonly Func<object?, bool>? _canExecute;
    
    public RelayCommand(Action<object?> execute, Func<object?, bool>? canExecute = null)
    {
        _execute = execute;
        _canExecute = canExecute;
    }
    
    public event EventHandler? CanExecuteChanged
    {
        add => CommandManager.RequerySuggested += value;
        remove => CommandManager.RequerySuggested -= value;
    }
    
    public bool CanExecute(object? parameter) => _canExecute?.Invoke(parameter) ?? true;
    public void Execute(object? parameter) => _execute(parameter);
}

// 2. RelayCommand<T> (generic — для typed parameter)
public class RelayCommand<T> : ICommand
{
    private readonly Action<T?> _execute;
    private readonly Func<T?, bool>? _canExecute;
    // ... аналогично, но с типизированным параметром
}

// 3. RoutedCommand (WPF built-in)
// Позволяет маршрутизировать команду через VisualTree
// Используется с CommandBinding
```

### CommandManager — как работает Requery

```csharp
// CommandManager.RequerySuggested генерируется при изменении:
// - Keyboard focus
// - Selection changes
// - Any UI interaction
// 
// НО: не генерируется при изменении свойств ViewModel вручную!
// Поэтому — вызывайте CommandManager.InvalidateRequerySuggested() явно
// Или используйте RelayCommand с событием от VM

// Лучшее решение — не полагаться на RequerySuggested
public class BetterRelayCommand : ICommand
{
    public event EventHandler? CanExecuteChanged;
    
    public void RaiseCanExecuteChanged()
        => CanExecuteChanged?.Invoke(this, EventArgs.Empty);
}
```

### CommandBinding + RoutedCommand

```xml
<Window.CommandBindings>
  <CommandBinding Command="ApplicationCommands.Save"
                  Executed="OnSave"
                  CanExecute="OnCanSave" />
</Window.CommandBindings>

<Button Command="ApplicationCommands.Save" Content="Save" />
<MenuItem Command="ApplicationCommands.Save" />
<!-- Одна команда — множественные триггеры -->
```

```csharp
private void OnCanSave(object sender, CanExecuteRoutedEventArgs e)
{
    e.CanExecute = !string.IsNullOrEmpty(_currentDocument?.Title);
}

private void OnSave(object sender, ExecutedRoutedEventArgs e)
{
    SaveDocument(_currentDocument);
}
```

### InputGestures (горячие клавиши)

```xml
<Button Command="ApplicationCommands.Save" Content="_Save">
  <Button.InputBindings>
    <KeyBinding Key="S" Modifiers="Control" Command="ApplicationCommands.Save" />
  </Button.InputBindings>
</Button>
```

**Факт:** Если CommandBinding не найден на пути VisualTree — `CommandTarget` не получит вызов. RoutedCommand идёт от фокуса вверх.

---

## Команды с async

```csharp
public class AsyncRelayCommand : ICommand
{
    private readonly Func<Task> _execute;
    private readonly Func<bool>? _canExecute;
    private bool _isExecuting;
    
    public AsyncRelayCommand(Func<Task> execute, Func<bool>? canExecute = null)
    {
        _execute = execute;
        _canExecute = canExecute;
    }
    
    public event EventHandler? CanExecuteChanged;
    
    public bool CanExecute(object? parameter) => !_isExecuting && (_canExecute?.Invoke() ?? true);
    
    public async void Execute(object? parameter)
    {
        if (_isExecuting) return;
        _isExecuting = true;
        RaiseCanExecuteChanged();
        
        try
        {
            await _execute();
        }
        finally
        {
            _isExecuting = false;
            RaiseCanExecuteChanged();
        }
    }
    
    public void RaiseCanExecuteChanged()
        => CanExecuteChanged?.Invoke(this, EventArgs.Empty);
}
```

**Факт:** `async void` в `ICommand.Execute` — это **fire-and-forget**. Исключения упадут в `SynchronizationContext`. Всегда оборачивайте в try-catch.

---

## Чек-лист

- [ ] Bubbling vs Tunneling vs Direct — порядок, `Preview*` префикс
- [ ] `e.Handled = true` — останавливает маршрутизацию, но не все обработчики на том же элементе
- [ ] `AddHandler(..., handledEventsToo: true)` — перехват уже обработанных событий
- [ ] ICommand: RelayCommand, кастомный async command
- [ ] RoutedCommand + CommandBinding — маршрутизация через VisualTree
- [ ] CommandManager.RequerySuggested — ограничения
- [ ] InputGestures: `KeyBinding`, `MouseBinding`

# WeakEvent Pattern и MVVM

---

## WeakEvent Pattern

### Проблема strong event subscription

```csharp
// Источник (ViewModel) держит strong reference на подписчика (View)
public class ViewModel : INotifyPropertyChanged
{
    public event PropertyChangedEventHandler? PropertyChanged;
    // ↑ Хранит список делегатов → strong references на подписчиков
}

// View подписывается
public class MyView
{
    public MyView(ViewModel vm)
    {
        vm.PropertyChanged += OnViewModelChanged;
        // ↑ GC не соберёт MyView, пока жив ViewModel
        // → Memory leak если View закрыта, а ViewModel жива
    }
}
```

### Решение — WeakEventManager

```csharp
// .NET 4.5+ — встроенный WeakEventManager
public class MyView
{
    public MyView(ViewModel vm)
    {
        WeakEventManager<ViewModel, PropertyChangedEventArgs>
            .AddHandler(vm, nameof(vm.PropertyChanged), OnViewModelChanged);
        // ↑ Использует WeakReference — GC может собрать View
    }
    
    private void OnViewModelChanged(object? sender, PropertyChangedEventArgs e)
    {
        // Обработчик
    }
}
```

### Кастомный WeakEventManager

```csharp
public class PropertyChangedWeakEventManager : WeakEventManager
{
    private PropertyChangedWeakEventManager() { }
    
    public static void AddListener(INotifyPropertyChanged source, IWeakEventListener listener)
    {
        CurrentManager.ProtectedAddListener(source, listener);
    }
    
    public static void RemoveListener(INotifyPropertyChanged source, IWeakEventListener listener)
    {
        CurrentManager.ProtectedRemoveListener(source, listener);
    }
    
    private static PropertyChangedWeakEventManager CurrentManager
    {
        get
        {
            var manager = GetCurrentManager(typeof(PropertyChangedWeakEventManager))
                as PropertyChangedWeakEventManager;
            
            if (manager == null)
            {
                manager = new PropertyChangedWeakEventManager();
                SetCurrentManager(typeof(PropertyChangedWeakEventManager), manager);
            }
            return manager;
        }
    }
    
    // Подписка на событие источника
    protected override ListenerList StartListening(object source)
    {
        ((INotifyPropertyChanged)source).PropertyChanged += OnPropertyChanged;
        return null!;
    }
    
    protected override void StopListening(object source)
    {
        ((INotifyPropertyChanged)source).PropertyChanged -= OnPropertyChanged;
    }
    
    private void OnPropertyChanged(object? sender, PropertyChangedEventArgs e)
    {
        DeliverEvent(sender, e);
    }
}
```

**Факт:** WPF сама использует WeakEventManager для Binding — при закрытии View, Binding не держит сборщик мусора.

**Факт:** WeakEventManager есть не для всех событий. Для событий без встроенного менеджера — используйте `WeakEventManager<TEventSource, TEventArgs>` (generic, .NET 4.5+).

---

## INotifyPropertyChanged

### Базовая реализация

```csharp
public abstract class ViewModelBase : INotifyPropertyChanged
{
    public event PropertyChangedEventHandler? PropertyChanged;
    
    protected bool SetProperty<T>(ref T field, T value,
        [CallerMemberName] string? propertyName = null)
    {
        if (EqualityComparer<T>.Default.Equals(field, value))
            return false;  // Без изменений — не вызываем событие
        
        field = value;
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
        return true;
    }
    
    protected void OnPropertyChanged([CallerMemberName] string? propertyName = null)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    }
}
```

### CommunityToolkit.Mvvm (генерация кода)

```csharp
// NuGet: CommunityToolkit.Mvvm

// [ObservableProperty] — генерирует property + INotifyPropertyChanged
public partial class MainViewModel : ObservableObject
{
    [ObservableProperty]
    private string _name = string.Empty;  // → public string Name { get; set; }
    
    [ObservableProperty]
    [NotifyPropertyChangedFor(nameof(FullName))]  // NameChanged → FullName тоже обновится
    private string _firstName = string.Empty;
    
    [ObservableProperty]
    [NotifyCanExecuteChangedFor(nameof(SaveCommand))]  // Изменение → перепроверка команды
    private bool _isDirty;
    
    public string FullName => $"{FirstName} {LastName}";
    
    // RelayCommand — атрибут (CommunityToolkit 8.1+)
    [RelayCommand]
    private async Task SaveAsync()  // → SaveCommand (IAsyncRelayCommand)
    {
        await _service.SaveAsync();
    }
    
    [RelayCommand(CanExecute = nameof(CanSave))]
    private void Save() { }
    private bool CanSave() => IsDirty;  // Автоматически перепроверяется
}
```

**Факт:** CommunityToolkit.Mvvm использует Roslyn Source Generators — **нет Reflection**, всё на этапе компиляции.

**Факт:** `CallerMemberName` — zero-cost при компиляции. Не используйте magic strings (`"Name"`) — они не проходят рефакторинг.

---

## MVVM — лучшие практики для Senior

### ViewModel First vs View First

**ViewModel First:**
```csharp
// DI контейнер создаёт VM
services.AddTransient<MainViewModel>();

// View подхватывает VM через DataTemplate
<DataTemplate DataType="{x:Type vm:MainViewModel}">
  <views:MainView />
</DataTemplate>

// Navigation — через DataTemplate resolution
var vm = _serviceProvider.GetRequiredService<MainViewModel>();
var template = FindResourceForType(vm.GetType());
var view = template.LoadContent();
```

**Когда:** Навигация через DI, modular apps, тестирование (VM можно создать без View).

**View First:**
```csharp
// View создаётся, в конструкторе получает VM из DI
public class MainView : Window
{
    public MainView(MainViewModel vm)
    {
        DataContext = vm;
        InitializeComponent();
    }
}
```

**Когда:** Простые приложения, code-behind навигация.

### Composition — Module-based MVVM

```csharp
// Parent VM содержит child VM
public class DashboardViewModel : ViewModelBase
{
    public DashboardViewModel(
        UserListViewModel userList,
        StatisticsViewModel stats,
        AlertsViewModel alerts)
    {
        UserList = userList;
        Statistics = stats;
        Alerts = alerts;
    }
    
    public UserListViewModel UserList { get; }
    public StatisticsViewModel Statistics { get; }
    public AlertsViewModel Alerts { get; }
}
```

```xml
<!-- View composition через ContentControl + DataTemplate -->
<Grid>
  <Grid.ColumnDefinitions>
    <ColumnDefinition Width="2*" />
    <ColumnDefinition Width="*" />
  </Grid.ColumnDefinitions>
  
  <ContentControl Content="{Binding UserList}" />
  <StackPanel Grid.Column="1">
    <ContentControl Content="{Binding Statistics}" />
    <ContentControl Content="{Binding Alerts}" />
  </StackPanel>
</Grid>
```

### Service Locator — антипаттерн

```csharp
// ❌ Service Locator — скрытые зависимости
public class MyViewModel
{
    public MyViewModel()
    {
        var service = App.ServiceLocator.Get<IMyService>();
        // Невозможно понять зависимости по конструктору
        // Трудно тестировать (не подменить в test)
    }
}

// ✅ DI — явные зависимости
public class MyViewModel
{
    public MyViewModel(IMyService service, ILogger<MyViewModel> logger)
    {
        // Зависимости видны, легко mock
    }
}
```

### Messenger (WeakReference-based)

```csharp
// CommunityToolkit.Mvvm — WeakReferenceMessenger
public class ItemSelectedMessage : ValueChangedMessage<Item>
{
    public ItemSelectedMessage(Item value) : base(value) { }
}

// Sender
WeakReferenceMessenger.Default.Send(new ItemSelectedMessage(selectedItem));

// Receiver (регистрация в конструкторе)
WeakReferenceMessenger.Default.Register<ItemSelectedMessage>(this, (r, m) =>
{
    SelectedItem = m.Value;
});

// Unsubscribe (важно!)
WeakReferenceMessenger.Default.Unregister<ItemSelectedMessage>(this);
```

---

## Чек-лист

- [ ] WeakEvent: утечки при подписке на события
- [ ] WeakEventManager: generic и кастомный
- [ ] INotifyPropertyChanged: SetProperty, CallerMemberName
- [ ] CommunityToolkit.Mvvm: [ObservableProperty], [RelayCommand]
- [ ] MVVM ViewModel First vs View First
- [ ] DI вместо Service Locator
- [ ] Messenger: WeakReference, отписка
- [ ] Composition: ContentControl + DataTemplate для вложенных VM

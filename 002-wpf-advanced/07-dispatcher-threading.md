# Dispatcher и многопоточность в WPF

---

## Dispatcher — основы

**Факты:**
- **Один UI thread** на окно (обычно один на всё приложение)
- Dispatcher = очередь сообщений (Win32 message loop)
- Все операции с UI должны выполняться в UI thread
- Исключение: frozen Freezable объекты (thread-safe)

```csharp
// Проверка: в UI ли мы?
if (!Application.Current.Dispatcher.CheckAccess())
{
    // Мы в background thread
}

// DispatcherOperation — можно отследить статус
DispatcherOperation op = Dispatcher.CurrentDispatcher.InvokeAsync(() =>
{
    UpdateUI();
});

// Статусы: Pending, Executing, Completed, Aborted
if (op.Status == DispatcherOperationStatus.Pending)
    op.Wait();  // Блокирует вызывающий поток до выполнения
```

---

## Dispatcher Priority

| Приоритет | Описание | Когда |
|---|---|---|
| `ContextIdle` | После всех сообщений UI | Фоновые обновления |
| `Background` | Фоновые операции | Long-running в UI thread |
| `Input` | После обработки ввода (Keyboard, Mouse) | Реакция на ввод |
| `Render` | После рендеринга | Изменения, которые должны быть видны |
| `DataBind` | После Data Binding | Обновление binding |
| `Normal` | Обычные операции | По умолчанию |
| `Send` | Немедленно (синхронно) | Критические операции |

**Порядок выполнения:**

```
Send → (все остальные по очереди) → после каждого Send: Input → Render → DataBind → Normal → Background → ContextIdle
```

```csharp
// Background — не блокирует UI, выполняется когда UI свободен
await Application.Current.Dispatcher.InvokeAsync(() =>
{
    StatusBar.Text = "Processing...";
}, DispatcherPriority.Background);

// Send — немедленно
Application.Current.Dispatcher.Invoke(() => CriticalUpdate(), DispatcherPriority.Send);
```

**Факт:** `DispatcherPriority.Background` позволяет UI оставаться отзывчивым при длительных операциях — dispatchable-операции выполняются между сообщениями Windows.

---

## Invoke vs InvokeAsync

```csharp
// Invoke — синхронный (блокирует вызывающий поток)
Application.Current.Dispatcher.Invoke(() => UpdateUI());

// InvokeAsync — асинхронный (возвращает DispatcherOperation)
await Application.Current.Dispatcher.InvokeAsync(() => UpdateUI());

// BeginInvoke — устарел (синтаксический сахар для InvokeAsync)
Application.Current.Dispatcher.BeginInvoke(new Action(UpdateUI));
```

| Метод | Возвращает | Блокирует | Когда |
|---|---|---|---|
| `Invoke` | void / T | Да (caller) | Нужен результат синхронно |
| `InvokeAsync` | `DispatcherOperation` / `DispatcherOperation<T>` | Нет | async/await контекст |
| `BeginInvoke` | `DispatcherOperation` | Нет | Legacy |

**Факт:** `Invoke` может вызвать deadlock, если вызывающий поток заблокирован в ожидании другого ресурса:

```csharp
// ❌ Deadlock
Task.Run(() =>
{
    Dispatcher.CurrentDispatcher.Invoke(() => { });  // Ждёт UI thread
});
// UI thread может быть занят ожиданием Task — deadlock!

// ✅ Safe
await Application.Current.Dispatcher.InvokeAsync(() => UpdateUI());
```

---

## Background Operations

### Task + Dispatcher (рекомендуется)

```csharp
public async Task LoadDataAsync()
{
    // 1. Показываем loading (UI thread)
    IsLoading = true;
    
    try
    {
        // 2. Background работа (ThreadPool)
        var data = await Task.Run(() => _service.FetchLargeData());
        
        // 3. Автоматически возвращаемся в UI thread
        // (если SynchronizationContext захвачен)
        Items = new ObservableCollection<Item>(data);
    }
    finally
    {
        IsLoading = false;
    }
}

// Без захвата SynchronizationContext
public async Task LoadDataNoContextAsync()
{
    var data = await Task.Run(() => _service.FetchData())
        .ConfigureAwait(false);  // Не захватываем UI контекст
    
    // ❌ Здесь НЕ в UI thread
    // Items = data; — InvalidOperationException!
    
    // Явный Dispatcher
    await Application.Current.Dispatcher.InvokeAsync(() =>
    {
        Items = data;
    });
}
```

### BackgroundWorker (устарел)

```csharp
// Legacy — не используйте в новых проектах
var worker = new BackgroundWorker
{
    WorkerReportsProgress = true,
    WorkerSupportsCancellation = true
};

worker.DoWork += (s, e) =>
{
    for (int i = 0; i < 100; i++)
    {
        if (worker.CancellationPending) break;
        worker.ReportProgress(i);
        Thread.Sleep(100);
    }
};

worker.ProgressChanged += (s, e) =>
{
    ProgressBar.Value = e.ProgressPercentage;  // UI thread
};

worker.RunWorkerCompleted += (s, e) =>
{
    if (e.Error != null) { /* handle */ }
    if (e.Cancelled) { /* handle */ }
};

worker.RunWorkerAsync();
```

---

## Multi-threaded UI — Dispatcher для нескольких окон

```csharp
// Создание окна в отдельном UI thread (редко, но бывает нужно)
var thread = new Thread(() =>
{
    var window = new SecondaryWindow();
    window.Show();
    
    System.Windows.Threading.Dispatcher.Run();  // Запуск message loop
});

thread.SetApartmentState(ApartmentState.STA);  // WPF требует STA
thread.IsBackground = true;
thread.Start();

// Доступ к Dispatcher другого окна
secondaryWindow.Dispatcher.Invoke(() => secondaryWindow.Update());
```

**Факт:** WPF требует STA (Single Thread Apartment). Все UI threads должны быть STA.

---

## Чек-лист

- [ ] Dispatcher.CheckAccess() — проверка UI thread
- [ ] DispatcherPriority: Background, Input, Render, Normal, Send
- [ ] InvokeAsync vs Invoke — async vs sync
- [ ] Deadlock: Invoke из Task → блокировка
- [ ] Task.Run + async/await для background
- [ ] BackgroundWorker — legacy, не использовать
- [ ] Multi-window: отдельные UI threads с STA

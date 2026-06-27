# Thread vs Task vs ThreadPool — факты

---

## Сравнительная таблица

| Критерий | `Thread` | `ThreadPool` | `Task` (TPL) |
|---|---|---|---|
| **Создание** | `new Thread().Start()` | `QueueUserWorkItem` | `Task.Run()` / `new Task().Start()` |
| **Стек** | ~1 MB (commit) | Переиспользуется из пула | Переиспользуется из пула |
| **Управление** | Ручное (Join, Abort) | Pул (авто) | TPL (планировщик, продолжения) |
| **Результат** | Нет | Нет | `Task<T>` |
| **Отмена** | `Thread.Abort()` — опасно | Нет | `CancellationToken` |
| **Исключения** | Теряются (unhandled → AppDomain) | Теряются | await пробрасывает |
| **Асинхронность** | Sync (блокирует) | Sync | async/await |
| **Параметры** | ParameterizedThreadStart | WaitCallback (object?) | Action / Func<Task> |
| **Foreground/Background** | Можно настроить | Всегда Background | Background (ThreadPool) |

**Факт:** `new Thread()` аллоцирует **~1 MB** под стек (commit). 1000 потоков = 1 GB только стеков. ThreadPool решает эту проблему.

**Факт:** `Thread.Abort()` помечен как `[Obsolete]` в .NET 5+ и **не гарантирует завершение** — может быть проигнорирован или вызвать утечку ресурсов.

---

## Thread — когда использовать

```csharp
// 1. STA поток (COM interop, WPF)
var staThread = new Thread(() =>
{
    var comObject = new SomeComClass();
    comObject.DoWork();
    Dispatcher.Run();  // Message loop
});
staThread.SetApartmentState(ApartmentState.STA);
staThread.Start();

// 2. Long-running операция (не на ThreadPool)
var longThread = new Thread(() =>
{
    while (!_cts.Token.IsCancellationRequested)
    {
        PollHardware();
        Thread.Sleep(50);
    }
})
{
    IsBackground = true,
    Priority = ThreadPriority.BelowNormal,
    Name = "Hardware Polling Thread"
};
longThread.Start();
```

**Факт:** `Background = true` — поток не держит процесс живым. `Foreground` (по умолчанию) — процесс не завершится, пока поток работает.

**Факт:** Thread priority — редко нужно. `ThreadPriority.Highest` может вызвать **starvation** других потоков.

---

## ThreadPool — детали

**Алгоритм управления:**
- Min threads = CPU count (обычно)
- Max threads = ~1000 (можно изменить)
- **Hill climbing** — алгоритм динамического регулирования

```csharp
// Настройка ThreadPool (до первого использования)
ThreadPool.SetMinThreads(workerThreads: 4, completionPortThreads: 4);
ThreadPool.SetMaxThreads(workerThreads: 50, completionPortThreads: 50);

// Проверка текущих значений
ThreadPool.GetAvailableThreads(out int workers, out int completionPorts);
ThreadPool.GetMinThreads(out int minWorkers, out int minPorts);

// Использование
ThreadPool.QueueUserWorkItem(state =>
{
    // state — object параметр
    DoWork((int)state!);
}, 42);

// Без параметра
ThreadPool.QueueUserWorkItem(_ => DoWork());
```

**Факт:** **IOCP (I/O Completion Ports)** — отдельный пул для async I/O (FileStream, Socket, HTTP). `ThreadPool.GetAvailableThreads` показывает два пула.

**Факт:** Если все threads пула заняты, новые задачи **ждут** в очереди. Никаких новых потоков не создаётся, пока не истечёт время ожидания (hill climbing).

---

## Task — современный подход

```csharp
// Task.Run — на ThreadPool
Task.Run(() => DoWork());

// LongRunning — новый поток (не ThreadPool)
Task.Factory.StartNew(() => DoWork(),
    TaskCreationOptions.LongRunning);

// Task с состоянием
var task = new Task<int>(() => Compute(), TaskCreationOptions.AttachedToParent);
task.Start();

// Parent/Child tasks
var parent = Task.Run(() =>
{
    var child = Task.Run(() => ChildWork());
    // child "attached" к parent? Нет, по умолчанию detached
});

// Attached child — parent завершится только после child
var parentWithChild = Task.Factory.StartNew(() =>
{
    Task.Factory.StartNew(() => ChildWork(),
        TaskCreationOptions.AttachedToParent);
});
```

**Факт:** `Task.Run()` = `Task.Factory.StartNew()` с `TaskScheduler.Default` (ThreadPool). Для LongRunning используйте `Task.Factory.StartNew` явно.

**Факт:** `AttachedToParent` — parent Task не перейдёт в `RanToCompletion`, пока все attached children не завершатся. Полезно для композиции, но может вызвать deadlock.

---

## Бенчмарк (принципиальный)

| Операция | Thread | ThreadPool | Task |
|---|---|---|---|
| 10 000 коротких задач | ❌ (1 GB stack) | ✅ (1 pool) | ✅ (1 pool + less overhead) |
| 1 долгая задача | ✅ (выделенный поток) | ❌ (занимает thread в пуле) | ✅ (LongRunning) |
| async I/O | ❌ (блокировка) | ❌ (блокировка) | ✅ (IOCP, без потока) |
| Возврат результата | ❌ (callback) | ❌ (callback) | ✅ (Task<T>) |
| Отмена | ❌ (Abort) | ❌ | ✅ (CancellationToken) |

---

## Чек-лист

- [ ] Thread: ~1 MB стек, STA, long-running
- [ ] ThreadPool: hill climbing, IOCP, min/max threads
- [ ] Task: Run vs Factory.StartNew, LongRunning, AttachedToParent
- [ ] Thread.Abort — устарел, не использовать
- [ ] Background vs Foreground thread
- [ ] Когда Thread (STA, COM) vs Task (всё остальное)

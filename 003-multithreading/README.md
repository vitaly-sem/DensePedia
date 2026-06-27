# Многопоточность в .NET — разделы

| # | Файл | Темы |
|---|------|------|
| 1 | [Thread vs Task vs ThreadPool](01-thread-task-threadpool.md) | Сравнение, когда что использовать, STA/MTA |
| 2 | [async/await — внутреннее устройство](02-async-await.md) | State Machine, SynchronizationContext, ConfigureAwait, ValueTask |
| 3 | [Task — продвинутые сценарии](03-task-advanced.md) | WhenAll/WhenAny, TaskCompletionSource, CancellationToken, TaskScheduler |
| 4 | [Синхронизация](04-synchronization.md) | lock, SemaphoreSlim, ReaderWriterLockSlim, Interlocked, SpinLock, Barrier |
| 5 | [Concurrent Collections](05-concurrent-collections.md) | ConcurrentDictionary, ConcurrentQueue, Channel, BlockingCollection, Immutable |
| 6 | [Parallel и PLINQ](06-parallel-plinq.md) | Parallel.For/ForEach/Invoke, PLINQ, DegreeOfParallelism |
| 7 | [TLS и TPL Dataflow](07-tls-dataflow.md) | ThreadLocal, AsyncLocal, TPL Dataflow, Pipeline, Backpressure |
| 8 | [Memory Model и низкий уровень](08-memory-model.md) | volatile, MemoryBarrier, DCL, Interlocked Memory, hazar pointers |

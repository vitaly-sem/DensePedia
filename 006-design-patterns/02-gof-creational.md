# GoF — Порождающие шаблоны (Creational)

---

## Singleton

**Факты:**
- Гарантирует **один экземпляр** класса
- Критикуется: усложняет тестирование (глобальное состояние)
- Используйте DI (AddSingleton) вместо ручного Singleton

```csharp
// ✅ Lazy<T> — потокобезопасен, минимальный overhead
public sealed class CacheService
{
    private static readonly Lazy<CacheService> _instance = 
        new(() => new CacheService());
    
    public static CacheService Instance => _instance.Value;
    
    private CacheService() { }
}

// ❌ Double-checked locking — устарел
// ❌ Static constructor — медленнее Lazy
// ❌ Static field — нет lazy initialization
```

**Факт:** `Lazy<T>` с `LazyThreadSafetyMode.ExecutionAndPublication` — инициализация одним потоком, остальные ждут.

**Факт:** Singleton через DI — предпочтительнее:

```csharp
builder.Services.AddSingleton<ICacheService, RedisCacheService>();
// Тот же Singleton, но тестируемый (можно подменить в тестах)
```

---

## Factory Method

```csharp
// Фабричный метод — делегирование создания подклассам
public abstract class ReportGenerator
{
    public abstract IReport CreateReport();  // Factory Method
    
    public async Task<byte[]> GenerateAsync()
    {
        var report = CreateReport();
        await report.LoadDataAsync();
        return await report.RenderAsync();
    }
}

public class PdfReportGenerator : ReportGenerator
{
    public override IReport CreateReport() => new PdfReport();
}

public class ExcelReportGenerator : ReportGenerator
{
    public override IReport CreateReport() => new ExcelReport();
}

// Использование
ReportGenerator gen = format switch
{
    "pdf" => new PdfReportGenerator(),
    "xlsx" => new ExcelReportGenerator(),
    _ => throw new NotSupportedException()
};
var bytes = await gen.GenerateAsync();
```

---

## Abstract Factory

```csharp
// Абстрактная фабрика — семейство связанных объектов
public interface IUIFactory
{
    IButton CreateButton();
    ICheckbox CreateCheckbox();
    ITextField CreateTextField();
}

public class LightThemeFactory : IUIFactory
{
    public IButton CreateButton() => new LightButton();
    public ICheckbox CreateCheckbox() => new LightCheckbox();
    public ITextField CreateTextField() => new LightTextField();
}

public class DarkThemeFactory : IUIFactory
{
    public IButton CreateButton() => new DarkButton();
    // ...
}

// Использование
public class App
{
    private readonly IUIFactory _ui;
    
    public App(IUIFactory ui) => _ui = ui;  // DI фабрики
    
    public void Render()
    {
        var btn = _ui.CreateButton();
        var chk = _ui.CreateCheckbox();
        // btn, chk — согласованной темы
    }
}
```

---

## Builder

**Факт:** Builder используется когда создание объекта — сложный многошаговый процесс.

```csharp
// Fluent Builder
public class EmailBuilder
{
    private readonly Email _email = new();
    
    public EmailBuilder From(string address) { _email.From = address; return this; }
    public EmailBuilder To(string address) { _email.To = address; return this; }
    public EmailBuilder Subject(string subject) { _email.Subject = subject; return this; }
    public EmailBuilder Body(string body) { _email.Body = body; return this; }
    public EmailBuilder WithAttachment(string path) { _email.Attachments.Add(path); return this; }
    
    public Email Build()
    {
        if (string.IsNullOrEmpty(_email.To))
            throw new InvalidOperationException("Recipient required");
        return _email;
    }
}

// Использование
var email = new EmailBuilder()
    .From("noreply@company.com")
    .To("user@example.com")
    .Subject("Welcome!")
    .Body("Thank you for registering")
    .Build();
```

**Факт:** `record` с `with` — встроенный builder в C# 9+:

```csharp
public record EmailRequest(string To, string Subject, string Body);

var base = new EmailRequest("", "", "");
var email = base with { To = "user@ex.com", Subject = "Hi" };
```

---

## Prototype

**Факт:** Клонирование объектов. В C# — через `ICloneable` (лучше свой интерфейс).

```csharp
public interface IPrototype<T> where T : class
{
    T DeepClone();
}

public class Configuration : IPrototype<Configuration>
{
    public string ConnectionString { get; set; }
    public Dictionary<string, string> Settings { get; set; } = new();
    public TimeSpan Timeout { get; set; }
    
    public Configuration DeepClone()
    {
        // Deep copy через serialization
        var json = JsonSerializer.Serialize(this);
        return JsonSerializer.Deserialize<Configuration>(json)!;
    }
}

// Использование
var original = new Configuration { /* ... */ };
var cloned = original.DeepClone();  // Независимая копия
```

---

## Чек-лист

- [ ] Singleton: Lazy<T>, DI предпочтительнее
- [ ] Factory Method: виртуальный конструктор, Template Method с созданием
- [ ] Abstract Factory: семейство объектов, согласованность
- [ ] Builder: сложная пошаговая инициализация, fluent
- [ ] Prototype: ICloneable, deep vs shallow copy

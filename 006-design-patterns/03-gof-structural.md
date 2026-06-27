# GoF — Структурные шаблоны (Structural)

---

## Adapter

```csharp
// Старый интерфейс (Legacy)
public interface IXmlSerializer
{
    string SerializeToXml(object obj);
}

// Новый интерфейс (Target)
public interface IJsonSerializer
{
    string SerializeToJson<T>(T obj);
}

// Adapter: Legacy → New
public class XmlToJsonAdapter : IJsonSerializer
{
    private readonly IXmlSerializer _xmlSerializer;
    
    public XmlToJsonAdapter(IXmlSerializer xmlSerializer)
        => _xmlSerializer = xmlSerializer;
    
    public string SerializeToJson<T>(T obj)
    {
        var xml = _xmlSerializer.SerializeToXml(obj);
        return ConvertXmlToJson(xml);  // Преобразование
    }
}

// Использование: старый код работает через новый интерфейс
var adapter = new XmlToJsonAdapter(new LegacyXmlSerializer());
var json = adapter.SerializeToJson(myObject);
```

**Факт:** В .NET — `TextReader` → `StreamReader` адаптер, `IEnumerable` → `IQueryable` адаптер (через AsQueryable).

---

## Decorator

**Факт:** Добавляет поведение объекту **динамически**, без изменения его класса.

```csharp
public interface IDataService
{
    Task<Data> GetAsync(int id);
}

// Base implementation
public class DataService : IDataService
{
    public async Task<Data> GetAsync(int id) { /* ... */ }
}

// Decorator 1: Caching
public class CachedDataService : IDataService
{
    private readonly IDataService _inner;
    private readonly IMemoryCache _cache;
    
    public CachedDataService(IDataService inner, IMemoryCache cache)
        => (_inner, _cache) = (inner, cache);
    
    public async Task<Data> GetAsync(int id)
        => await _cache.GetOrCreateAsync($"data-{id}", 
            async _ => await _inner.GetAsync(id));
}

// Decorator 2: Logging
public class LoggedDataService : IDataService
{
    private readonly IDataService _inner;
    private readonly ILogger _logger;
    
    public LoggedDataService(IDataService inner, ILogger logger)
        => (_inner, _logger) = (inner, logger);
    
    public async Task<Data> GetAsync(int id)
    {
        _logger.LogInformation("Getting data {Id}", id);
        var sw = Stopwatch.StartNew();
        try
        {
            return await _inner.GetAsync(id);
        }
        finally
        {
            _logger.LogInformation("Got data {Id} in {Ms}ms", id, sw.ElapsedMilliseconds);
        }
    }
}

// Композиция декораторов
builder.Services.AddScoped<IDataService>(sp =>
{
    var inner = new DataService();
    var cached = new CachedDataService(inner, sp.GetRequiredService<IMemoryCache>());
    return new LoggedDataService(cached, sp.GetRequiredService<ILogger<DataService>>());
});
```

**Факт:** В .NET `Stream` — классический Decorator: `FileStream` → `BufferedStream` → `GZipStream` → `CryptoStream`.

---

## Proxy

**Факт:** Заместитель — контроль доступа к объекту (Lazy, Virtual, Protection, Remote).

```csharp
// Virtual Proxy — ленивая загрузка
public class LazyImageProxy : IImage
{
    private readonly string _path;
    private HighResImage? _realImage;
    
    public LazyImageProxy(string path) => _path = path;
    
    public void Display()
    {
        _realImage ??= new HighResImage(_path);  // Lazy load
        _realImage.Display();
    }
}

// Protection Proxy — контроль доступа
public class AdminOnlyProxy : IService
{
    private readonly IService _inner;
    private readonly ICurrentUser _user;
    
    public AdminOnlyProxy(IService inner, ICurrentUser user)
        => (_inner, _user) = (inner, user);
    
    public async Task ExecuteAsync()
    {
        if (!_user.IsInRole("Admin"))
            throw new UnauthorizedAccessException();
        await _inner.ExecuteAsync();
    }
}

// Remote Proxy — .NET Remoting / gRPC
// WCF, gRPC client — transparent proxy для удалённого вызова
```

**Факт:** Castle DynamicProxy, DispatchProxy — создают Proxy динамически в runtime.

---

## Facade

```csharp
// Facade — упрощённый интерфейс к сложной подсистеме
public class OrderPlacementFacade
{
    private readonly IInventoryService _inventory;
    private readonly IPaymentService _payment;
    private readonly IShippingService _shipping;
    private readonly INotificationService _notifications;
    
    public async Task<OrderResult> PlaceOrderAsync(OrderRequest request)
    {
        // 1. Проверка inventory
        var availability = await _inventory.CheckAsync(request.Items);
        if (!availability.AllAvailable) return OrderResult.Failed("Not available");
        
        // 2. Резервирование
        await _inventory.ReserveAsync(request.Items);
        
        // 3. Платёж
        var payment = await _payment.ChargeAsync(request.Total, request.PaymentInfo);
        if (!payment.Success)
        {
            await _inventory.ReleaseAsync(request.Items);
            return OrderResult.Failed("Payment failed");
        }
        
        // 4. Доставка
        var shipment = await _shipping.ScheduleAsync(request.Address, request.Items);
        
        // 5. Уведомление
        await _notifications.SendOrderConfirmationAsync(request.Email, shipment.TrackingNumber);
        
        return OrderResult.Success(shipment.TrackingNumber);
    }
}
```

---

## Flyweight

**Факт:** Разделяет общее состояние между объектами для экономии памяти.

```csharp
// Flyweight — разделяемое состояние (intrinsic)
public class CharacterFlyweight
{
    public char Symbol { get; }
    public string FontFamily { get; }
    public int FontSize { get; }
    
    public CharacterFlyweight(char symbol, string fontFamily, int fontSize)
        => (Symbol, FontFamily, FontSize) = (symbol, fontFamily, fontSize);
}

// Flyweight Factory
public class CharacterFactory
{
    private readonly Dictionary<string, CharacterFlyweight> _cache = new();
    
    public CharacterFlyweight Get(char symbol, string font, int size)
    {
        var key = $"{symbol}|{font}|{size}";
        
        if (!_cache.TryGetValue(key, out var flyweight))
        {
            flyweight = new CharacterFlyweight(symbol, font, size);
            _cache[key] = flyweight;
        }
        
        return flyweight;
    }
}

// Использование: внешнее состояние (extrinsic) — позиция
// Внутреннее — символ + шрифт (разделяемое)
```

---

## Bridge

```csharp
// Bridge — разделение абстракции и реализации
public interface IRenderer  // Implementation
{
    void RenderCircle(float radius);
    void RenderSquare(float side);
}

// Concrete implementations
public class VectorRenderer : IRenderer { /* ... */ }
public class RasterRenderer : IRenderer { /* ... */ }

// Abstraction
public abstract class Shape
{
    protected readonly IRenderer _renderer;
    protected Shape(IRenderer renderer) => _renderer = renderer;
    public abstract void Draw();
}

public class Circle : Shape
{
    private readonly float _radius;
    public Circle(IRenderer renderer, float radius) : base(renderer) => _radius = radius;
    public override void Draw() => _renderer.RenderCircle(_radius);
}

// Bridge: Circle (abstraction) + VectorRenderer (implementation)
var circle = new Circle(new VectorRenderer(), 5);
```

---

## Composite

```csharp
// Composite — дерево «часть-целое»
public interface IFileSystemNode
{
    string Name { get; }
    long GetSize();
}

public class File : IFileSystemNode
{
    public string Name { get; }
    private readonly long _size;
    public long GetSize() => _size;
}

public class Directory : IFileSystemNode
{
    public string Name { get; }
    private readonly List<IFileSystemNode> _children = new();
    
    public void Add(IFileSystemNode node) => _children.Add(node);
    public long GetSize() => _children.Sum(c => c.GetSize());  // Делегирование
}
```

---

## Чек-лист

- [ ] Adapter: конвертация интерфейсов
- [ ] Decorator: динамическое добавление поведения (Stream, Middleware)
- [ ] Proxy: Lazy, Protection, Remote
- [ ] Facade: упрощение сложных подсистем
- [ ] Flyweight: разделение состояния, экономия памяти
- [ ] Bridge: абстракция отдельно от реализации
- [ ] Composite: дерево, единообразие leaf и composite

# Data Leakage Prevention в .NET-приложениях

---

## Логирование — не логировать PII

**Факты:**
- Логи — частая причина утечек PII (ELK, Splunk, Seq — данные остаются надолго)
- GDPR: если в логах есть PII, это обработка персональных данных (нужно согласие)
- PCI DSS: нельзя логировать полные номера карт, CVV, PIN

### Serilog — маскирование PII

```csharp
// 1. Структурированное логирование (всегда!)
// 2. Маскировщик PII

// Кастомный маскировщик
public class PiiMaskingEnricher : ILogEventEnricher
{
    private static readonly Regex EmailRegex = new(@"\b[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}\b", RegexOptions.IgnoreCase);
    private static readonly Regex SsnRegex = new(@"\b\d{3}-\d{2}-\d{4}\b");
    private static readonly Regex CcRegex = new(@"\b(?:\d[ -]*?){13,16}\b");
    
    public void Enrich(LogEvent logEvent, ILogEventPropertyFactory propertyFactory)
    {
        if (logEvent.MessageTemplate.Text.Contains("password", StringComparison.OrdinalIgnoreCase))
            return;  // Skip entirely? No, mask:
        
        // Mask message
        var message = logEvent.RenderMessage();
        message = EmailRegex.Replace(message, "***@***");
        message = SsnRegex.Replace(message, "***-**-****");
        message = CcRegex.Replace(message, m => $"****-****-****-{m.Value[^4..]}");
        
        logEvent.AddOrUpdateProperty(propertyFactory.CreateProperty("MaskedMessage", message));
    }
}

// Настройка Serilog
Log.Logger = new LoggerConfiguration()
    .Enrich.With<PiiMaskingEnricher>()
    .WriteTo.Seq("http://seq:5341")
    .CreateLogger();

// Альтернатива: Destructuring — отбрасывание sensitive полей
public class UserDestructuringPolicy : IDestructuringPolicy
{
    public bool TryDestructure(object value, ILogEventPropertyValueFactory factory, out LogEventPropertyValue result)
    {
        if (value is User user)
        {
            result = new StructureValue(new[]
            {
                new LogEventProperty("Id", new ScalarValue(user.Id)),
                new LogEventProperty("Name", new ScalarValue(user.Name)),
                // Не включаем Email, Phone, SSN
            });
            return true;
        }
        result = null;
        return false;
    }
}

// Использование:
// ❌ Нелогировать:
_logger.LogInformation("User logged in: {Email}, {Password}", email, password);

// ✅ Правильно:
_logger.LogInformation("User {UserId} logged in from {IpAddress}", userId, ip);
```

**Факт:** В Serilog `@` operator — serializes the object (destructure). Без `@` — `ToString()`. Сensitive objects нужно либо маскировать, либо исключать поля.

### Filtering sensitive parameters

```csharp
// ASP.NET Core — маскирование параметров запроса
builder.Services.AddControllers(options =>
{
    options.Filters.Add<LoggingFilter>();
});

public class LoggingFilter : IActionFilter
{
    public void OnActionExecuting(ActionExecutingContext context)
    {
        var sensitiveParams = new[] { "password", "ssn", "creditcard", "token" };
        
        foreach (var param in context.ActionArguments)
        {
            if (sensitiveParams.Any(s => param.Key.Contains(s, StringComparison.OrdinalIgnoreCase)))
            {
                _logger.LogInformation("Parameter {Name} omitted from logs (sensitive)", param.Key);
                // Не логировать значение
            }
            else
            {
                _logger.LogInformation("Parameter {Name} = {Value}", param.Key, param.Value);
            }
        }
    }
}
```

---

## API Response — маскирование данных

```csharp
// Автоматическое маскирование PII в API responses
public class PiiMaskingMiddleware
{
    private readonly RequestDelegate _next;
    
    public PiiMaskingMiddleware(RequestDelegate next) => _next = next;
    
    public async Task InvokeAsync(HttpContext context)
    {
        var originalBody = context.Response.Body;
        
        using var responseBody = new MemoryStream();
        context.Response.Body = responseBody;
        
        await _next(context);
        
        if (context.Response.ContentType?.Contains("json") == true)
        {
            responseBody.Seek(0, SeekOrigin.Begin);
            var body = await new StreamReader(responseBody).ReadToEndAsync();
            
            // Маскирование PII в JSON response
            var masked = MaskPiiInJson(body);
            
            var bytes = Encoding.UTF8.GetBytes(masked);
            await originalBody.WriteAsync(bytes);
        }
        else
        {
            responseBody.Seek(0, SeekOrigin.Begin);
            await responseBody.CopyToAsync(originalBody);
        }
    }
}

// Или через JsonSerializer — contract-based
public class UserResponse
{
    public int Id { get; set; }
    public string Name { get; set; }
    
    [JsonIgnore]  // Никогда не возвращать
    public string SSN { get; set; }
    
    [JsonConverter(typeof(EmailMaskConverter))]  // Маскировать
    public string Email { get; set; }
}

public class EmailMaskConverter : JsonConverter<string>
{
    public override string? Read(ref Utf8JsonReader reader, Type typeToConvert, JsonSerializerOptions options)
        => reader.GetString();
    
    public override void Write(Utf8JsonWriter writer, string value, JsonSerializerOptions options)
    {
        var parts = value?.Split('@');
        var masked = parts?.Length == 2
            ? $"{parts[0][0]}***@{parts[1]}"
            : "***";
        writer.WriteStringValue(masked);
    }
}
```

---

## Exception Handling — не показывать sensitive details

```csharp
// ❌ Опасно — stack trace и внутренние детали наружу
return BadRequest(ex.Message);

// ✅ Безопасно — generic error, sensitive details в лог
try
{
    return Ok(await ProcessAsync(data));
}
catch (PiiValidationException ex)
{
    _logger.LogWarning(ex, "PII validation failed for user {UserId}", userId);
    return Problem("Invalid data format");
}
catch (Exception ex)
{
    _logger.LogError(ex, "Unexpected error processing user {UserId}", userId);
    return Problem("An error occurred", statusCode: 500);
}

// ProblemDetails — стандартный формат ошибок (RFC 7807)
app.UseExceptionHandler(handler =>
{
    handler.Run(async context =>
    {
        var problem = new ProblemDetails
        {
            Status = 500,
            Title = "Internal Server Error",
            Detail = "An unexpected error occurred",  // Не ex.Message!
            Instance = context.Request.Path
        };
        
        context.Response.StatusCode = 500;
        context.Response.ContentType = "application/problem+json";
        await JsonSerializer.SerializeAsync(context.Response.Body, problem);
    });
});
```

---

## SecureString и защита в памяти

**Факты:**
- `string` в .NET — **immutable**, не удаляется из памяти (ждёт GC)
- `SecureString` — шифрует в памяти (`CryptProtectMemory`), можно очистить
- **НО:** `SecureString` считается устаревшим начиная с .NET 5+. Используйте `ArrayPool<byte>` или `ReadOnlyMemory<byte>`

```csharp
// SecureString (legacy, .NET Framework)
using var secure = new SecureString();
secure.AppendChar('p');
secure.AppendChar('a');
secure.AppendChar('s');
secure.AppendChar('s');
secure.MakeReadOnly();

// Преобразование обратно (опасно!)
IntPtr ptr = Marshal.SecureStringToBSTR(secure);
try
{
    var plain = Marshal.PtrToStringBSTR(ptr);
    // Используем немедленно и не сохраняем
}
finally
{
    Marshal.ZeroFreeBSTR(ptr);
}

// Современный подход ( .NET 6+)
public class SensitiveBuffer : IDisposable
{
    private byte[] _buffer;
    private GCHandle _handle;
    private bool _disposed;
    
    public SensitiveBuffer(int size)
    {
        _buffer = GC.AllocateArray<byte>(size, pinned: true);  // Pinned — не перемещается GC
        _handle = GCHandle.Alloc(_buffer, GCHandleType.Pinned);
    }
    
    public Span<byte> Span => _buffer.AsSpan();
    
    public void Zeroize()
    {
        CryptographicOperations.ZeroMemory(_buffer);  // Constant-time zero
    }
    
    public void Dispose()
    {
        if (!_disposed)
        {
            Zeroize();
            _handle.Free();
            _disposed = true;
        }
    }
}

// Использование
using var buffer = new SensitiveBuffer(1024);
// Заполняем buffer, используем, очищается автоматически
```

**Факт:** `GC.AllocateArray<T>(pinned: true)` — массив, который GC не перемещает. Позволяет безопасно занулить sensitive data.

**Факт:** `CryptographicOperations.ZeroMemory` — гарантированно не оптимизируется JIT (в отличие от простого `Array.Clear`).

---

## Data Export / Download — защита

```csharp
// Контроль экспорта данных
[Authorize("CanExportData")]
[HttpPost("export")]
public async Task<IActionResult> ExportData([FromBody] ExportRequest request)
{
    // 1. Rate limit на экспорт
    var exportLimit = await _rateLimiter.CanExportAsync(UserId);
    if (!exportLimit.Allowed)
        return StatusCode(429, "Export rate limit exceeded");
    
    // 2. Аудит экспорта
    await _auditLogger.LogExportAsync(UserId, request.DataType, request.Format);
    
    // 3. Максимальный размер
    if (request.Filter.DateRange > TimeSpan.FromDays(30))
        return BadRequest("Export limited to 30 days");
    
    // 4. Watermark или маркировка
    var data = await _exportService.ExportAsync(request);
    var watermarked = AddWatermark(data, UserId, DateTime.UtcNow);
    
    return File(watermarked, "application/octet-stream", "export.xlsx");
}

// Excel — защита листов паролем
public byte[] ProtectExcel(byte[] excelData, string password)
{
    using var workbook = new XLWorkbook(new MemoryStream(excelData));
    foreach (var ws in workbook.Worksheets)
    {
        ws.Protect(password);  // Защита листа
    }
    // Или Protect workbook для открытия
    workbook.LockPassword = password;
    
    using var ms = new MemoryStream();
    workbook.SaveAs(ms);
    return ms.ToArray();
}
```

---

## Clipboard и Copy Protection

```csharp
// WPF — запрет копирования sensitive данных
public class ProtectedTextBox : TextBox
{
    protected override void OnPreviewExecuted(ExecutedRoutedEventArgs e)
    {
        if (e.Command == ApplicationCommands.Copy ||
            e.Command == ApplicationCommands.Cut)
        {
            if (Tag?.ToString() == "Sensitive")
            {
                e.Handled = true;  // Блокируем копирование
                return;
            }
        }
        base.OnPreviewExecuted(e);
    }
}

// Блокировка PrintScreen (только Windows)
// Через SetWindowDisplayAffinity
[DllImport("user32.dll")]
private static extern bool SetWindowDisplayAffinity(IntPtr hwnd, uint affinity);

public static void PreventScreenCapture(Window window)
{
    var helper = new WindowInteropHelper(window);
    SetWindowDisplayAffinity(helper.Handle, 0x11);  // WDA_MONITOR
}
```

---

## Чек-лист

- [ ] Логи: не логировать PII, структурированное логирование (Serilog), маскирование
- [ ] API: JsonIgnore sensitive поля, мас
- [ ] Exception handling: не показывать stack trace и inner details
- [ ] SecureString / ZeroMemory для sensitive данных в памяти
- [ ] Data export: rate limit, аудит, watermark, защита паролем
- [ ] Clipboard protection: WPF, WinForms
- [ ] Screen capture prevention (WDA_MONITOR)

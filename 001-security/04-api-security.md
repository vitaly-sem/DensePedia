# Безопасность API в .NET — факты + примеры

---

## Rate Limiting

**Факты:**
- Защита от DDoS, brute-force, resource starvation
- ASP.NET 7+ встроенный rate limiter (`System.Threading.RateLimiting`)
- 4 стратегии: FixedWindow, SlidingWindow, TokenBucket, Concurrency

```csharp
builder.Services.AddRateLimiter(options =>
{
    // 1. Fixed Window — N запросов в окне
    options.AddFixedWindowLimiter("Fixed", opt =>
    {
        opt.Window = TimeSpan.FromMinutes(1);
        opt.PermitLimit = 100;
        opt.QueueLimit = 10;         // Queue после превышения
        opt.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
    });
    
    // 2. Sliding Window — плавное ограничение
    options.AddSlidingWindowLimiter("Sliding", opt =>
    {
        opt.Window = TimeSpan.FromMinutes(1);
        opt.SegmentsPerWindow = 10;  // 10 сегментов по 6 секунд
        opt.PermitLimit = 100;
    });
    
    // 3. Token Bucket — burst-капабилити (накопление токенов)
    options.AddTokenBucketLimiter("Burst", opt =>
    {
        opt.TokenLimit = 100;        // Макс. burst
        opt.ReplenishmentPeriod = TimeSpan.FromSeconds(1);
        opt.TokensPerPeriod = 10;    // 10 токенов/сек
    });
    
    // 4. Concurrency — ограничение одновременных запросов
    options.AddConcurrencyLimiter("Concurrent", opt =>
    {
        opt.PermitLimit = 10;        // 10 одновременных
        opt.QueueLimit = 5;
    });
});

app.UseRateLimiter();

// Применение к endpoint
app.MapGet("/api/data", async () => { ... })
   .RequireRateLimiting("Fixed");

// Кастомизация ответа при превышении лимита
app.UseRateLimiter(new RateLimiterOptions
{
    OnRejected = async (context, cancellationToken) =>
    {
        context.HttpContext.Response.StatusCode = 429;
        context.HttpContext.Response.Headers["Retry-After"] = "60";
        await context.HttpContext.Response.WriteAsync(
            """{"error":"Too many requests"}""", cancellationToken);
    }
});
```

**Факт:** Для распределённого rate limiting используйте Redis + `RedisCacheRateLimiter` (NuGet: `AspNetCore.RateLimit.Redis`).

**Факт:** На API Gateway (Ocelot, YARP, Kong) rate limiting делается на уровне шлюза — до контроллера.

---

## CORS (Cross-Origin Resource Sharing)

**Факты:**
- Браузер блокирует запросы с другого origin (protocol + host + port)
- Preflight request (`OPTIONS`) для non-simple запросов
- **Не безопасность** — это защита браузера, API можно вызвать с `curl`

```csharp
// ❌ Слишком широко
builder.Services.AddCors(options => options.AddPolicy("AllowAll", policy =>
    policy.AllowAnyOrigin().AllowAnyMethod().AllowAnyHeader()));

// ✅ Production-ready
builder.Services.AddCors(options =>
{
    options.AddPolicy("ApiCors", policy =>
        policy.WithOrigins("https://app.mycompany.com", "https://admin.mycompany.com")
              .AllowCredentials()                    // cookies, auth headers
              .WithMethods("GET", "POST", "PUT")     // только нужные методы
              .WithHeaders("Content-Type", "Authorization")
              .SetPreflightMaxAge(TimeSpan.FromMinutes(10)));  // кэш OPTIONS
});
```

**Факт:** `AllowCredentials()` + `AllowAnyOrigin()` = **ошибка**. Credentials требует конкретный origin.

**Факт:** Preflight-запросы кэшируются (`Access-Control-Max-Age`). Уменьшите время в dev, увеличьте в prod.

---

## Input Validation

```csharp
// Model Validation (Data Annotations)
public class CreateUserRequest
{
    [Required, MaxLength(100)]
    public string Name { get; set; }
    
    [EmailAddress]
    public string Email { get; set; }
    
    [Range(0, 150)]
    public int Age { get; set; }
    
    [RegularExpression(@"^\+?[1-9]\d{1,14}$")]
    public string Phone { get; set; }
}

// FluentValidation — для сложной валидации
public class CreateUserValidator : AbstractValidator<CreateUserRequest>
{
    public CreateUserValidator()
    {
        RuleFor(x => x.Email).EmailAddress();
        RuleFor(x => x.Name).NotEmpty().MaximumLength(100);
        RuleFor(x => x.Password)
            .MinimumLength(12)
            .Must(p => p.Any(char.IsUpper))
            .Must(p => p.Any(char.IsLower))
            .Must(p => p.Any(char.IsDigit));
        RuleFor(x => x.Email)
            .MustAsync(async (email, ct) => !await IsEmailTakenAsync(email))
            .WithMessage("Email already registered");
    }
}
```

**Факт:** Валидация на **сервере** обязательна всегда. Клиентская валидация — UX, не безопасность.

**Факт:** Decimal rounding — `[RegularExpression(@"^\d+(\.\d{1,2})?$")]` для цен (2 знака после запятой).

---

## Audit Logging

```csharp
[AttributeUsage(AttributeTargets.Method | AttributeTargets.Class)]
public class AuditAttribute : ActionFilterAttribute
{
    private readonly string _actionName;
    
    public AuditAttribute(string actionName = null)
    {
        _actionName = actionName;
    }
    
    public override async Task OnActionExecutionAsync(
        ActionExecutingContext context, ActionExecutionDelegate next)
    {
        var sw = Stopwatch.StartNew();
        var httpContext = context.HttpContext;
        var user = httpContext.User.Identity?.Name ?? "anonymous";
        var ip = httpContext.Connection.RemoteIpAddress?.ToString();
        var action = _actionName ?? context.ActionDescriptor.DisplayName;
        var body = await ReadBodyAsync(httpContext.Request);
        
        // Mask sensitive data
        body = MaskSensitiveData(body);
        
        await next();
        
        sw.Stop();
        
        Logger.LogAudit(
            Action: action,
            User: user,
            Ip: ip,
            Duration: sw.ElapsedMilliseconds,
            StatusCode: context.HttpContext.Response.StatusCode,
            RequestBody: body);
    }
}

// Использование Serilog + Seq/ElasticSearch
// audit_logs-YYYY.MM.DD индекс с retention 1 год
```

**Факт:** GDPR требует логировать кто, когда и какие персональные данные изменял. **Не логируйте сами PII** — логируйте operation + user ID + timestamp.

**Факт:** Audit trail должен быть **immutable** — append-only БД, подписанные логи (можно HMAC).

---

## API Versioning

```csharp
// URL-based (рекомендуется)
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;  // Header: api-supported-versions
    options.ApiVersionReader = new UrlSegmentApiVersionReader();
});

// Header-based (для backward compat)
options.ApiVersionReader = new HeaderApiVersionReader("X-API-Version");

// Deprecation
[ApiVersion(1.0)]
[ApiVersion(2.0, Deprecated = true)]
public class UsersController : Controller { ... }

// Security: старые версии могут иметь уязвимости
// На интервью: спрашивают про версионирование + безопасность
```

---

## Mass Assignment / Over-Posting Protection

**Факты:**
- Атака: злоумышленник добавляет поля в JSON-body, которые не должен контролировать
- Например: регистрация с `"role": "Admin"` или `"isApproved": true`

```csharp
// ❌ Уязвимо — Model Binding привяжет все поля
public class RegisterRequest
{
    public string Username { get; set; }
    public string Password { get; set; }
    public string Role { get; set; }       // Уязвимо!
    public bool IsApproved { get; set; }    // Уязвимо!
}

// ✅ DTO с только разрешёнными полями
public class RegisterRequestDto
{
    [Required] public string Username { get; set; }
    [Required] public string Password { get; set; }
    // Роль и IsApproved не включены — Model Binding проигнорирует
}

// ✅ Explicit binding (ручное)
[HttpPost("/register")]
public async Task<IActionResult> Register([FromBody] RegisterRequestDto dto)
{
    var user = new User
    {
        Username = dto.Username,
        Role = "User",          // Явно задаём
        IsApproved = false,     // Явно задаём
    };
    // ...
}

// ✅ Глобальная защита — запрет лишних полей
builder.Services.AddControllers()
    .AddJsonOptions(options =>
    {
        options.JsonSerializerOptions.UnmappedMemberHandling =
            JsonUnmappedMemberHandling.Disallow;  // .NET 8+
    });
```

**Факт:** `[BindNever]` атрибут и `[JsonIgnore]` предотвращают привязку, но надёжнее — не включать чувствительные поля в DTO совсем.

## GraphQL Security

```csharp
// GraphQL специфичные атаки:
// 1. Deep query (DoS) — злоумышленник создаёт глубоко вложенный запрос
// 2. Batching amplification — несколько запросов в одном HTTP-запросе
// 3. Field suggestion — introspection раскрывает схему

// ✅ Защита (HotChocolate)
builder.Services.AddGraphQLServer()
    .AddQueryType<Query>()
    .AddMutationType<Mutation>()
    .AddMaxExecutionDepthRule(10)           // Ограничение глубины запроса
    .AddMaxComplexityRule(maxComplexity: 100)  // Ограничение сложности
    .AllowIntrospection(                   // Introspection только в dev
        builder.Environment.IsDevelopment());
```

## WebSocket Security

```csharp
// WebSocket — нет CORS защиты браузера, нужна ручная проверка Origin
app.UseWebSockets(new WebSocketOptions
{
    KeepAliveInterval = TimeSpan.FromMinutes(2),
    AllowedOrigins = { "https://app.mycompany.com" }  // Не "*"!
});

app.Use(async (context, next) =>
{
    if (context.WebSockets.IsWebSocketRequest)
    {
        // Проверка Origin (WebSocket не защищён CORS браузером)
        var origin = context.Request.Headers["Origin"].FirstOrDefault();
        if (!IsAllowedOrigin(origin))
        {
            context.Response.StatusCode = 403;
            return;
        }
        
        // Проверка токена (WebSocket не отправляет Authorization header автоматически)
        var token = context.Request.Query["access_token"];
        // ... validate token
        
        using var ws = await context.WebSockets.AcceptWebSocketAsync();
        await HandleWebSocket(ws);
    }
    else await next();
});
```

## Logging Injection / Log Forging

```csharp
// ❌ Уязвимо — злоумышленник может внедрить "User logged in: \r\n[ERROR] Critical error"
_logger.LogInformation("User logged in: {User}", userInput);

// ✅ Безопасно — Serilog, NLog экранируют
// Но структурированные параметры тоже могут содержать \n
_logger.LogInformation("User {User} logged in from {Ip}", userId, ip);

// ✅ Дополнительная защита — санитизация
var sanitized = userInput.Replace(Environment.NewLine, " ").Replace("\r", " ").Replace("\n", " ");
```

**Факт:** Log injection может обмануть SIEM системы (Splunk, ELK) — злоумышленник может подставить фейковые "errors" в логи.

---

## Чек-лист

- [ ] Rate Limiting: Fixed/Sliding/TokenBucket/Concurrency для разных endpoint
- [ ] CORS: конкретные origins, AllowCredentials только с конкретными origin
- [ ] Input Validation: server-side (FluentValidation / Data Annotations)
- [ ] Mass Assignment: DTO без чувствительных полей, UnmappedMemberHandling.Disallow
- [ ] GraphQL: ограничение глубины, сложности, introspection только в dev
- [ ] WebSocket: проверка Origin, передача токена через query string
- [ ] Audit Logging: append-only, не логировать PII
- [ ] API Versioning: deprecation старых версий
- [ ] Log Injection: санитизация input в логах
- [ ] 429 Too Many Requests + Retry-After header

# OWASP Top 10 для .NET — факты + примеры

---

## SQL Injection

**Факты:**
- **Причина:** Конкатенация пользовательского ввода в SQL-запрос
- **.NET защита:** EF Core LINQ автоматически параметризует. `FromSqlRaw` / `ExecuteSqlRaw` — нет.
- **Dapper vs EF Core:** Dapper тоже требует ручной параметризации (через `new { param = value }`)
- **Safe:** Параметризованные запросы, хранимые процедуры, ORM (LINQ)

**Пример — уязвимость:**

```csharp
// ❌ Злоумышленник передаёт логин: ' OR 1=1; --
var sql = $"SELECT * FROM Users WHERE Login = '{login}'";
```

**Пример — защита (ADO.NET):**

```csharp
using var cmd = new SqlCommand("SELECT * FROM Users WHERE Login = @login AND IsActive = @active", conn);
cmd.Parameters.AddWithValue("@login", login);        // string → NVarChar
cmd.Parameters.AddWithValue("@active", isActive);    // bool → Bit
```

**Факт:** `AddWithValue` может вызвать проблемы с производительностью из-за implicit type inference — на высоконагруженных системах используйте `Add` с явным `SqlDbType`.

**Факт:** Second-order SQL Injection — данные, сохранённые в БД, могут быть опасны при повторном чтении, если их не экранировать при вставке.

---

## XSS (Cross-Site Scripting)

**Типы:**

| Тип | Описание | Пример |
|---|---|---|
| **Reflected** | Вредоносный скрипт в URL/параметрах | `?q=<script>alert(1)</script>` |
| **Stored** | Скрипт сохранён в БД | Комментарий с JS |
| **DOM-based** | Клиентский JS изменяет DOM | `document.write(location.hash)` |

**Защита в .NET:**

```csharp
// ASP.NET Core Razor — автоматическое HTML-кодирование
@model string
<div>@Model</div>   <!-- <script> → <script> -->

// ❌ Опасно
@Html.Raw(Model)

// ❌ Опасно
@(new HtmlString(Model))
```

**Дополнительная защита:**

```csharp
// Content Security Policy (CSP) — заголовок
app.Use(async (context, next) =>
{
    context.Response.Headers["Content-Security-Policy"] =
        "default-src 'self'; script-src 'self' https://trusted.cdn.com; style-src 'self' 'unsafe-inline'";
    await next();
});
```

**Факт:** `HttpUtility.HtmlEncode` старый, используйте `System.Web.HttpUtility` или Razor-кодирование (автоматически).

**Факт:** XSS в Blazor Server менее вероятен, т.к. UI обновляется через SignalR, но возможен через JavaScript interop.

---

## CSRF (Cross-Site Request Forgery)

**Факты:**
- Атака: злоумышленник заставляет браузер жертвы выполнить запрос к сайту, где у жертвы есть активная сессия
- **Cookie-based auth** — наиболее уязвима (cookie отправляется автоматически)
- **JWT в заголовке** — не уязвим (JS сам добавляет заголовок)

**Защита в ASP.NET Core:**

```csharp
// Включение антифордж-токенов
builder.Services.AddAntiforgery(options =>
{
    options.HeaderName = "X-CSRF-TOKEN";       // Для AJAX
    options.Cookie.HttpOnly = true;
    options.Cookie.SameSite = SameSiteMode.Strict;
});

// На форме
<form asp-antiforgery="true"> ... </form>

// В контроллере
[ValidateAntiForgeryToken]
[HttpPost]
public IActionResult Update(UpdateModel model) { ... }

// Для SPA — глобальный фильтр
builder.Services.AddControllersWithViews(options =>
    options.Filters.Add(new AutoValidateAntiforgeryTokenAttribute()));
```

**Факт:** SameSite=Strict блокирует CSRF, но может сломать легитимные cross-site запросы. SameSite=Lax — компромисс.

**Факт:** Blazor Server не требует антифордж-токенов — SignalR использует connection-based аутентификацию.

---

## Path Traversal

```csharp
// ❌ Уязвимо: ../../../etc/passwd
var path = Path.Combine(_basePath, userInput);
File.ReadAllText(path);

// ✅ Безопасно
var fullPath = Path.GetFullPath(Path.Combine(_basePath, userInput));
if (!fullPath.StartsWith(_basePath))
    throw new SecurityException("Access denied");
```

**Факт:** `Path.GetFullPath` нормализует `..` и симлинки — обязателен после `Path.Combine`.

---

## XXE (XML External Entities)

```csharp
// ❌ Уязвимо — DTD обрабатывается
var xmlReader = XmlReader.Create(stream);

// ✅ Безопасно — отключить DTD
var settings = new XmlReaderSettings
{
    DtdProcessing = DtdProcessing.Prohibit,  // .NET Core по умолчанию Prohibit
    XmlResolver = null
};
var xmlReader = XmlReader.Create(stream, settings);

// ✅ Для XMLDocument
var doc = new XmlDocument { XmlResolver = null };
doc.Load(xmlReader);
```

**Факт:** Начиная с .NET Core 3.1, `XmlReader` по умолчанию запрещает DTD. .NET Framework — нет.

---

## Insecure Deserialization

```csharp
// ❌ Опасно — BinaryFormatter вызывает конструкторы произвольных типов
var formatter = new BinaryFormatter();
var obj = formatter.Deserialize(stream);  // Выполнит код в конструкторе типа

// ✅ Безопасно — System.Text.Json
var obj = JsonSerializer.Deserialize<T>(json);

// ✅ Безопасно — DataContractSerializer с KnownTypes
var serializer = new DataContractSerializer(typeof(T), knownTypes);
```

**Факт:** `BinaryFormatter`, `NetDataContractSerializer`, `SoapFormatter`, `LosFormatter` — **никогда не используйте** с ненадёжными данными.

**Факт:** Начиная с .NET 9, `BinaryFormatter` полностью удалён из рантайма.

**Факт:** `Newtonsoft.Json` с `TypeNameHandling.Auto` или `All` — уязвим. `System.Text.Json` по умолчанию безопасен (не десериализует произвольные типы).

---

## Open Redirect

```csharp
// ❌ Опасно
return Redirect(returnUrl);

// ✅ Безопасно — валидация URL
if (!Url.IsLocalUrl(returnUrl))
    return RedirectToAction("Index");
return Redirect(returnUrl);
```

## Security Misconfiguration

**Факты:**
- **Debug endpoints** (`/swagger`, `/healthz`, `/hangfire`) отключить в Production
- **CORS** — не `AllowAnyOrigin()` в Production
- **Stack traces** — не показывать пользователю (`app.UseExceptionHandler` / Developer Exception Page только в dev)

```csharp
if (app.Environment.IsDevelopment())
    app.UseDeveloperExceptionPage();
else
    app.UseExceptionHandler("/Error");
```

---

## Sensitive Data Exposure

**Факты:**
- Пароли в логах — **никогда**. Используйте структурное логирование (Serilog) с маскированием
- PII в URL — не передавайте (`GET /user/email?email=...`)
- Credit card, SSN — шифровать в покое и в транзите

```csharp
// Serilog — маскирование
var loggerConfig = new LoggerConfiguration()
    .Destructure.With<MaskingDestructuringPolicy>()
    .Filter.ByExcluding(e => e.Properties.ContainsKey("password"))
    .CreateLogger();
```

---

## SSRF (Server-Side Request Forgery)

**Факты:**
- Атака: заставить сервер сделать запрос к внутреннему ресурсу (metadata service, internal API, cloud IMDS)
- OWASP 2021: A10 (новый)
- Особо опасна в cloud-окружениях (AWS IMDS `169.254.169.254`, Azure IMDS `169.254.169.254`)

```csharp
// ❌ Уязвимо — пользователь контролирует URL
var url = Request.Query["url"];
var client = new HttpClient();
var response = await client.GetStringAsync(url);  // Может запросить http://169.254.169.254/metadata/identity/oauth2/token

// ✅ Защита — whitelist доменов
var allowedHosts = new[] { "api.internal.com", "cdn.myapp.com" };
var uri = new Uri(userUrl);

if (!allowedHosts.Contains(uri.Host))
    throw new SecurityException("URL not in whitelist");

// ✅ Защита — блокировка private/internal IP
var ips = await Dns.GetHostAddressesAsync(uri.Host);
if (ips.Any(ip => IsPrivateIp(ip)))
    throw new SecurityException("Private IP not allowed");

bool IsPrivateIp(IPAddress ip)
{
    var bytes = ip.GetAddressBytes();
    return ip.IsIPv6SiteLocal ||  // fe80::/10
           bytes[0] == 127 ||     // 127.0.0.0/8
           bytes[0] == 10 ||      // 10.0.0.0/8
           (bytes[0] == 172 && bytes[1] >= 16 && bytes[1] <= 31) ||  // 172.16.0.0/12
           (bytes[0] == 192 && bytes[1] == 168);  // 192.168.0.0/16
}
```

**Факт:** `HttpClient` по умолчанию не ограничивает redirects. Злоумышленник может использовать open redirect для SSRF. Используйте `HttpClientHandler.AllowAutoRedirect = false` или валидируйте redirect-цели.

**Факт:** В Kubernetes SSRF может дать доступ к `http://kubernetes.default.svc` API server. Используйте NetworkPolicy для ограничения egress.

## Broken Access Control (OWASP A01 2021)

**Факты:**
- Самая частая уязвимость по OWASP 2021 (#1)
- Типы: Insecure Direct Object Reference (IDOR), privilege escalation, forced browsing

```csharp
// ❌ IDOR — злоумышленник меняет orderId=123 на orderId=124
[HttpGet("/api/orders/{orderId}")]
public async Task<Order> GetOrder(int orderId)
{
    return await _db.Orders.FindAsync(orderId);  // Нет проверки владельца!
}

// ✅ Защита — проверка владения
[HttpGet("/api/orders/{orderId}")]
public async Task<ActionResult<Order>> GetOrder(int orderId)
{
    var userId = User.FindFirstValue(ClaimTypes.NameIdentifier);
    var order = await _db.Orders
        .FirstOrDefaultAsync(o => o.Id == orderId && o.UserId == userId);
    
    if (order == null)
        return NotFound();  // Не показываем Forbidden — раскрываем существование заказа
    return order;
}
```

**Факт:** CORS ≠ Access Control. CORS защищает браузер пользователя, не сервер. API должен проверять авторизацию независимо.

## Software and Data Integrity Failures (OWASP A08 2021)

**Факты:**
- Непроверенные CDN-скрипты (Subresource Integrity — SRI)
- Insecure deserialization (уже рассмотрена выше)
- CI/CD pipeline compromise

```html
<!-- ❌ Нет integrity check -->
<script src="https://cdn.example.com/lib.js"></script>

<!-- ✅ SRI — браузер проверит хеш -->
<script src="https://cdn.example.com/lib.js"
        integrity="sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/ux+IoMRCADJyGJdUfGwuKQ0yIyYD8c="
        crossorigin="anonymous"></script>
```

## Security Logging and Monitoring Failures (OWASP A09 2021)

```csharp
// ✅ Аудит всех важных событий
_logger.LogInformation("User {UserId} logged in from IP {Ip}", userId, ip);
_logger.LogWarning("Failed login attempt for user {UserId} from IP {Ip}", userId, ip);
_logger.LogCritical("User {UserId} attempted to access order {OrderId} of another user", userId, orderId);

// ✅ Интеграция с SIEM (структурированные логи → Splunk/ELK)
// ✅ Alerting на аномалии (много 403, много failed login, escalation attempt)
// ✅ Логирование ошибок валидации input (может указывать на атаку)
```

**Факт:** OWASP рекомендует логировать: все failed attempts, все изменения прав доступа, все операции с sensitive данными, все валидационные ошибки серверной стороны.

---

## Чек-лист

- [ ] SQL Injection: параметризованные запросы, EF LINQ
- [ ] XSS: HTML-кодирование, CSP, HttpOnly cookie
- [ ] CSRF: Antiforgery token, SameSite cookie
- [ ] SSRF: whitelist доменов, блокировка private IP, контроль redirects
- [ ] Broken Access Control: проверка владения ресурсом, IDOR protection
- [ ] Path Traversal: `Path.GetFullPath` + `StartsWith`
- [ ] XXE: `DtdProcessing.Prohibit`
- [ ] Deserialization: не использовать BinaryFormatter, `TypeNameHandling.None`
- [ ] Open Redirect: `Url.IsLocalUrl()`
- [ ] Security Misconfiguration: отключить debug в prod, убрать дефолтные пароли
- [ ] Software Integrity: SRI для CDN, lock-файлы, проверка подписей пакетов
- [ ] Logging & Monitoring: аудит sensitive операций, алерты на аномалии

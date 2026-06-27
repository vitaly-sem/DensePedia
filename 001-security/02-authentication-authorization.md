# Аутентификация и авторизация в .NET — факты + примеры

---

## JWT (JSON Web Tokens)

**Факты:**

| Параметр | Рекомендация | Почему |
|---|---|---|
| Алгоритм | **RS256** (асимметричный) для микросервисов, **HS256** (симметричный) для монолита | RS256 — приватный ключ только у issuer, публичный у всех. HS256 — один секрет на всех |
| Длина ключа HS256 | ≥ 256 бит (32 символа) | Меньше — ломается брутфорсом |
| Срок Access Token | **15-30 минут** | Чем короче, тем меньше окно для атаки |
| Срок Refresh Token | **7-30 дней** | Долгий, но с отзывом |
| Хранение Refresh Token | **HttpOnly + Secure + SameSite=Strict cookie** | Недоступен JS, не отправляется на другие сайты |
| Хранение Access Token | **Memory** (SPA) или **Authorization header** | Не localStorage — уязвим для XSS |

**Полная настройка JWT:**

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidIssuer = "https://auth.myapp.com",
            ValidateAudience = true,
            ValidAudience = "https://api.myapp.com",
            ValidateLifetime = true,                    // Проверка exp
            ClockSkew = TimeSpan.Zero,                   // По умолчанию 5 мин — убираем
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(secret)),
            
            // Дополнительно:
            RequireExpirationTime = true,
            RequireSignedTokens = true,
        };
        
        // Для SPA — передача токена через cookie
        options.Events = new JwtBearerEvents
        {
            OnMessageReceived = context =>
            {
                context.Token = context.Request.Cookies["access_token"];
                return Task.CompletedTask;
            }
        };
    });
```

**Факт:** Если не задать `ValidAudience`, JWT с другого сервиса пройдёт аутентификацию!

**Факт:** `ClockSkew` по умолчанию 5 минут — для финансовых транзакций нужно убирать.

**Факт:** JWT нельзя отозвать на стороне сервера (нет server-side state). Решения: blacklist (Redis), короткий TTL, refresh token rotation.

### Refresh Token Rotation

```csharp
// При каждом обновлении — старый refresh token становится невалидным
// Если украденный refresh token попытаются использовать — оригинальный пользователь получит ошибку
// Это называется: Refresh Token Rotation + Automatic Reuse Detection

public async Task<TokenResponse> RefreshTokenAsync(string refreshToken)
{
    var storedToken = await _db.RefreshTokens.FirstOrDefaultAsync(t => t.Token == refreshToken);
    
    if (storedToken == null || storedToken.IsRevoked)
        return Unauthorized("Token revoked or reused");
    
    // Rotation: отзываем старый
    storedToken.IsRevoked = true;
    
    // Если токен уже был отозван (reuse detection)
    if (storedToken.ReplacedByToken != null)
    {
        // Отзываем всю family токенов пользователя
        await RevokeUserFamily(storedToken.UserId);
        return Unauthorized("Possible token theft");
    }
    
    // Выдаём новую пару
    var newRefreshToken = GenerateRefreshToken();
    storedToken.ReplacedByToken = newRefreshToken.Token;
    
    return Ok(new { AccessToken = GenerateAccessToken(), RefreshToken = newRefreshToken.Token });
}
```

---

## OAuth 2.0 + OpenID Connect (OIDC)

**Факты:**

| Flow | Когда | Особенности |
|---|---|---|
| **Authorization Code + PKCE** | SPA, mobile | PKCE обязателен — защита от перехвата authorization code |
| **Authorization Code** | Server-side (ASP.NET Core MVC) | Client Secret хранится на сервере |
| **Client Credentials** | Machine-to-machine (service-to-service) | Без пользователя |
| **Implicit** | Устарел (OAuth 2.1 запретил) | Уязвим для access token interception |

**PKCE (Proof Key for Code Exchange):**

```csharp
// Генерация code_verifier и code_challenge (на клиенте)
using var rng = RandomNumberGenerator.Create();
var codeVerifierBytes = new byte[32];
rng.GetBytes(codeVerifierBytes);
var codeVerifier = Base64UrlEncode(codeVerifierBytes);

var codeChallengeBytes = SHA256.HashData(Encoding.ASCII.GetBytes(codeVerifier));
var codeChallenge = Base64UrlEncode(codeChallengeBytes);
```

**Microsoft.Identity.Web — упрощённая интеграция с Entra ID:**

```csharp
builder.Services.AddMicrosoftIdentityWebAppAuthentication(configuration, "AzureAd")
    .EnableTokenAcquisitionToCallDownstreamApi()
    .AddInMemoryTokenCaches();
```

**Факт:** OAuth 2.1 (RFC) объединяет OAuth 2.0 + PKCE, запрещает Implicit flow, требует Proof-of-Possession.

---

## ASP.NET Core Identity

**Факты:**
- Использует **PBKDF2** с 600 000 итераций (с .NET 8) — достаточно для большинства приложений
- **Lockout** — блокировка после N неудачных попыток (защита от брутфорса)
- **Two-Factor Authentication** — TOTP (Authenticator app), SMS, Email
- **Security Stamp** — инвалидация всех сессий при смене пароля

```csharp
builder.Services.AddIdentity<ApplicationUser, IdentityRole>(options =>
{
    // Настройки пароля
    options.Password.RequiredLength = 12;
    options.Password.RequireDigit = true;
    options.Password.RequireNonAlphanumeric = true;
    options.Password.RequiredUniqueChars = 6;
    
    // Lockout
    options.Lockout.MaxFailedAccessAttempts = 5;
    options.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(15);
    
    // Security stamp
    options.SecurityStampValidationInterval = TimeSpan.FromMinutes(30);
})
.AddEntityFrameworkStores<AppDbContext>()
.AddDefaultTokenProviders();
```

---

## Авторизация — Claims, Policies, Roles

**Факты:**
- **Roles** — грубая авторизация (Admin, User, Manager)
- **Claims** — гибкая авторизация (permission-based: `reports:read`, `reports:write`)
- **Policies** — набор требований (claim, role, custom handler)
- **Resource-based authorization** — доступ зависит от объекта (владелец документа)

```csharp
// Claims-based policy
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("CanViewReports", policy =>
        policy.RequireClaim("permission", "reports:read"));
    
    options.AddPolicy("CanDeleteReports", policy =>
        policy.RequireAssertion(context =>
            context.User.HasClaim("permission", "reports:delete") &&
            context.User.HasClaim(c => c.Type == "department" && c.Value == "Admin")));
    
    options.AddPolicy("ElevatedAccess", policy =>
        policy.RequireRole("Admin")
              .RequireClaim("clearance", "level3")
              .RequireClaim("mfa", "true"));
});

// Resource-based authorization
public class DocumentAuthorizationHandler : AuthorizationHandler<DocumentEditRequirement, Document>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context, 
        DocumentEditRequirement requirement, 
        Document resource)
    {
        // Проверка: владелец документа или администратор
        if (context.User.FindFirstValue(ClaimTypes.NameIdentifier) == resource.OwnerId ||
            context.User.IsInRole("Admin"))
        {
            context.Succeed(requirement);
        }
        return Task.CompletedTask;
    }
}

// Использование в контроллере
[Authorize("CanViewReports")]
public IActionResult GetReport(int id) { ... }

// Resource-based в сервисе
public async Task<Document> GetDocument(int id)
{
    var doc = await _db.Documents.FindAsync(id);
    var authResult = await _authService.AuthorizeAsync(User, doc, "DocumentEdit");
    if (!authResult.Succeeded)
        throw new ForbiddenAccessException();
    return doc;
}
```

**Факт:** `[Authorize(Roles = "Admin,Manager")]` — OR (хотя бы одна роль). Для AND используйте политики.

**Факт:** Policy-based авторизация тестируется без контроллеров — unit test для `AuthorizationHandler`.

---

## WebAuthn / Passkeys (FIDO2)

**Факты:**
- WebAuthn — W3C стандарт для passwordless-аутентификации
- Passkeys — Apple/Google/Microsoft реализация FIDO2 (синхронизируются через iCloud/Google Password Manager)
- Устойчивы к фишингу (origin-bound), не требуют пароля
- Поддержка в .NET через `Fido2` (NuGet: `Fido2.AspNet`)

```csharp
// Регистрация passkey
public class WebAuthnController : Controller
{
    private readonly IFido2 _fido2;
    
    [HttpPost("webauthn/register/options")]
    public async Task<CredentialCreateOptions> MakeCredentialOptions([FromBody] string username)
    {
        var user = new Fido2User { Id = Encoding.UTF8.GetBytes(username), Name = username, DisplayName = username };
        var options = _fido2.RequestNewCredential(user, new List<PublicKeyCredentialDescriptor>());
        HttpContext.Session.SetString("fido2.attestation", options.ToJson());
        return options;
    }
    
    [HttpPost("webauthn/register")]
    public async Task<IActionResult> MakeCredential([FromBody] AuthenticatorAttestationRawResponse attestation)
    {
        var options = CredentialCreateOptions.FromJson(HttpContext.Session.GetString("fido2.attestation"));
        var result = await _fido2.MakeNewCredentialAsync(attestation, options, (args, _) => Task.FromResult(true));
        // Сохраняем result.Result.CredentialId и result.Result.PublicKey в БД
        return Ok(new { result.Result.CredentialId });
    }
    
    [HttpPost("webauthn/login")]
    public async Task<IActionResult> Login([FromBody] AuthenticatorAssertionRawResponse assertion)
    {
        var storedCreds = await _db.UserCredentials.Where(c => c.UserId == userId).ToListAsync();
        var result = await _fido2.MakeAssertionAsync(assertion, options,
            storedCreds.First().PublicKey, storedCreds.Select(c => c.Counter));
        
        // result.VerifySuccess — true если passkey подлинный
        // Counter используется для предотвращения replay
        return Ok(new { Token = GenerateJwt(user) });
    }
}
```

**Факт:** Passkeys устраняют: phishing, credential stuffing, password reuse, weak passwords. Google после внедрения passkeys сообщила о снижении взломов на 50%+.

## API Key Authentication

**Факты:**
- Для machine-to-machine коммуникации, где OAuth избыточен
- API Key ≠ JWT — это просто секретный токен (без claims, без истечения по умолчанию)
- Нужна ротация и возможность отзыва

```csharp
// API Key Authentication Handler
public class ApiKeyAuthenticationHandler : AuthenticationHandler<ApiKeyAuthenticationOptions>
{
    private readonly IApiKeyValidator _validator;
    
    protected override async Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        if (!Request.Headers.TryGetValue("X-API-Key", out var apiKeyHeader))
            return AuthenticateResult.NoResult();
        
        var apiKey = apiKeyHeader.First();
        var validationResult = await _validator.ValidateAsync(apiKey);
        
        if (!validationResult.IsValid)
            return AuthenticateResult.Fail("Invalid API Key");
        
        var claims = new[]
        {
            new Claim(ClaimTypes.NameIdentifier, validationResult.ClientId),
            new Claim("scope", validationResult.Scope ?? "read"),
            new Claim("apikey_id", validationResult.KeyId)
        };
        
        var identity = new ClaimsIdentity(claims, Scheme.Name);
        var principal = new ClaimsPrincipal(identity);
        var ticket = new AuthenticationTicket(principal, Scheme.Name);
        
        return AuthenticateResult.Success(ticket);
    }
}

// Регистрация
builder.Services.AddAuthentication()
    .AddScheme<ApiKeyAuthenticationOptions, ApiKeyAuthenticationHandler>(
        "ApiKey", options => { });
```

**Факт:** API Keys должны храниться в БД хешированными (SHA-256), не в открытом виде. При проверке — хешируем входящий ключ и сравниваем с хешем в БД.

## Certificate-Based Authentication (mTLS Auth)

```csharp
// Kestrel — требование клиентского сертификата
builder.WebHost.ConfigureKestrel(options =>
{
    options.ConfigureHttpsDefaults(httpsOptions =>
    {
        httpsOptions.ClientCertificateMode = ClientCertificateMode.RequireCertificate;
        httpsOptions.CheckCertificateRevocation = true;
    });
});

// Certificate Authentication
builder.Services.AddAuthentication(CertificateAuthenticationDefaults.AuthenticationScheme)
    .AddCertificate(options =>
    {
        options.AllowedCertificateTypes = CertificateTypes.All;
        options.Events = new CertificateAuthenticationEvents
        {
            OnCertificateValidated = context =>
            {
                // Дополнительная валидация: проверка Thumbprint по whitelist
                var allowedThumbprints = context.HttpContext.RequestServices
                    .GetRequiredService<IAllowedCertificatesService>()
                    .GetThumbprints();
                
                if (!allowedThumbprints.Contains(context.ClientCertificate.Thumbprint))
                    context.Fail("Certificate not in whitelist");
                
                return Task.CompletedTask;
            }
        };
    });
```

**Факт:** В Kubernetes с Istio/Linkerd mTLS работает через sidecar-прокси — приложение получает заголовки `X-Forwarded-Client-Cert`, которые нужно проверять.

---

## Чек-лист

- [ ] JWT: RS256 для микросервисов, ClockSkew = 0, ValidateAudience
- [ ] Refresh Token Rotation + Reuse Detection
- [ ] OAuth 2.0: Authorization Code + PKCE для SPA
- [ ] WebAuthn/Passkeys для passwordless аутентификации
- [ ] API Keys: хеширование при хранении, ротация
- [ ] Certificate Auth: проверка revocation, whitelist thumbprints
- [ ] Identity: lockout, security stamp, 2FA
- [ ] Policies vs Roles: когда что использовать
- [ ] Resource-based authorization

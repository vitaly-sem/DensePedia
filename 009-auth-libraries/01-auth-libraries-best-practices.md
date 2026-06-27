# Authorization Libraries & Best Practices

---

## Identity Providers — сравнение

| Провайдер | Тип | Open Source | Protocol | .NET SDK | Best for |
|---|---|---|---|---|---|
| **Duende IdentityServer** | Апач 2.0 / Commercial | ✅ (до v6) | OAuth 2.0 / OIDC | `Duende.IdentityServer` | Enterprise, кастомизация |
| **Microsoft Entra ID** | Cloud (SaaS) | ❌ | OAuth 2.0 / OIDC | `Microsoft.Identity.Web` | Azure ecosystem |
| **Auth0** | Cloud (SaaS) | ❌ | OAuth 2.0 / OIDC | `Auth0.AuthenticationApi` | Startups, SPA |
| **Keycloak** | Self-hosted | ✅ | OAuth 2.0 / OIDC | `Keycloak.Net` | Open-source, Java/.NET |
| **AWS Cognito** | Cloud (SaaS) | ❌ | OAuth 2.0 / OIDC | `Amazon.Extensions.CognitoAuthentication` | AWS ecosystem |
| **Firebase Auth** | Cloud (SaaS) | ❌ | OAuth 2.0 / OIDC | `FirebaseAdmin` | Mobile apps |

---

## Duende IdentityServer — настройка

**Факты:**
- Преемник IdentityServer4 (был Open Source, теперь Duende Software)
- Стоимость: бесплатно для доходов < $1M
- Полный OAuth 2.0 + OIDC, протоколы: authorization code, client credentials, device flow

```csharp
// Настройка Duende IdentityServer
public class Config
{
    public static IEnumerable<Client> Clients =>
        new[]
        {
            new Client
            {
                ClientId = "spa-client",
                AllowedGrantTypes = GrantTypes.Code,  // Authorization Code + PKCE
                RequirePkce = true,
                RequireClientSecret = false,  // Public client
                
                RedirectUris = { "https://localhost:4200/callback" },
                PostLogoutRedirectUris = { "https://localhost:4200" },
                AllowedCorsOrigins = { "https://localhost:4200" },
                
                AllowOfflineAccess = true,  // Refresh Token
                AllowedScopes = { "openid", "profile", "email", "api1" },
                
                AccessTokenLifetime = 900,     // 15 min
                IdentityTokenLifetime = 300,   // 5 min
            }
        };
    
    public static IEnumerable<ApiScope> ApiScopes =>
        new[] { new ApiScope("api1", "My API") };
    
    public static IEnumerable<IdentityResource> IdentityResources =>
        new IdentityResource[]
        {
            new IdentityResources.OpenId(),
            new IdentityResources.Profile(),
            new IdentityResources.Email(),
        };
}

// Program.cs
builder.Services.AddIdentityServer()
    .AddInMemoryClients(Config.Clients)
    .AddInMemoryApiScopes(Config.ApiScopes)
    .AddInMemoryIdentityResources(Config.IdentityResources)
    .AddTestUsers(TestUsers.Users);
```

---

## ABAC (Attribute-Based Access Control)

**Факты:**
- **RBAC** — роли (Admin, User, Manager)
- **ABAC** — атрибуты (user.department == "HR" && resource.sensitivity == "Low")
- ABAC гибче, но сложнее в управлении

```csharp
// ABAC — атрибуты + политики
public class DocumentAuthorizationHandler : AuthorizationHandler<DocumentRequirement, Document>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        DocumentRequirement requirement,
        Document resource)
    {
        // Атрибуты пользователя
        var userDept = context.User.FindFirstValue("department");
        var userClearance = context.User.FindFirstValue("clearance");
        var isWeekend = DateTime.UtcNow.DayOfWeek is DayOfWeek.Saturday or DayOfWeek.Sunday;
        
        // Атрибуты ресурса
        var docClassification = resource.Classification;
        var docDepartment = resource.Department;
        
        // ABAC правила
        bool canAccess = false;
        
        // Правило 1: сотрудник отдела может читать
        if (userDept == docDepartment && requirement.Action == "read")
            canAccess = true;
        
        // Правило 2: HR может читать всё
        if (userDept == "HR" && requirement.Action == "read")
            canAccess = true;
        
        // Правило 3: никто не может писать в выходные
        if (requirement.Action == "write" && isWeekend)
            canAccess = false;
        
        if (canAccess)
            context.Succeed(requirement);
        
        return Task.CompletedTask;
    }
}
```

---

## OPA (Open Policy Agent)

**Факт:** OPA — универсальный policy engine (не только для auth). Использует язык Rego.

```rego
# policy.rego — OPA policy
package authz

default allow = false

# Allow if user is admin
allow {
    input.user.roles[_] == "admin"
}

# Allow if user owns the resource
allow {
    input.user.id == input.resource.owner_id
}

# Allow if user is in same department AND resource is not classified
allow {
    input.user.department == input.resource.department
    input.resource.classification != "confidential"
}
```

```csharp
// .NET client для OPA
public class OpaAuthorizer
{
    private readonly HttpClient _http;
    
    public async Task<bool> AuthorizeAsync(string user, string action, string resource)
    {
        var request = new
        {
            input = new
            {
                user = new { id = user, roles = await GetRolesAsync(user) },
                action,
                resource = new { id = resource, department = GetDepartment(resource) }
            }
        };
        
        var response = await _http.PostAsJsonAsync("http://opa:8181/v1/data/authz", request);
        var result = await response.Content.ReadFromJsonAsync<OpaResult>();
        
        return result?.Result ?? false;
    }
}
```

---

## Policy-based Authorization в ASP.NET Core

```csharp
// 1. Объявление политик
builder.Services.AddAuthorization(options =>
{
    // RBAC
    options.AddPolicy("RequireAdmin", policy => policy.RequireRole("Admin"));
    options.AddPolicy("RequireManager", policy => policy.RequireRole("Manager"));
    
    // Claims-based
    options.AddPolicy("CanExportData", policy =>
        policy.RequireClaim("permission", "data:export"));
    options.AddPolicy("CanDeleteUsers", policy =>
        policy.RequireAssertion(context =>
            context.User.IsInRole("Admin") &&
            context.User.HasClaim("mfa", "true")));
    
    // Multi-factor
    options.AddPolicy("RequireMfa", policy =>
        policy.RequireClaim("amr", "mfa"));
});

// 2. Кастомный AuthorizationHandler
public class MinimumAgeHandler : AuthorizationHandler<MinimumAgeRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        MinimumAgeRequirement requirement)
    {
        var dobClaim = context.User.FindFirst(c => c.Type == "birthdate");
        
        if (dobClaim != null && DateTime.TryParse(dobClaim.Value, out var dob))
        {
            var age = DateTime.Today.Year - dob.Year;
            if (age >= requirement.MinimumAge)
                context.Succeed(requirement);
        }
        
        return Task.CompletedTask;
    }
}

// 3. Resource-based (возвращает 403/404)
public class DocumentController : Controller
{
    [HttpGet("documents/{id}")]
    public async Task<IActionResult> Get(int id)
    {
        var doc = await _db.Documents.FindAsync(id);
        if (doc == null) return NotFound();
        
        var auth = await _authService.AuthorizeAsync(User, doc, "DocumentRead");
        if (!auth.Succeeded) return Forbid();
        
        return Ok(doc);
    }
}
```

---

## Permission-based Authorization

**Факт:** Permission-based авторизация — роли + permissions (flat list):

| Роль | Permissions |
|---|---|
| Admin | `*:*` |
| Manager | `reports:read`, `reports:write`, `users:read` |
| User | `reports:read` |

```csharp
// Permission-based через клаймы
public static class Permissions
{
    // Resources
    public const string Reports = "reports";
    public const string Users = "users";
    public const string Settings = "settings";
    
    // Actions
    public const string Read = "read";
    public const string Write = "write";
    public const string Delete = "delete";
    
    // Combined
    public static string All(string resource) => $"{resource}:*";
    public static string Action(string resource, string action) => $"{resource}:{action}";
}

// Политики
options.AddPolicy("Reports.Read", policy =>
    policy.RequireClaim("permission", Permissions.Action(Permissions.Reports, Permissions.Read)));
options.AddPolicy("Reports.Write", policy =>
    policy.RequireClaim("permission", Permissions.Action(Permissions.Reports, Permissions.Write)));
```

---

## Best Practices — чек-лист

- [ ] **Использовать OAuth 2.0 + OIDC**, не самописную аутентификацию
- [ ] **PKCE обязателен** для SPA и mobile
- [ ] **Short-lived access tokens** (15 min), refresh token rotation
- [ ] **JWT vs Reference tokens**: JWT для микросервисов (нет lookup), Reference для revoke
- [ ] **ABAC** для сложной логики (атрибуты), **RBAC** для простой
- [ ] **OPA** для унифицированной политики в микросервисах
- [ ] **Resource-based authorization** для CRUD (не [Authorize] на контроллере)
- [ ] **Least privilege**: давать минимум permissions
- [ ] **Audit** всех authorization decisions
- [ ] **Rate limit** на login/register endpoints
- [ ] **MFA** для sensitive operations

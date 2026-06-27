# Конфигурация и транспортная безопасность — факты + примеры

---

## Безопасность конфигурации

### Факты

**Что НЕЛЬЗЯ хранить в appsettings.json:**
- Connection strings с паролями
- API keys, JWT secrets
- Сертификаты (private keys)
- Любые credentials

**Где хранить:**

| Окружение | Решение |
|---|---|
| **Development** | `dotnet user-secrets` / `appsettings.Development.json` |
| **Staging/Production** | Azure Key Vault / AWS Secrets Manager / HashiCorp Vault |
| **Kubernetes** | Kubernetes Secrets + External Secrets Operator |

```csharp
// User Secrets (dev)
// dotnet user-secrets init
// dotnet user-secrets set "Jwt:Secret" "super-secret-key-123"

// Azure Key Vault (prod)
builder.Configuration.AddAzureKeyVault(
    new Uri("https://myvault.vault.azure.net/"),
    new DefaultAzureCredential());

// Использование
var secret = configuration["Jwt:Secret"]; // Автоматически из Key Vault

// Опционально: prefix-based фильтрация
builder.Configuration.AddAzureKeyVault(
    new Uri("https://myvault.vault.azure.net/"),
    new DefaultAzureCredential(),
    new AzureKeyVaultConfigurationOptions
    {
        Manager = new PrefixKeyVaultSecretManager("MyApp")
    });
```

### Connection Strings — безопасность

```csharp
// ❌ Опасно — пароль в открытом виде
"Server=db;Database=app;User Id=sa;Password=Pass123"

// ✅ Managed Identity (Azure SQL) — без пароля
"Server=myserver.database.windows.net;Database=app;Authentication=ActiveDirectoryManagedIdentity"

// ✅ Integrated Security (Windows)
"Server=.;Database=app;Integrated Security=true;Trusted_Connection=true;"

// ✅ Encrypted connection
"Server=.;Database=app;Integrated Security=true;Encrypt=true;TrustServerCertificate=false;"
```

**Факт:** `TrustServerCertificate=true` отключает проверку TLS сертификата SQL Server — только для dev.

### Docker Compose — секреты

```yaml
# docker-compose.yml
services:
  api:
    secrets:
      - db_password
      - jwt_secret

secrets:
  db_password:
    file: ./secrets/db_password.txt
  jwt_secret:
    environment: JWT_SECRET
```

```csharp
// В приложении
var dbPassword = File.ReadAllText("/run/secrets/db_password");
```

---

## Dependency Confusion / Supply Chain

**Факты:**
- Атака: злоумышленник публикует пакет с тем же именем, что и internal пакет, в public feed
- NuGet по умолчанию проверяет все источники — если public feed раньше private, подмена возможна

```xml
<!-- nuget.config — защита -->
<configuration>
  <packageSources>
    <clear />   <!-- Очистить все источки по умолчанию -->
    <add key="MyPrivateFeed" value="https://pkgs.dev.azure.com/myorg/_packaging/myfeed/nuget/v3/index.json" />
  </packageSources>
  
  <packageSourceMapping>
    <!-- Явное указание: пакеты с префиксом MyCompany.* только из private feed -->
    <packageSource key="MyPrivateFeed">
      <package pattern="MyCompany.*" />
      <package pattern="MyApp.*" />
    </packageSource>
    
    <!-- Остальные пакеты из nuget.org -->
    <packageSource key="nuget.org">
      <package pattern="*" />
    </packageSource>
  </packageSourceMapping>
</configuration>
```

**Дополнительная защита:**
- `dotnet nuget verify` — проверка цифровой подписи
- Пиннинг версий (`Version="*"` — опасно)
- `lock файлы` — `dotnet restore --locked-mode` (воспроизводимые сборки)

```xml
<!-- .csproj — фиксация версий -->
<PackageReference Include="Newtonsoft.Json" Version="[13.0.3,)" />  <!-- >= 13.0.3 -->
<PackageReference Include="Serilog" Version="3.1.1" PrivateAssets="all" />  <!-- exact -->
```

**Факт:** `dotnet restore --locked-mode` использует `packages.lock.json` — если lock-файл не совпадает с resolve'ом, сборка падает.

---

## Транспортная безопасность

### HTTPS

```csharp
// Program.cs — полная настройка
var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseHsts();  // Strict-Transport-Security header
}

app.UseHttpsRedirection();  // 302 → https

// Кастомный порт для HTTPS
builder.WebHost.UseUrls("https://localhost:5001;http://localhost:5000");
```

**HSTS Preload:**
```csharp
app.UseHsts(options =>
{
    options.MaxAge = TimeSpan.FromDays(365);  // 1 год
    options.IncludeSubDomains = true;
    options.Preload = true;  // Google preload list
});
```

**Факт:** HSTS preload — отправка домена в Google Chrome preload list. После включения, HTTPS обязателен навсегда (или до следующего обновления Chrome).

### gRPC — TLS

```csharp
// Server-side
builder.WebHost.ConfigureKestrel(options =>
{
    options.ConfigureHttpsDefaults(httpsOptions =>
    {
        // mTLS — взаимная аутентификация
        httpsOptions.ClientCertificateMode = ClientCertificateMode.RequireCertificate;
        httpsOptions.CheckCertificateRevocation = true;
    });
});

// Client-side
var handler = new HttpClientHandler
{
    ClientCertificates = { clientCertificate },
    ServerCertificateCustomValidationCallback = (sender, cert, chain, errors) =>
    {
        // Кастомная валидация
        return errors == SslPolicyErrors.None || IsTrusted(cert);
    }
};
var channel = GrpcChannel.ForAddress("https://api.internal:5001",
    new GrpcChannelOptions { HttpHandler = handler });
```

### Service Mesh (mTLS без изменения кода)

**Факт:** В Kubernetes с Istio/Linkerd mTLS на стороне proxy (Envoy/Linkerd-proxy) — не требует настройки в .NET:

```yaml
# Istio PeerAuthentication — автоматический mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: prod
spec:
  mtls:
    mode: STRICT  # REQUIRE mTLS для всех сервисов
```

---

## Kubernetes Security (контекст для Senior)

```yaml
# Pod Security Context
apiVersion: v1
kind: Pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: api
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
      readOnlyRootFilesystem: true
```

```csharp
// В .NET приложении — проверка non-root
if (!OperatingSystem.IsLinux() || Environment.UserName == "root")
    logger.LogWarning("Running as root in container!");
```

---

## Secure HTTP Headers

```csharp
// Middleware для добавления security headers
app.Use(async (context, next) =>
{
    var headers = context.Response.Headers;
    
    // Prevent MIME type sniffing
    headers["X-Content-Type-Options"] = "nosniff";
    
    // Prevent clickjacking
    headers["X-Frame-Options"] = "DENY";  // или SAMEORIGIN для виджетов
    
    // Enable XSS filter in old browsers
    headers["X-XSS-Protection"] = "1; mode=block";
    
    // Referrer policy
    headers["Referrer-Policy"] = "strict-origin-when-cross-origin";
    
    // Permissions Policy (бывший Feature-Policy)
    headers["Permissions-Policy"] = "camera=(), microphone=(), geolocation=()";
    
    await next();
});
```

**Факт:** `X-Frame-Options` устаревает в пользу CSP `frame-ancestors 'none'`. Используйте оба для обратной совместимости.

## TLS 1.2/1.3 Enforcement in Kestrel

```csharp
builder.WebHost.ConfigureKestrel(options =>
{
    options.ConfigureHttpsDefaults(httpsOptions =>
    {
        // Только TLS 1.2 и 1.3 (отключаем TLS 1.0/1.1)
        httpsOptions.SslProtocols = SslProtocols.Tls12 | SslProtocols.Tls13;
        
        // Только сильные cipher suites
        httpsOptions.OnAuthenticate = (context, authOpts) =>
        {
            // Для Windows — через Schannel registry
            // Для Linux — OpenSSL cipher string
        };
    });
});
```

**Факт:** TLS 1.0 и 1.1 официально deprecated (RFC 8996, март 2021). PCI DSS требует TLS 1.2+ с 2018.

## Certificate Pinning

```csharp
// Certificate pinning — проверка, что серверный сертификат соответствует ожидаемому
// Защита от MITM через скомпрометированный CA

var handler = new HttpClientHandler
{
    ServerCertificateCustomValidationCallback = (sender, cert, chain, errors) =>
    {
        // Базовая проверка
        if (errors == SslPolicyErrors.None) return true;
        
        // Pinning по публичному ключу (SPKI fingerprint)
        var expectedSpkiPin = "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=";
        var actualSpki = Convert.ToBase64String(
            SHA256.HashData(cert!.PublicKey.EncodedKeyValue.RawData));
        var actualPin = $"sha256/{actualSpki}";
        
        return actualPin == expectedSpkiPin;
    }
};
```

**Факт:** Certificate pinning рискован — при смене сертификата приложение ломается. Альтернативы: Certificate Transparency monitoring, DNS CAA records, DANE/TLSA.

## Environment Variable Security

```csharp
// ❌ Опасно — переменные окружения читаются дочерними процессами
// Docker: env vars видны в docker inspect, docker-compose, /proc/1/environ

// ✅ Лучше: Docker secrets (монтируются как файлы)
var dbPassword = File.ReadAllText("/run/secrets/db_password").Trim();

// ✅ В K8s: заворачиваем sensitive config в secrets
builder.Configuration.AddAzureKeyVault(/* ... */);

// ❌ Никогда не передавайте secrets через CMD/ENTRYPOINT
// docker run -e DB_PASSWORD=secret123  → видно в docker inspect
// docker run -e DB_PASSWORD_FILE=/run/secrets/db_password ✅
```

---

## Чек-лист

- [ ] Secrets: User Secrets (dev), Key Vault (prod), Docker/K8s Secrets (не env vars)
- [ ] nuget.config: package source mapping, locked mode
- [ ] Пиннинг версий пакетов, lock-файлы
- [ ] HTTPS: HSTS, HSTS preload, TLS 1.2 minimum
- [ ] Secure Headers: X-Content-Type-Options, X-Frame-Options, CSP, Permissions-Policy
- [ ] mTLS для внутренних сервисов (gRPC или Service Mesh)
- [ ] Certificate Pinning: SPKI pinning или Certificate Transparency
- [ ] Pod Security: non-root, readOnlyRootFilesystem, drop ALL capabilities
- [ ] `TrustServerCertificate=false` в production

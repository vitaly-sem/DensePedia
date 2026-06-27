# Шифрование данных в покое и транзите

---

## Data at Rest — шифрование на уровне хранения

**Факты:**
- **Прозрачное шифрование (TDE)** — SQL Server шифрует данные на диске (файлы БД и логов). Прозрачно для приложения.
- **Always Encrypted** — SQL Server **не видит** данные (шифрование на клиенте). Допускает сравнения на равенство (deterministic encryption).
- **EBS / Disk Encryption** — BitLocker (Windows), LUKS (Linux), Azure Disk Encryption

### SQL Server — TDE (Transparent Data Encryption)

```sql
-- TDE — шифрование на уровне БД, приложение не замечает
-- Во время выполнения — данные расшифрованы в памяти
-- На диске — зашифрованы (даже backup'ы)

-- 1. Master key
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'StrongPassword123!';

-- 2. Certificate
CREATE CERTIFICATE TDECert WITH SUBJECT = 'TDE Certificate';

-- 3. Database Encryption Key
CREATE DATABASE ENCRYPTION KEY
    WITH ALGORITHM = AES_256
    ENCRYPTION BY SERVER CERTIFICATE TDECert;

-- 4. Включить TDE
ALTER DATABASE MyDatabase SET ENCRYPTION ON;
```

**Факт:** TDE не защищает от администраторов БД (у них есть доступ к сертификату).

### SQL Server — Always Encrypted (client-side encryption)

```csharp
// Always Encrypted — шифрование на клиенте
// SQL Server хранит только зашифрованные данные
// Ключи хранятся в Windows Certificate Store или Azure Key Vault

// Connection string
"Server=myserver;Database=mydb;Column Encryption Setting=Enabled;"

// Настройка в C#
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString, sqlOptions =>
    {
        sqlOptions.EnableRetryOnFailure(3);
        sqlOptions.ColumnEncryptionSetting = ColumnEncryptionSetting.Enabled;
    }));

// Модель с зашифрованной колонкой
public class Patient
{
    public int Id { get; set; }
    
    // Always Encrypted (deterministic — поддержка равенства)
    [Encrypted(ColumnEncryptionKey = "CEK_Auto1", EncryptionType = EncryptionType.Deterministic)]
    public string SSN { get; set; }  // Поиск по SSN работает!
    
    // Always Encrypted (randomized — более безопасно)
    [Encrypted(ColumnEncryptionKey = "CEK_Auto1", EncryptionType = EncryptionType.Randomized)]
    public string Diagnosis { get; set; }  // Нельзя искать по значению
    
    public string Name { get; set; }  // Обычный текст
}

// Работа с данными
var patient = await context.Patients
    .FirstOrDefaultAsync(p => p.SSN == "123-45-6789");  // Работает!
// SQL Server получает зашифрованный запрос (SSN шифруется на клиенте)
```

| Encryption Type | Возможности | Производительность |
|---|---|---|
| **Deterministic** | =, IN, JOIN, GROUP BY | Высокая (одинаковый plaintext → одинаковый ciphertext) |
| **Randomized** | Только запись и чтение | Максимальная безопасность |

**Факт:** Always Encrypted не поддерживает: LIKE, Range (`>`, `<`), ORDER BY, агрегации (SUM, AVG) на зашифрованных колонках.

---

## File-level encryption в .NET

### Azure Storage — Server-Side Encryption (SSE)

```csharp
// Azure Blob Storage — SSE включена по умолчанию (AES-256)
// Дополнительно: Customer-Provided Keys (CPK) или Customer-Managed Keys (CMK)

var blobClient = new BlobClient(connectionString, containerName, blobName);

// Customer-Provided Key (клиент передаёт ключ при каждом запросе)
var cpkOptions = new BlobRequestConditions
{
    EncryptionKey = Convert.FromBase64String("base64-encoded-256-bit-key"),
    EncryptionKeySha256 = Convert.FromBase64String("..."),
    EncryptionAlgorithm = EncryptionAlgorithm.Aes256
};

await blobClient.UploadAsync(stream, new BlobUploadOptions
{
    Conditions = cpkOptions
});
```

### File-level encryption в самом приложении

```csharp
// Шифрование файлов перед сохранением на диск
public class SecureFileStorage
{
    private readonly byte[] _key;
    
    public SecureFileStorage(byte[] key) => _key = key;
    
    public async Task EncryptAndSaveAsync(string path, Stream data)
    {
        using var fileStream = File.Create(path);
        using var aes = new AesGcm(_key, 16);
        
        var nonce = RandomNumberGenerator.GetBytes(AesGcm.NonceByteSizes.MinSize);
        await fileStream.WriteAsync(nonce.AsMemory(0, nonce.Length));
        
        using var memoryStream = new MemoryStream();
        await data.CopyToAsync(memoryStream);
        var plaintext = memoryStream.ToArray();
        
        var ciphertext = new byte[plaintext.Length];
        var tag = new byte[AesGcm.TagByteSizes.MaxSize];
        
        aes.Encrypt(nonce, plaintext, ciphertext, tag);
        
        await fileStream.WriteAsync(ciphertext.AsMemory(0, ciphertext.Length));
        await fileStream.WriteAsync(tag.AsMemory(0, tag.Length));
    }
}
```

---

## Data in Transit — шифрование канала

### TLS — обязательный минимум

```csharp
// Принудительный HTTPS
app.UseHttpsRedirection();
app.UseHsts(options =>
{
    options.MaxAge = TimeSpan.FromDays(365);
    options.IncludeSubDomains = true;
    options.Preload = true;
});

// TLS версии — отключить устаревшие
ServicePointManager.SecurityProtocol = SecurityProtocolType.Tls12 | SecurityProtocolType.Tls13;

// HttpClient — минимальные требования
var handler = new HttpClientHandler
{
    SslProtocols = SslProtocols.Tls12 | SslProtocols.Tls13,
    ServerCertificateCustomValidationCallback = (sender, cert, chain, errors) =>
    {
        if (errors == SslPolicyErrors.None) return true;
        
        // Production — только None
        // Dev — может быть RemoteCertificateChainErrors для самоподписанных
        return false;  // Безопасное поведение
    }
};
```

### mTLS — взаимная аутентификация

```csharp
// gRPC с mTLS
builder.WebHost.ConfigureKestrel(options =>
{
    options.ConfigureHttpsDefaults(httpsOptions =>
    {
        httpsOptions.ClientCertificateMode = ClientCertificateMode.RequireCertificate;
        httpsOptions.CheckCertificateRevocation = true;
    });
});

// Service-to-service с сертификатами
var handler = new HttpClientHandler();
handler.ClientCertificates.Add(clientCertificate);

var client = new HttpClient(handler)
{
    BaseAddress = new Uri("https://internal-api.service:443")
};

// Mutual TLS в Kubernetes (Istio) — автоматический
// Без изменения кода приложения
```

### Database connections — шифрование

```csharp
// SQL Server — Encrypt=True
"Server=myserver.database.windows.net;Database=mydb;Authentication=ActiveDirectoryManagedIdentity;Encrypt=True;TrustServerCertificate=False;"

// PostgreSQL — SSL mode
"Host=myhost;Database=mydb;SSL Mode=Require;Trust Server Certificate=false;"

// MongoDB — TLS
var settings = MongoClientSettings.FromConnectionString(connectionString);
settings.UseTls = true;
settings.SslSettings = new SslSettings
{
    EnabledSslProtocols = SslProtocols.Tls12,
    CheckCertificateRevocation = true
};
```

---

## Key Management — управление ключами

**Факты:**
- **HSM (Hardware Security Module)** — аппаратное хранение ключей
- **Azure Key Vault / AWS KMS** — managed HSM в облаке
- **Windows Certificate Store** — для сертификатов

```csharp
// Azure Key Vault — интеграция
builder.Configuration.AddAzureKeyVault(
    new Uri("https://myvault.vault.azure.net/"),
    new DefaultAzureCredential());

// Получение ключа для шифрования
public class EncryptionService
{
    private readonly SecretClient _secretClient;
    
    public EncryptionService(IConfiguration configuration)
    {
        var keyVaultUrl = configuration["KeyVault:Url"];
        _secretClient = new SecretClient(new Uri(keyVaultUrl), new DefaultAzureCredential());
    }
    
    public async Task<byte[]> GetEncryptionKeyAsync(string keyName)
    {
        var response = await _secretClient.GetSecretAsync(keyName);
        return Convert.FromBase64String(response.Value.Value);
    }
}
```

---

## Data Masking (Dynamic Data Masking)

```sql
-- SQL Server — Dynamic Data Masking (без изменения приложения)
ALTER TABLE Users
ALTER COLUMN Email ADD MASKED WITH (FUNCTION = 'email()');
ALTER TABLE Users
ALTER COLUMN SSN ADD MASKED WITH (FUNCTION = 'partial(0,"***-**-",4)');
ALTER TABLE Users
ALTER COLUMN Phone ADD MASKED WITH (FUNCTION = 'partial(3,"-***-***-",2)');

-- Результат для обычного пользователя:
-- Email: j***@contoso.com
-- SSN: ***-**-6789
-- Phone: 555-***-***-09
```

**Факт:** Dynamic Data Masking не защищает от пользователей с `UNMASK` permission (DBA).

---

## Anonymization и Pseudonymization

```csharp
// Pseudonymization — замена PII на псевдоним (обратимо для авторизованных)
public static class DataAnonymizer
{
    private static readonly byte[] _hmacKey = new byte[32];  // Хранить в Key Vault
    
    public static string Pseudonymize(string value)
    {
        // HMAC-based pseudonym — детерминированный, необратимый без ключа
        var hmac = new HMACSHA256(_hmacKey);
        var hash = hmac.ComputeHash(Encoding.UTF8.GetBytes(value));
        return Convert.ToBase64String(hash)[..16];  // truncate
    }
    
    public static string MaskEmail(string email)
    {
        var parts = email.Split('@');
        if (parts.Length != 2) return email;
        return $"{parts[0][0]}***@{parts[1]}";
    }
    
    public static string MaskPhone(string phone)
    {
        if (phone.Length < 4) return phone;
        return new string('*', phone.Length - 4) + phone[^4..];
    }
}
```

**Факт:** GDPR различает **anonymization** (необратимо — данные больше не PII) и **pseudonymization** (обратимо с ключём — остаётся PII).

---

## Чек-лист

- [ ] TDE vs Always Encrypted — когда что
- [ ] Deterministic vs Randomized encryption (Always Encrypted)
- [ ] Server-side encryption (SSE) vs Client-side encryption
- [ ] TLS 1.2/1.3 обязателен, TLS 1.0/1.1 отключить
- [ ] mTLS для service-to-service
- [ ] Key Management: Key Vault, HSM
- [ ] Dynamic Data Masking (SQL Server)
- [ ] Anonymization vs Pseudonymization (GDPR)

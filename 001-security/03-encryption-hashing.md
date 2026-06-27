# Шифрование и хеширование в .NET — факты + примеры

---

## Шифрование

### Факты

| Тип | Алгоритм | Когда использовать | .NET класс |
|---|---|---|---|
| **Симметричное** | AES-256-GCM | Шифрование данных в покое (БД, файлы) | `Aes.Create()` |
| **Симметричное** | AES-256-CBC | Legacy-совместимость (GCM предпочтительнее) | `Aes.Create()` |
| **Асимметричное** | RSA-2048/4096 | Обмен ключами, цифровая подпись | `RSA.Create()` |
| **Асимметричное** | ECDSA | Цифровая подпись (быстрее RSA) | `ECDsa.Create()` |
| **Асимметричное** | ECDH | Обмен ключами | `ECDiffieHellman.Create()` |

**Факт:** AES-GCM — **рекомендуемый** симметричный шифр: обеспечивает **confidentiality + integrity** (authenticated encryption). AES-CBC требует отдельного HMAC.

**Факт:** AES-GCM в .NET использует **hardware acceleration** (AES-NI) — скорость > 1 GB/s на современных CPU.

### AES-GCM (рекомендуемый способ)

```csharp
public static byte[] Encrypt(byte[] plaintext, byte[] key, byte[]? nonce = null)
{
    using var aes = new AesGcm(key, tagSizeInBytes: 16);  // 128-bit tag
    
    nonce ??= RandomNumberGenerator.GetBytes(AesGcm.NonceByteSizes.MinSize); // 12 bytes
    
    var ciphertext = new byte[plaintext.Length];
    var tag = new byte[AesGcm.TagByteSizes.MaxSize];
    
    aes.Encrypt(nonce, plaintext, ciphertext, tag);
    
    // Возвращаем nonce + ciphertext + tag
    var result = new byte[nonce.Length + ciphertext.Length + tag.Length];
    nonce.CopyTo(result, 0);
    ciphertext.CopyTo(result, nonce.Length);
    tag.CopyTo(result, nonce.Length + ciphertext.Length);
    
    return result;
}

public static byte[] Decrypt(byte[] data, byte[] key)
{
    var nonceSize = AesGcm.NonceByteSizes.MinSize;
    var tagSize = AesGcm.TagByteSizes.MaxSize;
    
    var nonce = data[..nonceSize];
    var tag = data[^tagSize..];
    var ciphertext = data[nonceSize..^tagSize];
    
    using var aes = new AesGcm(key, tagSizeInBytes: 16);
    var plaintext = new byte[ciphertext.Length];
    
    aes.Decrypt(nonce, ciphertext, tag, plaintext);
    return plaintext;
}
```

**Факт:** Nonce (IV) для AES-GCM — **12 байт** (96 бит). Должен быть уникальным для каждого шифрования с одним ключом. Никогда не повторяйте nonce с одним ключом.

**Факт:** Для AES-GCM макс. объём данных на одном ключе — ~2^32 блоков (64 GB). Для больших объёмов — ротация ключей.

### RSA (асимметричное)

```csharp
// Генерация ключей
using var rsa = RSA.Create(2048);  // 2048 — минимум, 4096 — для высокой безопасности
var privateKey = rsa.ExportRSAPrivateKey();
var publicKey = rsa.ExportRSAPublicKey();

// Шифрование (макс. размер = keySize/8 - 42 для OAEP-SHA256)
var encrypted = rsa.Encrypt(data, RSAEncryptionPadding.OaepSHA256);

// Расшифровка
var decrypted = rsa.Decrypt(encrypted, RSAEncryptionPadding.OaepSHA256);

// Подпись
var signature = rsa.SignData(data, HashAlgorithmName.SHA256, RSASignaturePadding.Pss);

// Верификация
var valid = rsa.VerifyData(data, signature, HashAlgorithmName.SHA256, RSASignaturePadding.Pss);
```

**Факт:** RSA не предназначен для шифрования больших данных — макс. размер = (keySize/8) - 2×hashSize/8 - 2. Для RSA-2048 + OAEP-SHA256 = 190 байт. Используйте **hybrid encryption**: RSA для ключа, AES для данных.

### Hybrid Encryption (RSA + AES)

```csharp
public (byte[] encryptedKey, byte[] encryptedData) HybridEncrypt(byte[] data, RSA publicKey)
{
    // 1. Генерируем случайный AES-ключ
    var aesKey = RandomNumberGenerator.GetBytes(32);
    
    // 2. Шифруем данные AES
    var encryptedData = AesGcmEncrypt(data, aesKey);
    
    // 3. Шифруем AES-ключ RSA
    var encryptedKey = publicKey.Encrypt(aesKey, RSAEncryptionPadding.OaepSHA256);
    
    return (encryptedKey, encryptedData);
}
```

---

## Data Protection API (DPAPI) ASP.NET Core

**Факты:**
- **Не используйте** для долгоживущих данных (гифки, файлы) — ключи ротируются
- Идеально для: cookies, anti-CSRF tokens, temp data, query strings
- Ключи хранятся в файловой системе по умолчанию (или Azure Blob/Redis/Key Vault)
- Автоматическая ротация ключей

```csharp
builder.Services.AddDataProtection()
    .PersistKeysToAzureBlobStorage(connectionString, "key-ring", "keys.xml")
    .ProtectKeysWithAzureKeyVault(keyIdentifier, credential)
    .SetApplicationName("my-app")          // Изоляция между приложениями
    .SetDefaultKeyLifetime(TimeSpan.FromDays(90));  // Ротация каждые 90 дней

public class TokenProtector
{
    private readonly IDataProtector _protector;
    
    public TokenProtector(IDataProtectionProvider provider)
    {
        _protector = provider.CreateProtector("tokens.v1");  // Purpose — изоляция
    }
    
    public string Protect(string data) => _protector.Protect(data);
    public string Unprotect(string data) => _protector.Unprotect(data);
    
    // Time-limited protection
    public string ProtectTimed(string data)
    {
        return _protector.ToTimeLimitedDataProtector()
            .Protect(data, lifetime: TimeSpan.FromHours(1));
    }
}
```

**Факт:** **Purpose string** изолирует данные: если "purpose" различается, `Unprotect` упадёт с `CryptographicException`.

---

## Хеширование паролей

**Факты:**

| Алгоритм | Итерации (2024) | Стойкость | .NET поддержка |
|---|---|---|---|
| **PBKDF2** | 600 000+ (SHA-256) | Высокая | `Rfc2898DeriveBytes` (built-in) |
| **bcrypt** | 10-12 (cost factor) | Высокая | NuGet `BCrypt.Net-Next` |
| **scrypt** | N=2^14, r=8, p=1 | Высокая | NuGet `Scrypt.NET` |
| **argon2id** | t=3, m=64MB, p=4 | Очень высокая (победитель PHC) | NuGet `Konscious.Security.Cryptography` |
| MD5/SHA1 | — | **Сломан** (коллизии) | Не использовать |

**Правильный PBKDF2 (ASP.NET Core Identity):**

```csharp
public static string HashPassword(string password)
{
    var salt = RandomNumberGenerator.GetBytes(16);
    var hash = Rfc2898DeriveBytes.Pbkdf2(
        password,
        salt,
        iterations: 600_000,        // .NET 8+ default
        HashAlgorithmName.SHA256,
        outputLength: 32);
    
    // Формат: {iterations}.{salt}.{hash} (base64)
    return $"{600000}.{Convert.ToBase64String(salt)}.{Convert.ToBase64String(hash)}";
}

public static bool VerifyPassword(string password, string storedHash)
{
    var parts = storedHash.Split('.');
    var iterations = int.Parse(parts[0]);
    var salt = Convert.FromBase64String(parts[1]);
    var hash = Convert.FromBase64String(parts[2]);
    
    var computedHash = Rfc2898DeriveBytes.Pbkdf2(
        password, salt, iterations, HashAlgorithmName.SHA256, 32);
    
    return CryptographicOperations.FixedTimeEquals(hash, computedHash);  // Constant-time
}
```

**Факт:** `CryptographicOperations.FixedTimeEquals` — **обязателен** для сравнения хешей. Обычное `==` подвержено **timing attack**.

**Факт:** В .NET 8+ `Rfc2898DeriveBytes` по умолчанию использует **600 000 итераций**. В .NET 7 было 100 000. В .NET Framework было 1000.

**Факт:** argon2id — **рекомендуемый** алгоритм для новых проектов. Он memory-hard (устойчив к GPU/ASIC брутфорсу).

---

## Random Number Generation

```csharp
// ❌ System.Random — НЕ криптостойкий, предсказуемый
var rng = new Random();
var password = rng.Next();  // Предсказуем при известном seed

// ✅ Криптостойкий — RandomNumberGenerator
var key = RandomNumberGenerator.GetBytes(32);  // .NET 6+
var intValue = RandomNumberGenerator.GetInt32(1000);  // .NET 6+
```

**Факт:** `Guid.NewGuid()` — не для криптографии (алгоритм — version 4 random, но не гарантирует криптостойкость). Используйте `RandomNumberGenerator`.

---

## Certificate Validation

```csharp
// ❌ Опасно — доверяет любому сертификату
handler.ServerCertificateCustomValidationCallback = (_, _, _, _) => true;

// ✅ Правильная валидация
handler.ServerCertificateCustomValidationCallback = (sender, cert, chain, errors) =>
{
    if (errors == SslPolicyErrors.None) return true;
    
    // Allow self-signed in dev only
    if (errors == SslPolicyErrors.RemoteCertificateChainErrors)
    {
        // Проверка отпечатка
        return cert?.Thumbprint == knownThumbprint;
    }
    
    return false;
};
```

---

## ChaCha20-Poly1305

**Факты:**
- Альтернатива AES-GCM для платформ без AES-NI (ARM, mobile, старые x86)
- Используется в TLS 1.3 как обязательный cipher suite
- В .NET доступен через `System.Security.Cryptography.ChaCha20Poly1305` (.NET 6+)

```csharp
public static byte[] EncryptChaCha(byte[] plaintext, byte[] key)
{
    using var chacha = new ChaCha20Poly1305(key);
    
    var nonce = RandomNumberGenerator.GetBytes(12);  // ChaCha20 nonce = 96 бит (как AES-GCM)
    var ciphertext = new byte[plaintext.Length];
    var tag = new byte[16];  // Poly1305 tag = 128 бит
    
    chacha.Encrypt(nonce, plaintext, ciphertext, tag);
    
    // nonce (12) + ciphertext + tag (16)
    var result = new byte[12 + ciphertext.Length + 16];
    nonce.CopyTo(result, 0);
    ciphertext.CopyTo(result, 12);
    tag.CopyTo(result, 12 + ciphertext.Length);
    
    return result;
}
```

**Факт:** AES-GCM быстрее на x86 с AES-NI (~3-5 GB/s). ChaCha20-Poly1305 быстрее на ARM (~1-2 GB/s без аппаратного ускорения). Выбирайте в зависимости от целевой платформы.

## X.509 Certificate Management

```csharp
// Генерация self-signed сертификата (dev/testing)
using var ecdsa = ECDsa.Create(ECCurve.NamedCurves.nistP256);
var req = new CertificateRequest("CN=localhost", ecdsa, HashAlgorithmName.SHA256);
req.CertificateExtensions.Add(
    new X509BasicConstraintsExtension(false, false, 0, false));
req.CertificateExtensions.Add(
    new X509KeyUsageExtension(X509KeyUsageFlags.DigitalSignature | X509KeyUsageFlags.KeyEncipherment, false));
req.CertificateExtensions.Add(
    new X509SubjectAlternativeNameBuilder { DnsNames = { "localhost" } }.Build());

var cert = req.CreateSelfSigned(DateTimeOffset.UtcNow, DateTimeOffset.UtcNow.AddYears(1));

// Экспорт в PFX (с private key)
var pfxBytes = cert.Export(X509ContentType.Pfx, "password123");

// Загрузка из Azure Key Vault (production)
var secretClient = new SecretClient(new Uri("https://myvault.vault.azure.net/"), new DefaultAzureCredential());
var certSecret = await secretClient.GetSecretAsync("myapp-cert");
var certBytes = Convert.FromBase64String(certSecret.Value.Value);
var certificate = new X509Certificate2(certBytes, "", X509KeyStorageFlags.MachineKeySet);

// Kestrel — загрузка сертификата
builder.WebHost.ConfigureKestrel(options =>
{
    options.ConfigureHttpsDefaults(httpsOptions =>
    {
        httpsOptions.ServerCertificate = certificate;
        httpsOptions.SslProtocols = SslProtocols.Tls13;  // Только TLS 1.3
    });
});
```

## EncryptedXml (XML Encryption)

```csharp
// Шифрование XML-документа (SAML assertions, SOAP)
public static XmlDocument EncryptXml(XmlDocument doc, RSA publicKey)
{
    var encryptedXml = new EncryptedXml();
    var sessionKey = RandomNumberGenerator.GetBytes(32);  // AES-256
    
    // Шифруем элемент (например, <saml:Assertion>)
    var elementToEncrypt = doc.GetElementsByTagName("saml:Assertion")[0] as XmlElement;
    var encryptedData = encryptedXml.EncryptData(elementToEncrypt, sessionKey,
        new SymmetricAlgorithm { Key = sessionKey });
    
    // Шифруем session key RSA (key transport)
    var encryptedKey = new EncryptedKey();
    encryptedKey.CipherData = new CipherData(
        publicKey.Encrypt(sessionKey, RSAEncryptionPadding.OaepSHA256));
    
    // AES-256-CBC (EncryptedXml не поддерживает GCM)
    encryptedData.EncryptionMethod = new EncryptionMethod(
        "http://www.w3.org/2001/04/xmlenc#aes256-cbc");
    
    EncryptedXml.ReplaceElement(elementToEncrypt, encryptedData, false);
    return doc;
}
```

**Факт:** `EncryptedXml` использует AES-CBC (не GCM) — добавляйте HMAC отдельно для authenticated encryption XML.

---

## Чек-лист

- [ ] AES-GCM для симметричного шифрования (Nonce + Ciphertext + Tag)
- [ ] ChaCha20-Poly1305 для ARM/mobile платформ
- [ ] RSA для обмена ключами, подписи
- [ ] ECDSA для цифровой подписи (быстрее, меньше ключи)
- [ ] Hybrid encryption (RSA + AES) для больших данных
- [ ] DPAPI для временных данных ASP.NET Core
- [ ] PBKDF2 / argon2id для паролей
- [ ] `FixedTimeEquals` для сравнения хешей
- [ ] `RandomNumberGenerator` вместо `System.Random`
- [ ] Не отключать certificate validation
- [ ] X.509 сертификаты из Key Vault, не в файлах

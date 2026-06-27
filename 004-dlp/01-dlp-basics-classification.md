# Основы DLP и классификация данных

---

## Что такое DLP (Data Loss Prevention)

**Факты:**
- DLP — система мер и технологий для предотвращения утечки, утраты и несанкционированного доступа к данным
- Три состояния данных:
  - **Data at Rest** — данные в хранилище (БД, файлы, бэкапы)
  - **Data in Transit** — данные в передаче (сеть, API)
  - **Data in Use** — данные в обработке (RAM, CPU)

**Три типа DLP-решений:**

| Тип | Описание | Примеры |
|---|---|---|
| **Network DLP** | Мониторинг сетевого трафика | Сигнатуры, DPI, email filtering |
| **Endpoint DLP** | Агенты на рабочих станциях | USB-block, clipboard monitor, screen capture prevention |
| **Cloud DLP** | Защита в SaaS/IaaS/PaaS | Azure Purview, Microsoft 365 DLP, CASB |

---

## Типы защищаемых данных

| Категория | Примеры | Регуляция |
|---|---|---|
| **PII** (Personally Identifiable Info) | Имя, адрес, телефон, email, IP | GDPR, CCPA, LGPD |
| **PHI** (Protected Health Info) | Медицинские записи, диагнозы | HIPAA, HITECH |
| **PCI** (Payment Card Industry) | Номер карты, CVV, PIN | PCI DSS |
| **FinData** | Банковские счета, транзакции | SOX, GLBA |
| **IP** (Intellectual Property) | Исходный код, патенты, trade secrets | Various |

**Факт:** PCI DSS запрещает хранить CVV после авторизации. Номер карты можно хранить только в замаскированном виде (последние 4 цифры).

**Факт:** GDPR требует **Data Protection Impact Assessment (DPIA)** для обработки sensitive data.

---

## Классификация данных (Data Classification)

**Уровни классификации (ISO 27001 / common):**

| Уровень | Пример | Маркировка |
|---|---|---|
| **Public** | Маркетинговые материалы | Зелёный |
| **Internal** | Внутренние инструкции, org-диаграммы | Синий |
| **Confidential** | Данные клиентов, финансовые отчёты | Жёлтый |
| **Restricted** | Trade secrets, PII, пароли | Красный |

**Реализация классификации в .NET:**

```csharp
// Атрибут классификации
[AttributeUsage(AttributeTargets.Class | AttributeTargets.Property)]
public class DataClassificationAttribute : Attribute
{
    public ClassificationLevel Level { get; }
    public string? Category { get; }
    
    public DataClassificationAttribute(ClassificationLevel level, string? category = null)
    {
        Level = level;
        Category = category;
    }
}

public enum ClassificationLevel
{
    Public = 0,
    Internal = 1,
    Confidential = 2,
    Restricted = 3
}

// Применение к моделям
public class UserProfile
{
    [DataClassification(ClassificationLevel.Public)]
    public string DisplayName { get; set; }
    
    [DataClassification(ClassificationLevel.Restricted, "PII")]
    public string Email { get; set; }
    
    [DataClassification(ClassificationLevel.Restricted, "PII")]
    public string Phone { get; set; }
    
    [DataClassification(ClassificationLevel.Confidential)]
    public string Department { get; set; }
}
```

**Факт:** Microsoft Purview Information Protection поддерживает **automatic classification** (ML-based) и **manual classification** (пользователь выбирает метку).

---

## Data Discovery — обнаружение чувствительных данных

**Факты:**
- **Structured:** Сканирование БД (колонки с SSN, Credit Card, Email)
- **Unstructured:** Сканирование файловых хранилищ (SharePoint, OneDrive, S3)
- **Pattern-based:** Регулярные выражения для PII

```csharp
// Data Discovery — регулярные выражения для PII
public static class PiiDetector
{
    // Email
    private static readonly Regex EmailRegex = new(
        @"\b[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}\b",
        RegexOptions.IgnoreCase | RegexOptions.Compiled);
    
    // SSN (US) — 123-45-6789
    private static readonly Regex SsnRegex = new(
        @"\b\d{3}-\d{2}-\d{4}\b",
        RegexOptions.Compiled);
    
    // Credit Card (Luhn validated)
    private static readonly Regex CcRegex = new(
        @"\b(?:\d[ -]*?){13,16}\b",
        RegexOptions.Compiled);
    
    // Phone (E.164)
    private static readonly Regex PhoneRegex = new(
        @"\+?[1-9]\d{1,14}\b",
        RegexOptions.Compiled);
    
    public static IEnumerable<PiiMatch> FindPii(string text, string? context = null)
    {
        var matches = new List<PiiMatch>();
        
        foreach (Match match in EmailRegex.Matches(text))
            matches.Add(new PiiMatch("Email", match.Value, match.Index, context));
        
        foreach (Match match in SsnRegex.Matches(text))
            matches.Add(new PiiMatch("SSN", match.Value, match.Index, context));
        
        foreach (Match match in CcRegex.Matches(text))
        {
            // Luhn validation
            var cleaned = string.Join("", match.Value.Where(char.IsDigit));
            if (IsLuhnValid(cleaned))
                matches.Add(new PiiMatch("CreditCard", MaskCc(cleaned), match.Index, context));
        }
        
        return matches;
    }
    
    private static bool IsLuhnValid(string digits)
    {
        // Luhn algorithm — проверка контрольной суммы кредитных карт
        int sum = 0;
        bool alternate = false;
        for (int i = digits.Length - 1; i >= 0; i--)
        {
            int n = digits[i] - '0';
            if (alternate)
            {
                n *= 2;
                if (n > 9) n -= 9;
            }
            sum += n;
            alternate = !alternate;
        }
        return sum % 10 == 0;
    }
    
    private static string MaskCc(string cc) 
        => $"****-****-****-{cc[^4..]}";
}

public record PiiMatch(string Type, string Value, int Position, string? Context);
```

---

## Data Inventory — карта данных

**Факт:** GDPR Article 30 требует **Record of Processing Activities (ROPA)** — карту всех потоков данных:

```csharp
// Data Flow Map — документация потоков данных
public record DataFlow(
    string DataElement,           // Email
    ClassificationLevel Level,    // Restricted
    string Source,                // Registration Form
    string[] Destinations,        // {CRM, Newsletter Service, Analytics}
    string Storage,               // Azure SQL (TDE-enabled)
    string RetentionPolicy,       // 3 years after account deletion
    bool IsEncryptedAtRest,       // true
    bool IsEncryptedInTransit,    // true
    string[] SharedWithThirdParties  // {Mailchimp (DPA signed)}
);
```

---

## Data Retention Policies

| Тип данных | Срок хранения | Основание |
|---|---|---|
| Логи аудита | 1-7 лет | SOX, PCI DSS |
| Финансовые записи | 7 лет | Налоговое законодательство |
| PII | Пока есть business need | GDPR (right to erasure) |
| Backup'ы | 30-90 дней (циклические) | Business continuity |

```csharp
// Data Retention — политика хранения
[AttributeUsage(AttributeTargets.Class)]
public class RetentionPolicyAttribute : Attribute
{
    public TimeSpan RetentionPeriod { get; }
    public RetentionAction Action { get; }
    
    public RetentionPolicyAttribute(int days, RetentionAction action = RetentionAction.Delete)
    {
        RetentionPeriod = TimeSpan.FromDays(days);
        Action = action;
    }
}

public enum RetentionAction
{
    Delete,           // Полное удаление
    Anonymize,        // Анонимизация (замена на "*")
    Archive,          // Перемещение в cold storage
    Mask              // Маскирование sensitive полей
}

[RetentionPolicy(365, RetentionAction.Delete)]
public class UserLoginHistory { ... }

[RetentionPolicy(2555, RetentionAction.Archive)]  // 7 years
public class FinancialTransaction { ... }
```

---

## Чек-лист

- [ ] Три состояния данных: at rest, in transit, in use
- [ ] Network DLP vs Endpoint DLP vs Cloud DLP
- [ ] PII / PHI / PCI / FinData / IP — различия
- [ ] Классификация: Public, Internal, Confidential, Restricted
- [ ] Data Discovery: regex-based PII detection, Luhn validation
- [ ] Data Inventory / ROPA (GDPR Article 30)
- [ ] Retention policies: сроки и действия (delete, anonymize, archive)

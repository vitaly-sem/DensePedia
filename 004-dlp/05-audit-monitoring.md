# Аудит, мониторинг и реагирование

---

## Audit Trails — неизменяемые логи

**Факты:**
- **Audit trail** — запись кто, что, когда сделал с данными
- **Immutable** — нельзя изменить или удалить (append-only)
- **Chain of custody** — доказательная цепочка для суда

### Audit Logging в .NET

```csharp
// Audit-логи должны быть отдельно от application-логов
// Immutable storage: SQL Ledger, Azure Blob (WORM), Elasticsearch (append-only)

public class AuditEntry
{
    public long Id { get; set; }
    public DateTime Timestamp { get; set; }
    public string UserId { get; set; }
    public string Action { get; set; }        // CREATE, READ, UPDATE, DELETE, EXPORT
    public string ResourceType { get; set; }  // PatientRecord, FinancialTransaction
    public string ResourceId { get; set; }
    public string? OldValueHash { get; set; }  // Hash of previous state
    public string? NewValueHash { get; set; }
    public string IpAddress { get; set; }
    public string UserAgent { get; set; }
    public AuditSeverity Severity { get; set; }
    public string CorrelationId { get; set; }
}

// Interceptor для автоматического аудита (EF Core)
public class AuditSaveInterceptor : SaveChangesInterceptor
{
    private readonly ICurrentUserService _userService;
    
    public override async ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData, InterceptionResult<int> result,
        CancellationToken cancellationToken = default)
    {
        var context = eventData.Context;
        var userId = _userService.UserId;
        var ip = _userService.IpAddress;
        var timestamp = DateTime.UtcNow;
        
        var auditEntries = new List<AuditEntry>();
        
        foreach (var entry in context.ChangeTracker.Entries())
        {
            if (entry.Entity is IAuditable)
            {
                var audit = new AuditEntry
                {
                    Timestamp = timestamp,
                    UserId = userId,
                    Action = entry.State switch
                    {
                        EntityState.Added => "CREATE",
                        EntityState.Modified => "UPDATE",
                        EntityState.Deleted => "DELETE",
                        _ => null
                    },
                    ResourceType = entry.Entity.GetType().Name,
                    ResourceId = GetPrimaryKey(entry),
                    IpAddress = ip,
                    CorrelationId = _userService.CorrelationId
                };
                
                if (audit.Action != null)
                    auditEntries.Add(audit);
            }
        }
        
        // Сохраняем audit entries в отдельную immutable таблицу
        if (auditEntries.Count > 0)
        {
            context.Set<AuditEntry>().AddRange(auditEntries);
        }
        
        return await base.SavingChangesAsync(eventData, result, cancellationToken);
    }
}
```

### Integrity verification — проверка целостности audit log

```csharp
// Chain hashing — каждый log содержит hash предыдущего
// Обнаружение: если кто-то изменил log #5 → hash в log #6 не совпадает

public class ChainHashedAuditLog
{
    public long Id { get; set; }
    public DateTime Timestamp { get; set; }
    public string Data { get; set; }
    public string PreviousHash { get; set; }
    public string Hash { get; set; }
    
    public static string ComputeHash(ChainHashedAuditLog entry)
    {
        var data = $"{entry.Id}|{entry.Timestamp:O}|{entry.Data}|{entry.PreviousHash}";
        return Convert.ToHexString(SHA256.HashData(Encoding.UTF8.GetBytes(data)));
    }
    
    public static bool VerifyChain(IEnumerable<ChainHashedAuditLog> logs)
    {
        string? previousHash = null;
        foreach (var log in logs)
        {
            if (previousHash != log.PreviousHash)
                return false;  // Цепочка нарушена!
            
            if (ComputeHash(log) != log.Hash)
                return false;  // Log изменён!
            
            previousHash = log.Hash;
        }
        return true;
    }
}
```

---

## SIEM Integration — Security Information and Event Management

**Факты:**
- SIEM = централизованный сбор и анализ событий безопасности
- Популярные: Splunk, Azure Sentinel, Elastic SIEM, QRadar
- **Syslog** / **CEF** (Common Event Format) — стандартные форматы
- .NET приложения отправляют audit events в SIEM

### Azure Sentinel — интеграция

```csharp
// Отправка audit events в Azure Sentinel через Log Analytics
public class SentinelAuditSink : IAuditSink
{
    private readonly LogAnalyticsClient _client;
    
    public SentinelAuditSink(string workspaceId, string sharedKey)
    {
        _client = new LogAnalyticsClient(workspaceId, sharedKey);
    }
    
    public async Task SendAsync(AuditEntry entry)
    {
        // Custom log type — DLP_CL
        var logEntry = new
        {
            Timestamp = entry.Timestamp,
            UserId = entry.UserId,
            Action = entry.Action,
            ResourceType = entry.ResourceType,
            ResourceId = entry.ResourceId,
            IpAddress = entry.IpAddress,
            Severity = entry.Severity.ToString(),
            CorrelationId = entry.CorrelationId
        };
        
        await _client.IngestAsync("DLP_CL", logEntry);
    }
}

// В Program.cs
builder.Services.AddSingleton<IAuditSink, SentinelAuditSink>();
```

### Splunk HTTP Event Collector (HEC)

```csharp
public class SplunkAuditSink : IAuditSink
{
    private readonly HttpClient _http;
    private readonly string _hecUrl;
    private readonly string _token;
    
    public SplunkAuditSink(string hecUrl, string token)
    {
        _http = new HttpClient();
        _hecUrl = hecUrl;
        _token = token;
    }
    
    public async Task SendAsync(AuditEntry entry)
    {
        var splunkEvent = new
        {
            sourcetype = "dlp_audit",
            source = "dotnet-api",
            host = Environment.MachineName,
            @event = entry
        };
        
        var request = new HttpRequestMessage(HttpMethod.Post, _hecUrl)
        {
            Headers = { { "Authorization", $"Splunk {_token}" } },
            Content = new StringContent(
                JsonSerializer.Serialize(splunkEvent),
                Encoding.UTF8,
                "application/json")
        };
        
        await _http.SendAsync(request);
    }
}
```

---

## Azure Monitor / Application Insights

```csharp
// Application Insights — мониторинг приложения
builder.Services.AddApplicationInsightsTelemetry(options =>
{
    options.ConnectionString = "InstrumentationKey=...;IngestionEndpoint=...";
});

// Кастомные метрики DLP
public class DlpMetrics
{
    private readonly TelemetryClient _telemetry;
    
    public DlpMetrics(TelemetryClient telemetry)
    {
        _telemetry = telemetry;
    }
    
    public void TrackPiiAccess(string userId, string resourceType, bool allowed)
    {
        _telemetry.TrackEvent("PiiAccess", new Dictionary<string, string>
        {
            ["UserId"] = HashUserId(userId),   // Hash — не хранить raw PII
            ["ResourceType"] = resourceType,
            ["Allowed"] = allowed.ToString()
        });
    }
    
    public void TrackDlpAlert(string rule, string severity)
    {
        _telemetry.TrackMetric("DlpAlert", 1, new Dictionary<string, string>
        {
            ["Rule"] = rule,
            ["Severity"] = severity
        });
    }
    
    private static string HashUserId(string userId)
    {
        // Не храним raw userId в метриках
        return Convert.ToHexString(SHA256.HashData(Encoding.UTF8.GetBytes(userId)));
    }
}
```

---

## UEBA (User and Entity Behavior Analytics)

**Факты:**
- ML-based анализ поведения пользователей
- Baseline: типичное поведение (часы работы, объём данных, время доступа)
- Anomaly detection: массовый экспорт, доступ в 3am, access from new location
- Интеграция с Azure Sentinel UEBA

```csharp
// Детекция аномалий — эвристика для DLP
public class AnomalyDetector
{
    private readonly Dictionary<string, UserBaseline> _baselines = new();
    
    public DlpAlertSeverity AnalyzeAccess(AccessRequest request)
    {
        var severity = DlpAlertSeverity.Low;
        
        // 1. Время доступа — нерабочее время?
        if (request.Timestamp.Hour < 8 || request.Timestamp.Hour > 19)
            severity = DlpAlertSeverity.Medium;
        
        // 2. Объём данных — превышение baseline?
        if (_baselines.TryGetValue(request.UserId, out var baseline))
        {
            if (request.DataSize > baseline.AverageDataSize * 3)
                severity = DlpAlertSeverity.High;
        }
        
        // 3. Новое устройство / IP?
        if (!request.IsKnownDevice)
            severity = DlpAlertSeverity.High;
        
        // 4. Частота — слишком много запросов?
        if (request.RecentAccessCount > 100)
            severity = DlpAlertSeverity.Critical;
        
        return severity;
    }
}
```

---

## Incident Response Plan

**Фазы реагирования на инцидент DLP:**

```
1. Идентификация
  - SIEM alert или пользователь сообщил
  - Подтверждение: реальная утечка или false positive?

2. Сдерживание (Containment)
  - Отозвать доступ пользователя
  - Заблокировать compromised credential
  - Остановить экспорт/деактивировать интеграцию

3. Исследование (Investigation)
  - Audit trail: кто, что, когда
  - Data discovery: какие данные утекли
  - Impact assessment: PII, PHI, PCI?

4. Устранение (Eradication)
  - Удалить вредоносное ПО / закрыть уязвимость
  - Rotate keys и credentials

5. Восстановление (Recovery)
  - Восстановить данные из backup (без compromised данных)
  - Вернуть доступ (если безопасно)

6. Post-mortem (Lessons Learned)
  - Почему произошло?
  - Что изменить в DLP политиках?
  - Update runbook
```

```csharp
// Incident Response — автоматическое реагирование
public class DlpResponder
{
    private readonly IUserService _users;
    private readonly ICredentialManager _credentials;
    private readonly IAuditSink _audit;
    
    public async Task RespondAsync(DlpAlert alert)
    {
        _audit.SendAsync(new AuditEntry
        {
            Action = "DLP_RESPOND",
            ResourceType = alert.ResourceType,
            Severity = alert.Severity,
            // ...
        });
        
        switch (alert.Severity)
        {
            case DlpAlertSeverity.Critical:
                // Немедленная блокировка
                await _users.RevokeAccessAsync(alert.UserId);
                await _credentials.RotateAsync(alert.UserId);
                await NotifySecOpsAsync(alert, "CRITICAL");
                break;
                
            case DlpAlertSeverity.High:
                // Review required
                await _users.DisableAccessAsync(alert.UserId);
                await NotifyManagerAsync(alert.UserId);
                break;
                
            case DlpAlertSeverity.Medium:
                // Log and monitor
                await AddSurveillanceAsync(alert.UserId);
                break;
        }
    }
}
```

---

## DLP Compliance Reporting

**Факты:**
- **GDPR** — Data Breach Notification в течение 72 часов
- **PCI DSS** — annual compliance report
- **HIPAA** — breach notification affected individuals
- **SOX** — финансовые отчёты + audit trails (7 лет)

```csharp
// Compliance Dashboard — отчётность
public class ComplianceReport
{
    public DateTime ReportingPeriod { get; set; }
    public int TotalPiiAccessEvents { get; set; }
    public int DlpAlertsGenerated { get; set; }
    public int IncidentsEscalated { get; set; }
    public double MeanTimeToRespond { get; set; }  // Minutes
    public Dictionary<string, int> TopViolatedRules { get; set; }
    public List<DataSubjectRequest> DataSubjectRequests { get; set; }
}

// Data Subject Request (DSR) — GDPR Right to Erasure
public async Task HandleRightToErasureAsync(string userId)
{
    // 1. Идентификация всех систем, где есть данные пользователя
    var dataMap = await _dataInventory.FindAsync(userId);
    
    // 2. Анонимизация или удаление
    foreach (var entry in dataMap)
    {
        await _anonymizer.AnonymizeAsync(entry);
    }
    
    // 3. Подтверждение выполнения
    await _audit.LogDsrAsync(userId, "RightToErasure");
}
```

---

## Чек-лист

- [ ] Audit trails: immutable, chain hashing, append-only
- [ ] SIEM: Azure Sentinel, Splunk HEC, CEF format
- [ ] Application Insights: DLP metrics, PiiAccess events
- [ ] UEBA: anomaly detection, user baselines
- [ ] Incident Response: 6 фаз (ID → Contain → Investigate → Eradicate → Recover → Post-mortem)
- [ ] Автоматическое реагирование: revoke access, rotate credentials
- [ ] Compliance: GDPR 72h breach notification, DSR (Right to Erasure)
- [ ] Data Inventory Map для DSR

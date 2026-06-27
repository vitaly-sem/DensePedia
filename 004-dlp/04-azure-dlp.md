# DLP в облаке (Azure, Microsoft 365)

---

## Microsoft Purview — комплексное DLP-решение

**Факты:**
- Преемник Azure Information Protection + Microsoft 365 DLP
- Покрывает: Exchange, SharePoint, OneDrive, Teams, Endpoints, SQL, Storage
- **Sensitivity Labels** — метки конфиденциальности для файлов и email
- **Data classification** — автоматическое обнаружение PII, PCI, PHI

### Sensitivity Labels

```
[Restricted]
├── Label: Highly Confidential
├── Encryption: Microsoft 365 (user-defined permissions)
├── Visual marking: Header "HIGHLY CONFIDENTIAL", Footer with user info
└── Auto-labeling: Content contains SSN, Credit Card
```

```csharp
// Azure SDK — программное применение меток
using Azure.Identity;
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Models;

public async Task ApplySensitivityLabelAsync(string blobUrl, string labelId)
{
    var blobClient = new BlobClient(new Uri(blobUrl), new DefaultAzureCredential());
    
    // Apply label as metadata
    await blobClient.SetMetadataAsync(new Dictionary<string, string>
    {
        ["MSIP_Label_${labelId}_Enabled"] = "true",
        ["MSIP_Label_${labelId}_Method"] = "Standard",
        ["MSIP_Label_${labelId}_SetDate"] = DateTime.UtcNow.ToString("O")
    });
}
```

**Факт:** Sensitivity Labels — это **Azure AD Conditional Access** для данных. Можно запретить доступ к метке `Highly Confidential` с незарегистрированных устройств.

---

## Microsoft 365 DLP

**Факты:**
- **Политики DLP** — правила, что можно делать с sensitive data (нельзя отправить SSN в email внешнему получателю)
- **DLP rules** для: Exchange, SharePoint, OneDrive, Teams, Endpoints
- **Policy tips** — уведомление пользователя (не отправляйте этот файл)
- **DLP reports** — инциденты в Compliance Center

```xml
<!-- Пример политики DLP (через PowerShell) -->
<Policy>
  <Name>Credit Card Protection</Name>
  <Rules>
    <Rule>
      <Conditions>
        <ContentContainsSensitiveInformation>
          <SensitiveInfoType id="Credit Card Number" />
        </ContentContainsSensitiveInformation>
      </Conditions>
      <Actions>
        <!-- Блокировать отправку внешнему получателю -->
        <BlockAccess>
          <Message>This email contains credit card information and cannot be sent externally.</Message>
        </BlockAccess>
        <!-- Уведомить Compliance Officer -->
        <NotifyUser>
          <NotifyUser>
            <OverrideOption>False</OverrideOption>
          </NotifyUser>
        </NotifyUser>
        <GenerateIncidentAlert>
          <Severity>High</Severity>
        </GenerateIncidentAlert>
      </Actions>
    </Rule>
  </Rules>
</Policy>
```

---

## Azure SQL — DLP на уровне БД

### SQL Audit — аудит доступа

```sql
-- Включить аудит
CREATE SERVER AUDIT DlpAudit
TO APPLICATION_LOG
WITH (QUEUE_DELAY = 1000);

-- Аудит на уровне БД
CREATE DATABASE AUDIT SPECIFICATION DlpAuditSpec
FOR SERVER AUDIT DlpAudit
ADD (SELECT ON OBJECT::dbo.Users BY public),  -- Кто читает Users
ADD (DELETE ON OBJECT::dbo.Users BY dba_role),  -- Кто удаляет
ADD (DATABASE_PERMISSION_CHANGE_GROUP);         -- Изменения прав
```

### Azure SQL — Ledger (Blockchain-based audit)

```sql
-- Ledger — tamper-evident таблицы (защита от изменения данных DBA)
CREATE TABLE [dbo].[Transactions] (
    TransactionId INT IDENTITY PRIMARY KEY,
    UserId INT NOT NULL,
    Amount DECIMAL(18,2) NOT NULL,
    TransactionDate DATETIME2 NOT NULL
)
WITH (SYSTEM_VERSIONING = ON, LEDGER = ON);

-- Каждая транзакция подписывается криптографически
-- Любое изменение обнаруживается
```

**Факт:** Azure SQL Ledger использует **Merkle tree** для верификации — можно проверить, что данные не были изменены даже администратором БД.

---

## Azure Storage — DLP

### Blob Storage — Immutable Storage

```csharp
// Immutable Blob — WORM (Write Once, Read Many)
// Защита от изменения/удаления данных

var immutabilityPolicy = new BlobImmutabilityPolicy
{
    PolicyMode = BlobImmutabilityPolicyMode.Locked,  // Нельзя снять политику
    RetentionPeriod = TimeSpan.FromDays(365)
};

await blobClient.SetImmutabilityPolicyAsync(immutabilityPolicy);

// Legal Hold — судебная блокировка (даже если retention policy expired)
await blobClient.SetLegalHoldAsync(hasLegalHold: true);
```

### Storage Firewall — ограничение доступа

```csharp
// Azure Storage Firewall — доступ только из specified VNet/IPs
// Включить: Settings → Networking → Selected networks
// Добавить: Azure SQL, App Service, Specific IP ranges

// Service Endpoint — прямой доступ из VNet без public internet
// Private Endpoint — полная изоляция из VNet
```

---

## Microsoft Defender for Cloud

**Факты:**
- **Cloud Security Posture Management (CSPM)** — оценка безопасности
- **Cloud Workload Protection (CWPP)** — защита workloads
- **DLP recommendations** — "Enable TDE on SQL Database", "Apply sensitivity labels to Storage"
- **File Integrity Monitoring** — изменения на VMs

```csharp
// Defender для Cloud — интеграция через SDK
public class SecurityAlertHandler
{
    public async Task HandleAlertAsync(SecurityAlert alert)
    {
        // Severity: Low, Medium, High, Critical
        if (alert.Severity == AlertSeverity.High)
        {
            await _notificationService.NotifySecOpsAsync(alert);
            await _autoResponder.TakeActionAsync(alert);
        }
    }
}
```

---

## Azure Policy — Governance

```json
{
  "policyRule": {
    "if": {
      "field": "Microsoft.Sql/servers/databases/transparentDataEncryption.status",
      "equals": "Disabled"
    },
    "then": {
      "effect": "deny"  // Запретить создание БД без TDE
    }
  }
}

// Azure Policy для DLP:
// - Все Storage Accounts должны иметь HTTPS-only
// - Все SQL DB должны иметь TDE включён
// - Все Key Vault должны иметь Soft Delete
// - Все Virtual Machines должны иметь Disk Encryption
```

---

## Data Residency — регионы данных

**Факты:**
- GDPR: данные резидентов EU должны храниться в EU
- Azure **Data Residency** — выбор региона при создании ресурса
- **Data at rest** — автоматически в выбранном регионе
- **Data in transit** — может выходить за регион (аутентификация, мониторинг)

```csharp
// Azure Policy — ограничение регионов
{
  "policyRule": {
    "if": {
      "field": "location",
      "notIn": ["westeurope", "northeurope"]
    },
    "then": {
      "effect": "deny"
    }
  }
}
```

---

## DLP в CI/CD — предотвращение утечек в процессе разработки

```yaml
# GitHub Actions — секреты и DLP
- name: Secret Scanning
  uses: github/codeql-action/upload-sarif@v3

- name: Detect secrets in code
  uses: Yelp/detect-secrets@v1
  with:
    args: --baseline .secrets.baseline

# .gitignore — не пушить sensitive
*.pfx
*.p12
*.key
secrets.*
appsettings.*.local.json
```

```xml
<!-- .gitattributes — защита от пуша sensitive файлов -->
*.pfx filter=lfs diff=lfs merge=lfs -text
secrets/** filter=git-crypt
```

---

## Чек-лист

- [ ] Microsoft Purview: Sensitivity Labels, автоматическая классификация
- [ ] M365 DLP: политики для Exchange, SharePoint, Teams
- [ ] Azure SQL: Audit, Ledger (tamper-evident)
- [ ] Azure Storage: Immutable Blob (WORM), Legal Hold
- [ ] Defender for Cloud: CSPM, CWPP, Security Alerts
- [ ] Azure Policy: restriction регионов, обязательный TDE
- [ ] Data Residency: регионы данных, GDPR
- [ ] CI/CD: secret scanning, .gitignore, git-crypt

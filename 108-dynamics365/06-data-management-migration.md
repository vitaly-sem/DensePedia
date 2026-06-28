# Управление данными и миграция в Dynamics 365

---

## Data Management Framework (DMF)

DMF — основной инструмент импорта/экспорта данных в D365 F&O:

```text
┌─────────────────────────────────────────────────────┐
│           Data Management Workspace                  │
│                                                     │
│  ┌──────────┐   ┌──────────┐   ┌───────────────┐   │
│  │ Import   │   │ Export   │   │ Data Projects │   │
│  │ Jobs     │   │ Jobs     │   │               │   │
│  └────┬─────┘   └────┬─────┘   └───────┬───────┘   │
│       │              │                 │            │
│       └──────────────┼─────────────────┘            │
│                      ▼                              │
│         ┌─────────────────────────┐                 │
│         │   Staging Tables         │                 │
│         │   (Промежуточные таблицы)│                 │
│         └───────────┬─────────────┘                 │
│                     │                               │
│         ┌───────────▼─────────────┐                 │
│         │   Target Tables          │                 │
│         │   (Целевые таблицы)      │                 │
│         └─────────────────────────┘                 │
└─────────────────────────────────────────────────────┘
```

### Источники данных для импорта/экспорта

| Источник | Формат | Применение |
|---|---|---|
| **Excel / CSV** | Файлы | Быстрая загрузка мастер-данных |
| **XML** | Файлы | Структурированные данные |
| **Azure Blob Storage** | Любой | Большие объёмы, автоматизация |
| **SharePoint** | Файлы | Совместная работа с данными |
| **OData** | REST | Интеграционная загрузка в реальном времени |
| **BYOD (Bring Your Own Database)** | SQL DB | Прямой экспорт в Azure SQL |
| **Data Lake / Synapse** | Parquet, CSV | Аналитические сценарии |

---

## Data Entities — единая точка интеграции

### Структура Data Entity

```xpp
// Composite Data Entity: объединяет несколько таблиц
[DataEntityAttribute]
public class SalesOrderHeaderEntity extends DataEntity
{
    SalesId         SalesOrderNumber;
    CustAccount     CustomerAccountNumber;
    SalesName       CustomerName;
    SalesStatus     SalesOrderStatus;
    DeliveryDate    RequestedDeliveryDate;
    CurrencyCode    CurrencyCode;

    // Computed field (не в таблице — вычисляется)
    [DataEntityField]
    public Amount displayTotalAmount()
    {
        return this.calculateTotalAmount();
    }
}
```

### Типы Data Entities

| Тип | Описание | Применение |
|---|---|---|
| **Simple Entity** | Одна таблица | Мастер-данные (Customers, Products) |
| **Composite Entity** | Несколько таблиц → одна структура | Документы (Sales Orders, Purchase Orders) |
| **Aggregate Entity** | Данные из нескольких источников (view-like) | Отчётность, аналитика |
| **Staging Entity** | С промежуточной таблицей и валидацией | Миграция с контролем качества |

### Стандартные Data Entities (из коробки)

| Категория | Ключевые Entities |
|---|---|
| **Customers** | Customers V3, Customer Groups, Customer Addresses |
| **Vendors** | Vendors V2, Vendor Groups, Vendor Bank Accounts |
| **Products** | Released Products V2, Products V2, Product Categories |
| **Chart of Accounts** | Main Accounts, Financial Dimensions, Account Structures |
| **Sales Orders** | Sales Order Headers V2, Sales Order Lines V2 |
| **Purchase Orders** | Purchase Order Headers V2, Purchase Order Lines V2 |
| **General Journal** | General Journal Entries, General Journal Account Entries |
| **Inventory** | Inventory On-Hand, Inventory Transactions, Inventory Sites |

---

## Стратегия миграции данных

### Фазы миграции

```text
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│ Extract  │ → │Transform │ → │Validate  │ → │  Load    │ → │Reconcile │
│ (Извлечь)│   │(Преобр.) │   │(Провер.) │   │(Загрузить)│   │(Сверить) │
└──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────────┘
```

| Фаза | Инструменты | Ключевые действия |
|---|---|---|
| **Extract** | SQL SSIS, Azure Data Factory, KingswaySoft | Извлечение из legacy-систем |
| **Transform** | ADF, SSIS, Power Query, X++ | Очистка, маппинг, нормализация |
| **Validate** | DMF Validation, X++ Scripts | Проверка referential integrity, mandatory fields |
| **Load** | DMF Import, OData, Custom X++ | Загрузка в правильном порядке (master → transactional) |
| **Reconcile** | DMF Compare, SQL Queries, Power BI | Сверка сумм, количества записей, балансов |

### Порядок загрузки данных (критичен!)

```text
1. Organization Structure (Legal Entities, Sites, Warehouses)
     ↓
2. Financial Foundation (Chart of Accounts, Currencies, Calendars)
     ↓
3. Reference Data (Tax Codes, Terms of Payment, Charge Codes)
     ↓
4. Master Data — Simple (Customer Groups, Vendor Groups, Product Categories)
     ↓
5. Master Data — Complex (Customers, Vendors, Products, BOMs)
     ↓
6. Open Balances (GL Balances, Open AR/AP, Inventory On-Hand)
     ↓
7. Open Transactions (Open SO, Open PO, Open Production Orders)
     ↓
8. Historical Data (если нужно)
```

> **Факт:** Самая частая причина провала миграции — неправильный порядок загрузки. Если загрузить Customers до Customer Groups — миграция упадёт на foreign key constraint. DMF сам проверяет referential integrity на staging-уровне.

---

## Примеры миграции данных

### Миграция через Excel (малые объёмы)

```text
Data Management Workspace
    → Import
        → Select Entity: Customers V3
        → Upload Excel file
        → Map fields (Source → Staging)
        → Validate
        → Copy to Target
```

### Миграция через DMF REST API (автоматизация)

```http
POST https://[instance].operations.dynamics.com/data/DataManagementDefinitionGroups/Microsoft.Dynamics.DataEntities.ImportFromPackage
Authorization: Bearer [token]
Content-Type: application/json

{
  "definitionGroupId": "CustomerMigration",
  "packageName": "Customers_v1.zip",
  "execute": true,
  "overwrite": true
}
```

### Массовая миграция через Azure Data Factory

```text
Legacy ERP DB (on-prem)
        │
        ▼  (Self-Hosted Integration Runtime)
┌────────────────────┐
│ Azure Data Factory │
│  ┌──────────────┐  │
│  │ Copy Activity │  │
│  │ + Mapping    │  │
│  └──────┬───────┘  │
└─────────┼──────────┘
          │
          ▼
  Azure Blob Storage
  (Staging: CSV/Parquet)
          │
          ▼
  D365 F&O (DMF REST API)
```

---

## BYOD (Bring Your Own Database)

Экспорт данных из D365 в собственную Azure SQL DB:

```text
D365 F&O Tables
        │
        ▼  (Incremental Push / Full Push)
┌──────────────────────┐
│   Your Azure SQL DB   │
│  ┌──────────────────┐ │
│  │ Sales Data        │ │
│  │ Inventory Data    │ │
│  │ Financial Data    │ │
│  └──────────────────┘ │
└──────────────────────┘
        │
        ▼
   Power BI / Azure Synapse / Custom Apps
```

| Характеристика | Описание |
|---|---|
| **Тип данных** | Детальные (не агрегированные) |
| **Режим обновления** | Full Push (все данные), Incremental Push (только изменения) |
| **Задержка** | Минуты-часы (не real-time) |
| **Назначение** | Аналитика, отчётность, интеграции |
| **Лимит** | До 5 BYOD инстансов на окружение F&O |

---

## Data Lake / Synapse Link

### Azure Synapse Link for Dataverse

```text
Dataverse (CRM data)
        │
        ▼  (Near real-time, без ETL)
┌──────────────────────────┐
│   Azure Data Lake Gen 2   │
│   (Delta Lake формат)     │
└───────────┬──────────────┘
            │
    ┌───────┴───────┐
    ▼               ▼
Azure Synapse    Power BI
(Spark, SQL)     (Direct Lake)
```

| Характеристика | Описание |
|---|---|
| **Синхронизация** | Near real-time (несколько минут) |
| **Формат** | Delta Lake (Parquet + транзакционный лог) |
| **Назначение** | Big Data аналитика, ML, AI |
| **Безопасность** | Наследует Dataverse security |

### Export to Data Lake (для F&O)

С 2024 года доступен прямой экспорт F&O данных в Data Lake:

```text
D365 F&O ──→ Azure Data Lake Gen2 ──→ Synapse / Fabric / Power BI
```

---

## Качество данных и очистка

### Типичные проблемы данных при миграции

| Проблема | Решение |
|---|---|
| **Дубликаты** | Dedup через Power Query, Fuzzy Grouping |
| **Неполные данные** | Default values в Data Entity, staging validation |
| **Несоответствие форматов** | Transform в ADF/Power Query перед загрузкой |
| **Отсутствующие reference данные** | Предварительная загрузка справочников |
| **Битые FK** | Staging Validation в DMF, pre-validation скрипты |
| **Кодировки** | UTF-8 везде, конвертация legacy ANSI |

### DMF Validation Framework

```text
Upload to Staging
        │
        ▼
Auto-validation:
  • Mandatory fields check
  • Data type check
  • Referential integrity
  • Business rules (если настроены)
        │
   ┌────┴────┐
   ▼         ▼
 Passed    Failed
   │         │
   ▼         ▼
Copy to   Error Log
Target    (Excel export)
```

---

## Инструменты миграции

| Инструмент | Назначение | Для кого |
|---|---|---|
| **DMF (встроенный)** | Базовый импорт/экспорт | Консультанты, аналитики |
| **Excel Add-in** | Быстрое редактирование данных | Пользователи, консультанты |
| **Azure Data Factory** | ETL для сложных сценариев | Data Engineers |
| **KingswaySoft** | SSIS-коннекторы для D365 | Data Engineers (.NET стек) |
| **Scribe Insight** | ETL для миграции (legacy) | Консультанты |
| **SSIS (SQL Server Integration Services)** | On-premises миграция | Data Engineers |
| **Power Query** | Трансформация данных | Консультанты, аналитики |
| **D365FO.Tools (PowerShell)** | Скриптовая миграция | DevOps, администраторы |

---

## Data Governance в D365

### Управление жизненным циклом данных

| Функция | Описание |
|---|---|
| **Data Retention Policies** | Автоматическое удаление устаревших данных |
| **Archive** | Архивация исторических данных (нативная или ISV) |
| **GDPR Compliance** | Поиск и удаление персональных данных |
| **Data Classification** | Метки конфиденциальности на полях |
| **Audit Trail** | Полный лог изменений данных |

### Best Practices

1. **Всегда используйте Data Entities** вместо прямого доступа к таблицам
2. **Тестируйте миграцию** на Sandbox с копией production данных
3. **Разбивайте большие миграции** на пакеты по 50K-100K записей
4. **Используйте Batch** для асинхронной загрузки больших объёмов
5. **Ведите log ошибок** и механизм повторных попыток
6. **Сверяйте balances** после каждой фазы миграции

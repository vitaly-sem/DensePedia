# Техническая архитектура Dynamics 365 F&O и разработка

---

## Язык X++

X++ — объектно-ориентированный язык, похожий на Java/C#, созданный специально для Microsoft Dynamics AX/F&O.

### Синтаксис и особенности

```xpp
// Объявление класса
[DataContract]
class SalesOrderService
{
    // Переменные экземпляра
    private SalesTable salesTable;
    private InventTable inventTable;

    // Метод с транзакцией
    [DataMember]
    public SalesId createSalesOrder(CustAccount _custAccount,
                                     ItemId _itemId,
                                     SalesQty _qty)
    {
        SalesId salesId;
        ttsBegin;  // Begin transaction

        salesTable.clear();
        salesTable.initValue();
        salesTable.CustAccount = _custAccount;
        salesTable.insert();

        // Создание строки
        SalesLine salesLine;
        salesLine.SalesId = salesTable.SalesId;
        salesLine.ItemId = _itemId;
        salesLine.SalesQty = _qty;
        salesLine.insert();

        ttsCommit;  // Commit transaction
        return salesTable.SalesId;
    }

    // Select с join
    public static SalesTable lookupOrder(SalesId _salesId)
    {
        SalesTable salesTable;
        select firstonly salesTable
            where salesTable.SalesId == _salesId
            && salesTable.SalesStatus == SalesStatus::Backorder;
        return salesTable;
    }
}
```

### Ключевые особенности X++

| Особенность | Описание |
|---|---|
| **ttsBegin/ttsCommit/ttsAbort** | Управление транзакциями (аналог SqlTransaction) |
| **select statement** | Встроенный в язык ORM (аналог LINQ to SQL) |
| **Table Inheritance** | Таблицы могут наследовать другие таблицы (с поддержкой FK) |
| **DataEntityView** | Поддержка views с computed полями |
| **Map** | Аналог интерфейса для таблиц (структурная типизация) |
| **Extensions** | Расширение любых артефактов (class, table, form) без изменения исходного кода |
| **CoC (Chain of Command)** | Декорирование методов через `next()` |
| **Attributes** | `[DataContract]`, `[DataMember]`, `[SysEntryPoint]` |

### Select Statement — встроенный ORM

```xpp
// Простой select
CustTable custTable;
select firstonly custTable
    where custTable.AccountNum == 'CUST-001';

// Join
SalesTable salesTable;
SalesLine salesLine;
while select salesTable
    join salesLine
    where salesLine.SalesId == salesTable.SalesId
{
    info(strFmt('%1 - %2', salesTable.SalesId, salesLine.ItemId));
}

// Агрегация
SalesLine salesLine;
select sum(QtyOrdered), sum(LineAmount)
    from salesLine
    where salesLine.SalesId == 'SO-0012345';
```

---

## Модель данных

### Таблицы (Tables)

```xpp
public class SalesTable extends Common
{
    SalesId         SalesId;        // Primary Key
    CustAccount     CustAccount;    // FK → CustTable
    SalesStatus     SalesStatus;    // Enum: Open, Confirmed, Delivered, Invoiced, Cancelled
    SalesName       SalesName;
    DeliveryDate    DeliveryDate;
    CurrencyCode    CurrencyCode;
}

// Отношения (Relations)
// На SalesTable:
// - CustTable (ForeignKey: CustAccount → CustTable.AccountNum)
// - SalesLine (Cascade: SalesId → SalesLine.SalesId)
```

### Типы таблиц

| Тип | Описание |
|---|---|
| **Regular** | Обычная таблица с данными |
| **TempDB** | Временная таблица в tempdb (сессия/запрос) |
| **InMemory** | Таблица в памяти (не пишется в БД) |
| **Worksheet** | Для временного хранения при пакетной обработке |

### Data Entities

Data Entities — абстрактный слой над таблицами для интеграций:

```text
                    ┌─────────────────┐
                    │   Data Entity    │  ← Единая точка интеграции
                    │  (SalesOrder)    │
                    └────────┬────────┘
                             │
         ┌───────────────────┼───────────────────┐
         ▼                   ▼                   ▼
   ┌──────────┐       ┌──────────┐       ┌──────────┐
   │SalesTable│       │SalesLine │       │CustTable │
   └──────────┘       └──────────┘       └──────────┘
```

| Тип Data Entity | Описание |
|---|---|
| **Composite Entity** | Объединяет несколько таблиц в одну структуру |
| **Aggregate Entity** | Данные из нескольких источников (view-like) |
| **Staging Entity** | Для импорта через DMF с валидацией |

---

## MorphX — среда разработки

MorphX — IDE, встроенная прямо в D365 F&O клиент:

| Инструмент MorphX | Назначение |
|---|---|
| **AOT (Application Object Tree)** | Дерево всех объектов приложения |
| **Table Browser** | Просмотр и редактирование данных таблиц |
| **Code Editor** | Редактор X++ с IntelliSense |
| **Debugger** | Встроенный отладчик (breakpoints, watch, call stack) |
| **Form Designer** | Drag-and-drop дизайнер форм |
| **Report Designer** | Дизайнер SSRS-отчётов |
| **Best Practice Checker** | Статический анализ кода |
| **Cross-reference** | Поиск использования объектов по всей системе |

> **Факт:** С 2020 года Microsoft активно переводит разработку на Visual Studio + Azure DevOps. Новые инструменты: X++ extension в VS Code, DevOps build pipelines. MorphX остаётся для отладки и быстрых правок в production.

---

## Модель расширений (Extensions)

Расширения позволяют модифицировать систему без изменения базового кода (overlay vs extensions):

### Chain of Command (CoC)

```xpp
[ExtensionOf(tableStr(SalesTable))]
final class SalesTable_MyExtension
{
    public void insert()
    {
        // Код до оригинального метода
        this.MyPreInsertValidation();

        next insert();  // Вызов оригинального метода

        // Код после оригинального метода
        this.MyPostInsertLogic();
    }
}
```

| Тип расширения | Что можно расширять |
|---|---|
| **Class Extension** | Добавление методов, полей, событий |
| **Table Extension** | Новые поля, индексы, relations, методы |
| **Form Extension** | Новые контролы, data sources, методы форм |
| **Enum Extension** | Новые значения enum |

### Event Handlers

```xpp
// Pre-event handler
[PreHandlerFor(tableStr(SalesTable), tableMethodStr(SalesTable, insert))]
public static void SalesTable_Pre_insert(XppPrePostArgs _args)
{
    SalesTable salesTable = _args.getThis();
    // Логика до insert
}

// Post-event handler
[PostHandlerFor(tableStr(SalesTable), tableMethodStr(SalesTable, insert))]
public static void SalesTable_Post_insert(XppPrePostArgs _args)
{
    SalesTable salesTable = _args.getThis();
    // Логика после insert
}
```

---

## CI/CD и Azure DevOps

### Сборка пакетов

```yaml
# Azure DevOps Pipeline (azure-pipelines.yml)
trigger:
- main

pool:
  vmImage: 'windows-latest'

steps:
- task: Dynamics365BuildTools@1
  inputs:
    command: 'Build'
    projects: '**/*.rnrproj'
    outputPath: '$(Build.ArtifactStagingDirectory)/DeployablePackages'

- task: PublishBuildArtifacts@1
  inputs:
    pathToPublish: '$(Build.ArtifactStagingDirectory)'
    artifactName: 'DeployablePackage'
```

### Deployable Package

Структура Deployable Package (развёртываемый пакет):

```text
DeployablePackage.zip
├── AOSService/
│   ├── bin/            # Скомпилированные DLL (X++ → CIL)
│   ├── Packages/       # Исходный код в виде .axpp (архив X++ файлов)
│   └── Schema/         # Синхронизация схемы БД
├── manifests/          # Манифесты пакета и версии
└── resources/          # Дополнительные ресурсы
```

---

## Среда выполнения

### Компиляция X++

```text
X++ Source Code (.xpp)
        │
        ▼
   P-Code (байт-код) — интерпретируется
        │
        ▼
   CIL (Common Intermediate Language) — .NET IL
        │
        ▼
   Native Code (JIT) — исполняется CLR
```

| Слой | Описание |
|---|---|
| **P-Code** | Байт-код для интерпретации (используется в dev-сценариях) |
| **CIL** | Компиляция в .NET Intermediate Language (production) |
| **Build** | Автоматическая CIL-компиляция при деплое |

### AOS (Application Object Server)

| Компонент AOS | Функция |
|---|---|
| **Session Management** | Управление пользовательскими сессиями |
| **Caching** | Global, Session, User, Company-level кэши |
| **Batch Framework** | Асинхронное выполнение задач по расписанию |
| **Security** | Role-based access, проверка привилегий |
| **Telemetry** | Сбор метрик, ошибок, производительности |
| **DB Connection Pool** | Управление соединениями с SQL Server |

---

## Dual-write — синхронизация с Dataverse

Dual-write обеспечивает синхронизацию в реальном времени между F&O и Dataverse (CRM/Power Platform):

```text
┌──────────────────┐         ┌──────────────────┐
│   D365 F&O        │ ◄─────► │    Dataverse      │
│  (ERP Database)   │  Dual-  │  (CRM Database)   │
│                   │  write  │                   │
├──────────────────┤         ├──────────────────┤
│ Customers         │ ◄─────► │ Accounts          │
│ Products          │ ◄─────► │ Products          │
│ Sales Orders      │ ◄─────► │ Sales Orders      │
│ Vendors           │ ◄─────► │ Contacts          │
│ Warehouses        │ ◄─────► │ custom entities   │
└──────────────────┘         └──────────────────┘
```

| Характеристика | Описание |
|---|---|
| **Задержка** | Near real-time (секунды) |
| **Направление** | Двусторонняя синхронизация |
| **Разрешение конфликтов** | Last-write-wins, custom resolution |
| **Настройка** | Выбор entity mapping через UI |
| **Мониторинг** | Dashboard ошибок синхронизации |

---

## Интеграции через REST API (OData)

### Стандартная OData-интеграция

```http
GET https://[instance].operations.dynamics.com/data/SalesOrdersV2
Authorization: Bearer [token]
Content-Type: application/json
```

### C# клиент для OData

```csharp
var context = new Resources(new Uri("https://[instance].operations.dynamics.com/data/"));
context.SendingRequest2 += (s, e) =>
{
    e.RequestMessage.SetHeader("Authorization", $"Bearer {token}");
};

var salesOrders = context.SalesOrdersV2
    .Where(so => so.SalesOrderStatus == "Open")
    .Take(100)
    .ToList();
```

### Custom Services (X++)

```xpp
[SysEntryPoint(true)]
public class MyIntegrationService
{
    [HttpPost]
    public container processOrder(container _orderData)
    {
        // Десериализация и обработка заказа
        return [true, "Order processed successfully"];
    }
}
```

---

## Инструменты администрирования

| Инструмент | Назначение |
|---|---|
| **Lifecycle Services (LCS)** | Управление средами, обновления, мониторинг |
| **System Administration** | Встроенный модуль администрирования |
| **SQL Insights** | Мониторинг производительности запросов |
| **Environment Monitoring** | LCS dashboard с метриками и алертами |
| **Azure Application Insights** | Детальная телеметрия (если настроена) |
| **Trace Parser** | Анализ трассировки производительности |
| **Performance SDK** | Нагрузочное тестирование (MSDyn365FOPerformanceSDK) |

---

## Особенности разработки

### Нумерация (Number Sequences)

```xpp
NumberSeq numSeq = NumberSeq::newGetNum(SalesParameters::numRefSalesId());
SalesId = numSeq.salesId();
salesTable.SalesId = SalesId;
```

### Batch Processing

```xpp
public void run()
{
    // Код пакетной обработки
    SalesTable salesTable;
    while select salesTable
        where salesTable.SalesStatus == SalesStatus::Open
    {
        this.processOrder(salesTable);
    }
}

// Запуск в batch
public static void main(Args _args)
{
    MyBatchClass batch = new MyBatchClass();
    BatchHeader batchHeader = BatchHeader::construct();
    batchHeader.addTask(batch);
    batchHeader.save();
}
```

### SysOperation Framework

Фреймворк для асинхронных операций с поддержкой UI и batch:

```xpp
class MyExportOperation extends SysOperationServiceBase
{
    public void exportData(MyExportContract _contract)
    {
        // Длительная операция
    }
}
```

---

## Лицензионные и правовые аспекты

| Аспект | Описание |
|---|---|
| **ISV Licensing** | Партнёрские решения (ISV) сертифицируются Microsoft и продаются через AppSource |
| **AppSource** | Маркетплейс решений для Dynamics 365 |
| **Embedded licenses** | Лицензии для встраивания D365 в партнёрские SaaS-решения |
| **Customer Self-Service** | Клиенты могут создавать собственные расширения без ISV-сертификации |
| **Database Access** | Прямой доступ к данным (BYOD, Entity Store) разрешён; доступ к SQL напрямую — нет для SaaS |

---

## Полезные инструменты

| Инструмент | Назначение |
|---|---|
| **D365FO.Tools** | PowerShell-модуль для администрирования |
| **D365FOAdminToolKit** | Набор утилит администратора |
| **ATL (Acceptance Test Library)** | Фреймворк для автотестов на X++ |
| **SysTest** | Встроенный фреймворк модульного тестирования |
| **Tusk** | Инструмент управления кодом/задачами внутри MorphX |
| **D365 FO ISV DevOps** | Шаблоны Azure DevOps для ISV-решений |

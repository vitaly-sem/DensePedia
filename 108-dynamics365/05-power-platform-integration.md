# Power Platform и интеграция с Dynamics 365

---

## Обзор Power Platform

Power Platform — low-code/no-code платформа Microsoft, неразрывно связанная с Dynamics 365:

```text
┌──────────────────────────────────────────────────────────────┐
│                      POWER PLATFORM                           │
│                                                              │
│  ┌───────────┐ ┌──────────────┐ ┌─────────┐ ┌───────────┐  │
│  │ Power Apps│ │Power Automate│ │Power BI │ │Power Pages│  │
│  │  (Apps)   │ │  (Workflow)  │ │  (BI)   │ │ (Portals) │  │
│  └─────┬─────┘ └──────┬───────┘ └────┬────┘ └─────┬─────┘  │
│        │              │             │             │         │
│        └──────────────┼─────────────┼─────────────┘         │
│                       ▼             ▼                        │
│               ┌──────────────────────────┐                   │
│               │        DATAVERSE          │                   │
│               │   (Common Data Service)   │                   │
│               └────────────┬─────────────┘                   │
└────────────────────────────┼─────────────────────────────────┘
                             │
         ┌───────────────────┼───────────────────┐
         ▼                   ▼                   ▼
┌─────────────────┐ ┌──────────────┐ ┌──────────────────┐
│   D365 CRM      │ │  D365 F&O    │ │  External Data   │
│ (Sales, CS, FS) │ │  (ERP DB)    │ │  (SQL, API, ...) │
└─────────────────┘ └──────────────┘ └──────────────────┘
```

---

## Dataverse — ядро данных

Dataverse (бывший Common Data Service / CDS) — облачная база данных с бизнес-логикой:

| Характеристика | Описание |
|---|---|
| **Тип** | Мультиарендная облачная БД (Azure SQL + Blob + Cosmos DB под капотом) |
| **Сущности** | Стандартные (Account, Contact, ...) и кастомные |
| **Безопасность** | Role-Based, Record-Level, Field-Level |
| **Бизнес-логика** | Business Rules, Workflows, Plugins, Power Automate |
| **API** | OData, Web API, FetchXML, SQL (TDS endpoint) |
| **Лимиты** | 4 TB на инстанс, 10 000 операций/сек (стандарт) |

### Dataverse vs SQL Server (для разработчика)

| Критерий | Dataverse | SQL Server |
|---|---|---|
| **Транзакции** | Plugin pipeline, concurrency control | ACID, explicit transactions |
| **Индексы** | Автоматические + managed | Ручное управление |
| **Схема** | Автоматическая, миграции через solutions | DDL-скрипты, миграции |
| **Аудит** | Встроенный на уровне полей | Ручная настройка |
| **Логика** | Plugins (C#), Formulas (Power Fx), Business Rules | Stored Procedures, Triggers |

---

## Power Apps — создание приложений

### Model-Driven Apps

Строятся на модели данных Dataverse. Используются для внутренних бизнес-приложений:

```text
Data Model (Dataverse) → Site Map → Forms → Views → Charts → Dashboards
```

| Особенность | Описание |
|---|---|
| **Data-first** | Приложение «вырастает» из модели данных |
| **Адаптивность** | Автоматически адаптируется под десктоп/мобильные |
| **Стандартные компоненты** | Forms, Views, Charts, Timeline, Business Process Flow |
| **Custom Pages** | Canvas-страницы внутри model-driven приложений |

### Canvas Apps

Pixel-perfect дизайн, похоже на PowerPoint:

```powerfx
// Power Fx — формульный язык (похож на Excel)
If(
    IsBlank(ComboBox_Account.Selected),
    Notify("Выберите организацию", NotificationType.Warning),
    Patch(
        Opportunities,
        Defaults(Opportunities),
        {
            Name: TextInput_Name.Text,
            'Account': ComboBox_Account.Selected,
            EstimatedCloseDate: DatePicker_CloseDate.SelectedDate,
            EstimatedValue: Value(TextInput_Value.Text)
        }
    )
)
```

| Тип приложения | Когда использовать |
|---|---|
| **Model-Driven** | Сложные данные, бизнес-логика, связанные таблицы |
| **Canvas** | Точный контроль интерфейса, мобильные устройства, простые данные |
| **Custom Page** | Комбинация: canvas-дизайн внутри model-driven |

---

## Power Automate — автоматизация процессов

### Типы потоков (Flows)

| Тип | Триггер | Применение |
|---|---|---|
| **Automated** | Событие (запись создана/изменена) | Автоматический workflow |
| **Instant** | Ручной запуск (кнопка) | Утилиты, разовые операции |
| **Scheduled** | По расписанию | Ночная синхронизация, отчёты |
| **Business Process Flow** | Пошаговый процесс | Поэтапный бизнес-процесс (stage-gate) |
| **Desktop Flow (RPA)** | UI-автоматизация | Автоматизация legacy desktop приложений |

### Пример: утверждение заказа на закупку

```text
D365 F&O: Purchase Order Created
        │
        ▼
Trigger: When a record is created (Dataverse / F&O connector)
        │
        ▼
Get Manager of the requestor (Office 365 Users connector)
        │
        ▼
Send Approval Email (Outlook / Teams)
        │
        ├── Approved ──→ Update PO Status in D365 F&O
        │                Send Notification to Requestor
        │
        └── Rejected ──→ Update PO Status, Notify with Reason
```

### Коннекторы для Dynamics 365

| Коннектор | Для |
|---|---|
| **Dynamics 365 (Dataverse)** | CRM-приложения (Sales, Customer Service) |
| **Dynamics 365 Finance & Operations** | Прямые вызовы DMF, OData, SOAP |
| **Dynamics 365 Business Central** | OData API, SOAP |
| **Common Data Service (legacy)** | Старые интеграции с CDS |

---

## Dual-write — живая синхронизация F&O ↔ Dataverse

### Принцип работы

```text
┌──────────────────────┐          ┌──────────────────────┐
│      D365 F&O        │          │      Dataverse        │
│                      │          │                      │
│  SalesTable          │◄────────►│  salesorder          │
│  SalesLine           │◄────────►│  salesorderdetail    │
│  CustTable           │◄────────►│  account             │
│  InventTable         │◄────────►│  product             │
│  VendTable           │◄────────►│  contact             │
│  InventSite          │◄────────►│  msdyn_warehouse     │
│                      │          │                      │
└──────────────────────┘          └──────────────────────┘
         ▲                                ▲
         │                                │
         └────────────┬───────────────────┘
                      │
              Dual-write Runtime
           (Azure Service Fabric)
```

| Характеристика | Описание |
|---|---|
| **Направление** | Двустороннее (Bi-directional) |
| **Задержка** | Секунды (near real-time) |
| **Инфраструктура** | Azure Service Fabric |
| **Маппинг** | Entity Maps — предопределённые пары сущностей |
| **Трансформации** | Настраиваемые трансформации значений |
| **Конфликты** | Last-write-wins (по умолчанию), custom resolution |
| **Мониторинг** | Dual-write dashboard с логами ошибок |

### Настройка Dual-write

1. Установить Dual-write Orchestration Solution в Dataverse
2. Связать окружения F&O и Dataverse
3. Выбрать Entity Maps из каталога
4. Запустить Initial Sync (массовая синхронизация)
5. Live sync запускается автоматически

```text
Initial Sync (один раз)          Live Sync (постоянно)
─────────────────────           ─────────────────────
Копирует все записи             Синхронизирует изменения
из F&O в Dataverse              в реальном времени
(и наоборот)                    через Change Tracking
```

---

## Power BI — аналитика

### Режимы интеграции с D365

| Режим | Описание |
|---|---|
| **Power BI Embedded** | Дашборды прямо внутри D365 workspace'ов |
| **Entity Store** | Агрегированные F&O данные во встроенном columnstore |
| **BYOD + Azure Synapse** | Детальные данные во внешней БД → Power BI |
| **DirectQuery** | Прямые запросы к OData endpoint D365 |
| **Dataverse Connector** | Подключение к CRM-данным через Dataverse |

### Entity Store

```text
D365 F&O OLTP (SQL Server)
        │
        ▼  (Refresh Entity Store — nightly batch)
┌──────────────────────────┐
│      Entity Store         │
│  (In-Memory Columnstore)  │
│  ┌────────┐ ┌───────────┐ │
│  │GL Data │ │Sales Data │ │
│  ├────────┤ ├───────────┤ │
│  │Inv Data│ │PO Data    │ │
│  └────────┘ └───────────┘ │
└───────────┬──────────────┘
            │
            ▼
        Power BI
```

| Характеристика | Описание |
|---|---|
| **Структура** | In-memory columnstore (быстрые агрегации) |
| **Обновление** | Пакетное (ночное), до 30 агрегатных мер |
| **Объём** | До 500 млн строк на measure |
| **Назначение** | Быстрые аналитические отчёты без нагрузки на OLTP |

---

## Power Pages — внешние порталы

Порталы для клиентов, поставщиков, партнёров:

| Сценарий | Пример |
|---|---|
| **Customer Self-Service** | Просмотр заказов, счетов, баланса |
| **Vendor Portal** | Подача заявок, просмотр заказов |
| **Community** | Форумы, база знаний, тикеты |
| **Partner Portal** | Дилерский портал, совместные продажи |

```text
                    ┌────────────────┐
                    │  Power Pages    │
                    │  (Public URL)   │
                    └───────┬────────┘
                            │
                  ┌─────────▼─────────┐
                  │     Dataverse      │
                  │  (CRM + F&O data)  │
                  └───────────────────┘
```

---

## AI Builder и Copilot

### AI Builder

Интеграция AI без написания кода:

| Модель | Применение |
|---|---|
| **Form Processing** | Извлечение данных из счетов, накладных |
| **Object Detection** | Распознавание объектов на фото (склад, полевое обслуживание) |
| **Text Classification** | Категоризация обращений, определение тональности |
| **Prediction** | Предсказание оттока, скоринг лидов |

### Microsoft Copilot в Dynamics 365

| Приложение | Copilot-сценарии |
|---|---|
| **Sales** | Суммаризация сделок, генерация писем, анализ встреч в Teams |
| **Customer Service** | Автоответы, draft emails, knowledge base поиск |
| **Finance** | Анализ вариаций бюджета, NLP-запросы к данным |
| **Supply Chain** | Copilot в планировании закупок, анализ рисков поставок |
| **Business Central** | Автосоздание описаний товаров, маркетинговых текстов |

---

## Безопасность и управление

### Power Platform Admin Center

| Инструмент | Назначение |
|---|---|
| **Environments** | Управление средами (Dev/Test/Prod) |
| **DLP Policies** | Контроль использования коннекторов |
| **Capacity Management** | Мониторинг лимитов и потребления |
| **Analytics** | Отчёты по использованию платформы |
| **Tenant Isolation** | Ограничение кросс-тенантных подключений |

> **Факт:** Power Platform использует тот же Entra ID (Azure AD) для аутентификации, что и Dynamics 365. Это обеспечивает сквозной SSO и единый security model.

---

## Сравнение сценариев расширения

| Сценарий | Инструмент | Когда использовать |
|---|---|---|
| **Пользовательский UI** | Power Apps (Canvas) | Мобильные приложения, простые формы |
| **Сложная бизнес-логика** | X++ / AL / C# Plugins | Высокопроизводительные вычисления |
| **Workflow / Approval** | Power Automate | Автоматизация утверждений |
| **Внешний портал** | Power Pages | Клиентский портал |
| **Аналитика** | Power BI | Дашборды, отчёты |
| **AI-сценарии** | AI Builder / Copilot | Извлечение данных, предсказания |
| **RPA** | Power Automate Desktop | Автоматизация legacy-приложений |

# Dynamics 365 Finance & Operations — глубокое погружение

---

## Архитектура F&O

Dynamics 365 Finance & Operations имеет трёхслойную архитектуру:

```text
┌─────────────────────────────────────────────────────┐
│                PRESENTATION LAYER                   │
│  Web Client (HTML5) | Mobile App | Office Add-ins   │
├─────────────────────────────────────────────────────┤
│                 APPLICATION LAYER                   │
│  AOS (Application Object Server) | IIS | .NET       │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ │
│  │ Finance      │ │ Supply Chain │ │ Commerce     │ │
│  │ Modules      │ │ Modules      │ │ Modules      │ │
│  └──────────────┘ └──────────────┘ └──────────────┘ │
├─────────────────────────────────────────────────────┤
│                   DATA LAYER                        │
│  SQL Server / Azure SQL DB | Entity Store | BYOD    │
└─────────────────────────────────────────────────────┘
```

### Ключевые компоненты

| Компонент | Описание |
|---|---|
| **AOS (Application Object Server)** | Сервер приложений, исполняет X++ бизнес-логику |
| **SQL Server / Azure SQL DB** | Единая база данных для всех модулей (без разделения на OLTP/OLAP) |
| **Entity Store** | Агрегированные данные для аналитики внутри D365 |
| **BYOD (Bring Your Own Database)** | Экспорт данных во внешние Azure SQL DB |
| **Batch Framework** | Асинхронное выполнение пакетных заданий |
| **OData / REST API** | Внешние интеграции и мобильные приложения |

---

## Модуль Finance — финансовое ядро

### General Ledger (Главная книга)

```text
Chart of Accounts → Legal Entity → Business Unit → Department → Cost Center
         │
         ├── Main Accounts (Balance Sheet, P&L, Revenue, Expense)
         ├── Financial Dimensions (до 50+ аналитик одновременно)
         ├── Posting Profiles (автоматическая разметка проводок)
         └── Financial Periods (Fiscal Year, Open/Close, Adjustments)
```

| Концепция | Описание |
|---|---|
| **Chart of Accounts** | Иерархический план счетов, общий для всех компаний или специфичный |
| **Financial Dimensions** | Кастомизируемые аналитики (Department, CostCenter, Project, Region...) |
| **Dimension Sets** | Предопределённые комбинации аналитик для отчётов |
| **Ledger Settlement** | Автоматический зачёт дебета и кредита в главной книге |
| **Intercompany Accounting** | Автоматическое зеркалирование проводок между юрлицами |

### Accounts Payable (Кредиторская задолженность)

| Функция | Описание |
|---|---|
| **Vendor Master** | Карточка поставщика, банковские реквизиты, налоговые профили |
| **Purchase Orders** | Трёхстороннее сопоставление (PO → Receipt → Invoice) |
| **Invoice Register** | Журнал входящих счетов-фактур |
| **Payment Journal** | Платёжные поручения, поддержка SEPA, ACH, SWIFT |
| **Vendor Collaboration** | Портал самообслуживания для поставщиков |
| **1099 Processing** | Налоговая отчётность для США |

### Accounts Receivable (Дебиторская задолженность)

| Функция | Описание |
|---|---|
| **Customer Master** | Карточка клиента, кредитные лимиты, условия оплаты |
| **Sales Orders** | Полный цикл: Quotation → SO → Picking List → Packing Slip → Invoice |
| **Collections Management** | Автоматическая работа с просроченной задолженностью |
| **Credit Management** | Блокировка отгрузки при превышении кредитного лимита |
| **Revenue Recognition** | ASC 606 / IFRS 15 — признание выручки по этапам |

### Budgeting (Бюджетирование)

| Функция | Описание |
|---|---|
| **Budget Planning** | Планирование бюджетов через Excel-шаблоны и workflow |
| **Budget Control** | Автоматический контроль бюджета при создании документов |
| **Budget Register Entries** | Журнал бюджетных проводок |
| **Position Budgeting** | Бюджетирование штатного расписания |

### Fixed Assets (Основные средства)

| Функция | Описание |
|---|---|
| **Acquisition** | Приобретение ОС — из PO, ручной ввод |
| **Depreciation** | 20+ методов амортизации (Straight-line, Declining Balance, MACRS...) |
| **Disposal** | Выбытие, продажа, списание |
| **Fixed Asset Groups** | Группировка ОС по налоговым/бухгалтерским правилам |
| **Revaluation** | Переоценка ОС (справедливая стоимость) |

### Cash & Bank Management

| Функция | Описание |
|---|---|
| **Bank Accounts** | Управление банковскими счетами в разных валютах |
| **Bank Reconciliation** | Сверка банковских выписок (ручная + Advanced Bank Reconciliation) |
| **Letter of Credit / Guarantee** | Аккредитивы и банковские гарантии |
| **Cash Flow Forecasting** | Прогнозирование денежных потоков на основе заказов/договоров |

### Tax (Налоги)

| Функция | Описание |
|---|---|
| **Sales Tax** | Налог с продаж (США) — расчёт, отчётность |
| **VAT** | НДС — начисление, возмещение, декларации |
| **Withholding Tax** | Налог у источника |
| **Tax Calculation Service** | Внешний микросервис расчёта налогов для сложных юрисдикций (Бразилия, Индия) |
| **Electronic Reporting** | Единый механизм форматирования электронных отчётов |

> **Факт:** Tax Calculation Service — отдельный Azure-сервис, дополняющий встроенную логику. Используется для стран со сверхсложным налогообложением (Бразилия — ICMS, IPI, PIS, COFINS, ISS; Индия — GST с SGST, CGST, IGST).

---

## Модуль Supply Chain Management

### Inventory Management

```text
                    ┌───────────────┐
                    │  Item Master  │
                    └───────┬───────┘
                            │
          ┌─────────────────┼─────────────────┐
          ▼                 ▼                   ▼
   ┌──────────────┐ ┌──────────────┐ ┌──────────────────┐
   │  Purchased   │ │ Manufactured │ │  Service Items   │
   │   Items      │ │    Items     │ │                  │
   └──────┬───────┘ └──────┬───────┘ └────────┬─────────┘
          │                │                   │
          ▼                ▼                   ▼
   ┌──────────────────────────────────────────────────────┐
   │              Inventory Dimensions                     │
   │  Site → Warehouse → Location → Pallet → Batch/Serial │
   └──────────────────────────────────────────────────────┘
```

| Функция | Описание |
|---|---|
| **Item Model Groups** | Определяют складскую модель (FIFO, LIFO, Weighted Avg, Standard Cost) |
| **Storage Dimensions** | Site → Warehouse → Location (3 уровня) |
| **Tracking Dimensions** | Batch Number, Serial Number |
| **Quality Management** | Контроль качества на приёмке и отгрузке |
| **Quarantine** | Блокировка партий до проверки качества |
| **Catch Weight** | Учёт веса для товаров с переменным весом (мясо, рыба, сыр) |

### Warehouse Management (WHS)

| Функция | Описание |
|---|---|
| **Warehouse Mobile App** | Терминал сбора данных (ТСД), работа через браузер |
| **Wave Processing** | Группировка работ по складу в волны для оптимизации сборки |
| **Work Templates** | Настройка шагов складских операций (Pick → Put, Move) |
| **Location Directives** | Правила поиска мест хранения при приёмке/отгрузке |
| **Container Packing** | Упаковка в контейнеры с контролем веса/объёма |
| **Cycle Counting** | Циклическая инвентаризация (по зонам, по товарам, по порогу) |
| **Cross-docking** | Прямая перегрузка с приёмки в отгрузку |
| **License Plate** | Уникальный идентификатор паллеты/контейнера |
| **FEFO/FIFO Picking** | Приоритетная сборка по сроку годности/дате прихода |

### Procurement & Sourcing

| Функция | Описание |
|---|---|
| **Purchase Requisitions** | Заявки на закупку с workflow утверждения |
| **RFQ (Request for Quotation)** | Запрос предложений от поставщиков |
| **Purchase Agreements** | Долгосрочные договоры: Purchase Agreement, Trade Agreement |
| **Vendor Rebates** | Расчёт бонусов и скидок от поставщиков |
| **Procurement Categories** | Каталогизация товаров/услуг |
| **Change Management** | Версионирование и утверждение изменений заказов |

### Master Planning (MRP)

| Параметр | Описание |
|---|---|
| **Master Schedule** | Главный производственный план (MPS) |
| **MRP Run** | Расчёт чистых потребностей (Net Requirements) |
| **Coverage Groups** | Правила покрытия: Period, Requirement, Min/Max |
| **Safety Stock** | Страховой запас (фиксированный, динамический по спросу) |
| **Action Messages** | Рекомендации системы: создать/отменить/перенести заказ |
| **Forecast Plans** | Планирование на основе прогнозов спроса |
| **Intercompany Planning** | Сквозное планирование между юрлицами |

> **Факт:** Master Planning в D365 использует оптимизационный движок **Planning Optimization** (с 2021 года) — облачный микросервис, заменивший встроенный MRP-движок. Он работает в памяти (in-memory) и может обрабатывать миллионы транзакций за минуты.

### Manufacturing (Производство)

| Функция | Описание |
|---|---|
| **BOM (Bill of Materials)** | Спецификации — дискретные, фантомные, с формулами |
| **Routes (Маршруты)** | Производственные маршруты с операциями и временами |
| **Production Orders** | Дискретное производство (штучное) |
| **Batch Orders** | Партионное производство (формульное, химия, пища) |
| **Lean Manufacturing** | Канбан-доски, вытягивающее производство |
| **Process Manufacturing** | Побочные продукты, co-products, формулы с yield |
| **Subcontracting** | Передача операций на субподряд |
| **Shop Floor Control** | Регистрация выработки с терминалов (MES) |
| **Resource Scheduling** | Планирование загрузки оборудования и персонала |

### Transportation & Landed Cost

| Функция | Описание |
|---|---|
| **Transportation Management (TMS)** | Планирование рейсов, выбор перевозчика, расчёт тарифов |
| **Landed Cost** | Калькуляция полной стоимости импорта (фрахт, пошлина, страховка) |
| **Route Planning** | Последовательность доставки (Hub → Spoke → Final) |
| **Freight Reconciliation** | Сверка счетов перевозчиков |

---

## Глобальные возможности (Globalization)

### Multi-Legal Entity

D365 F&O нативно поддерживает множество юридических лиц в одной инсталляции:

```text
Holding Company (US)
├── Legal Entity: US Trading LLC
├── Legal Entity: EU Trading GmbH (Germany)
├── Legal Entity: APAC Trading KK (Japan)
└── Legal Entity: Manufacturing CN (China)
```

| Функция | Описание |
|---|---|
| **Shared Chart of Accounts** | Единый план счетов или разные для юрлиц |
| **Intercompany Trade** | Автоматические межфирменные продажи (SO → IC PO) |
| **Centralized Payments** | Платёжный центр (одно юрлицо платит за всех) |
| **Consolidation** | Консолидация отчётности с элиминацией |
| **Currency Translation** | Пересчёт валют для консолидации |

### Локализация

| Регион | Особенности |
|---|---|
| **Россия** | Книги покупок/продаж, НДС, счета-фактуры, бух. проводки по РСБУ |
| **Европа** | Intrastat, EU Sales List, SEPA, VAT Reporting |
| **США** | 1099, Sales Tax (Avalara интеграция), GAAP |
| **Китай** | Golden Tax, китайские основные средства, локализованная главная книга |
| **Бразилия** | Nota Fiscal Eletrônica (NF-e), SPED, ICMS/IPI/COFINS/PIS |
| **Индия** | GST (SGST + CGST + IGST), TDS/TCS, e-Invoice, e-Way Bill |
| **Япония** | Japanese Fixed Assets, Consolidated Invoice |

### Electronic Reporting (ER)

ER — фреймворк для создания любых электронных отчётов без кода:

```text
Data Model → Model Mapping → Format (Excel, XML, TXT, JSON)
     │              │               │
     ▼              ▼               ▼
  Абстрактная   Привязка к       Выходной формат
  схема данных  таблицам D365    (шаблон Excel/конфиг)
```

> **Факт:** Electronic Reporting был представлен в 2015 году как замена SSRS-отчётов и legacy X++ отчётности. Позволяет консультантам (не разработчикам) создавать электронные отчёты под требования регуляторов.

---

## Финансовая отчётность (Financial Reporting)

### Management Reporter / Financial Reporter

| Функция | Описание |
|---|---|
| **Row Definition** | Строки отчёта (счета, группы счетов, формулы) |
| **Column Definition** | Колонки (периоды, бюджеты, сравнения) |
| **Reporting Tree** | Иерархия юрлиц/подразделений для консолидации |
| **Output** | Excel, HTML, PDF |

### Аналитика и BI

| Инструмент | Описание |
|---|---|
| **Power BI Embedded** | Встроенные дашборды прямо в D365 workspace |
| **Entity Store** | Агрегированные данные (in-memory) для быстрых отчётов |
| **BYOD + Azure Synapse** | Экспорт детальных данных в Azure для аналитики |
| **Microsoft Fabric** | Единая аналитическая платформа (новинка 2024) |
| **Financial Insights** | AI-driven прогнозы cash flow, предсказание оплат |

---

## Workflow и Approvals

D365 F&O использует встроенный workflow-движок:

| Возможность | Описание |
|---|---|
| **Approval Workflows** | Утверждение PO, Invoices, Expenses, Budgets, Journals |
| **Workflow Editor** | Визуальный редактор через веб-клиент |
| **Escalation Rules** | Эскалация при превышении SLA |
| **Delegation** | Делегирование полномочий на время отсутствия |
| **Work Items** | Центр уведомлений с actionable сообщениями |
| **Power Automate Integration** | Внешние workflow через облачные триггеры |

---

## Безопасность и Compliance

### Role-Based Security

```text
Security Role (e.g., Accountant)
├── Duties (e.g., Maintain GL)
│   └── Privileges (e.g., Post Journal)
│       └── Permissions (Read/Create/Update/Delete)
└── Sub-roles
```

| Уровень | Описание |
|---|---|
| **Roles** | Набор обязанностей (Accountant, Warehouse Worker, Buyer) |
| **Duties** | Логические группы привилегий |
| **Privileges** | Доступ к конкретным операциям (Post Invoice, Create PO) |
| **Permissions** | CRUD на уровне таблицы/поля |

### Аудит

| Функция | Описание |
|---|---|
| **Database Log** | Логирование изменений на уровне таблиц |
| **Audit Trail** | История изменений документов (who, when, what) |
| **Segregation of Duties** | Автоматическая проверка конфликта обязанностей |
| **GDPR Compliance** | Управление персональными данными, право на забвение |
| **SOX Compliance** | Поддержка требований закона Сарбейнса-Оксли |

---

## Жизненный цикл внедрения (Lifecycle Services — LCS)

```text
┌────────────┐   ┌────────────┐   ┌────────────┐   ┌────────────┐
│  Analysis  │ → │   Design   │ → │   Build    │ → │   Deploy   │
└────────────┘   └────────────┘   └────────────┘   └────────────┘
                                                        │
                    ┌───────────────────────────────────┘
                    ▼
┌────────────┐   ┌────────────┐
│  Operate   │ ← │   Test     │
└────────────┘   └────────────┘
```

| Инструмент LCS | Описание |
|---|---|
| **Project Management** | Управление проектом внедрения (методология, задачи, документы) |
| **Environment Management** | Управление средами (Sandbox, UAT, Production) |
| **Issue Search** | База знаний по известным проблемам и исправлениям |
| **Update Management** | Установка обновлений и hotfix |
| **Business Process Modeler (BPM)** | Визуальное моделирование бизнес-процессов |
| **Asset Library** | Хранилище пакетов, дампов БД, конфигураций |
| **Telemetry & Monitoring** | Мониторинг производительности, ошибок, использования |

### Среда внедрения

| Среда | Назначение | Особенности |
|---|---|---|
| **Dev (Developer VM)** | Разработка и отладка | Локальная VM в Azure, 1 пользователь |
| **Build** | CI/CD, сборка пакетов | Автоматическая через Azure DevOps |
| **Sandbox Tier-2** | Тестирование | Полноценный AOS + SQL, до 100 пользователей |
| **UAT** | Приёмочное тестирование | Копия production конфигурации |
| **Production** | Промышленная эксплуатация | HA, DR, автомасштабирование |

---

## Интеграции

| Тип интеграции | Технология |
|---|---|
| **Dual-write** | Реального времени между F&O и Dataverse |
| **OData (REST)** | Стандартный REST API для внешних систем |
| **SOAP (Custom Services)** | X++ сервисы для legacy интеграций |
| **Data Entities** | Абстракции над таблицами для импорта/экспорта |
| **Data Management Framework (DMF)** | Пакетный импорт/экспорт через Excel, CSV, XML |
| **Logic Apps / Power Automate** | Cloud-to-cloud интеграция через коннекторы |
| **Service Bus / Event Hub** | Событийная интеграция (Azure Event Hub) |
| **File-based Integration** | Обмен файлами (SFTP, Azure Blob, SharePoint) |

---

## Преимущества и недостатки F&O

### Сильные стороны

| Преимущество | Описание |
|---|---|
| **Единая платформа** | Finance + SCM + Commerce + HR в одном runtime, без промежуточных шин |
| **Интеграция с Microsoft 365** | Excel, Teams, Outlook, SharePoint — нативная интеграция |
| **Power Platform** | Low-code расширения без изменения ядра F&O |
| **Microsoft Copilot** | AI-помощник в финансах и цепочках поставок (2024+) |
| **Глобализация** | 38+ стран локализации из коробки |
| **One Version** | Все клиенты на одной версии с ежемесячными обновлениями |
| **Azure-экосистема** | Active Directory, Key Vault, Monitor, Synapse |

### Слабые стороны

| Недостаток | Описание |
|---|---|
| **Сложность внедрения** | Проекты 12–24 месяца, требуется бизнес-аналитика и консалтинг |
| **Стоимость** | Высокая стоимость лицензий + внедрение ($500K–$5M+) |
| **Производительность** | SQL Server — бутылочное горлышко; нет in-memory Computing как у SAP HANA |
| **Кастомизация** | Over-customization — частая проблема, усложняющая обновления |
| **X++** | Нишевый язык, мало разработчиков на рынке |
| **Сложность обновлений** | Каждое обновление требует тестирования всех кастомизаций |
| **UI inconsistency** | Часть интерфейса старая (MorphX-стиль), часть новая (HTML5) |

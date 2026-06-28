# Microsoft Dynamics 365 — обзор экосистемы

---

## Что такое Microsoft Dynamics 365

Microsoft Dynamics 365 — это облачная (и гибридная) платформа бизнес-приложений от Microsoft, объединяющая ERP (Enterprise Resource Planning) и CRM (Customer Relationship Management) в единую экосистему. Запущена в 2016 году как эволюция Dynamics AX (Axapta), Dynamics NAV (Navision), Dynamics GP и Dynamics CRM.

### Ключевые факты

| Параметр | Значение |
|---|---|
| **Год запуска бренда D365** | 2016 (эволюция с 1998 — Damgaard Axapta) |
| **Модель развёртывания** | SaaS (облако), On-premises, Hybrid |
| **Основной стек** | .NET, X++, Power Platform, Azure |
| **Языки разработки** | X++, C# (.NET), JavaScript/TypeScript, Power Fx |
| **Доля рынка ERP** | ~5-7% мирового рынка ERP (2-е место после SAP) |
| **Число клиентов** | 700 000+ организаций (все приложения D365) |
| **Основной конкурент** | SAP S/4HANA, Oracle Fusion Cloud, NetSuite |

---

## Состав экосистемы Dynamics 365

### ERP-приложения (Finance & Operations)

| Приложение | Назначение | Ключевые модули |
|---|---|---|
| **Dynamics 365 Finance** | Управление финансами предприятия | General Ledger, AP/AR, Budgeting, Fixed Assets, Tax, Treasury |
| **Dynamics 365 Supply Chain Management** | Управление цепочками поставок | Inventory, Warehouse, Transportation, Procurement, Master Planning |
| **Dynamics 365 Commerce** | Розничная и омниканальная торговля | POS, Merchandising, Call Center, E-commerce |
| **Dynamics 365 Human Resources** | Управление персоналом | Core HR, Benefits, Leave/Absence, Compensation, Performance |
| **Dynamics 365 Project Operations** | Управление проектами | Project Planning, Resourcing, Time & Expense, Billing |

> **Факт:** Finance и Supply Chain Management разделены на отдельные приложения с 2023 года (ранее — единое Finance & Operations). Однако технически это одна кодовая база и один runtime.

### CRM-приложения (Customer Engagement)

| Приложение | Назначение |
|---|---|
| **Dynamics 365 Sales** | Управление продажами, воронки, lead scoring, Copilot |
| **Dynamics 365 Customer Service** | Поддержка клиентов, Service Level Agreements, Omnichannel |
| **Dynamics 365 Field Service** | Выездной сервис, планирование ресурсов, IoT |
| **Dynamics 365 Marketing** | Маркетинговые кампании, Customer Journey, Email marketing |

### Специализированные приложения

| Приложение | Назначение |
|---|---|
| **Dynamics 365 Business Central** | ERP для малого и среднего бизнеса (эволюция NAV) |
| **Dynamics 365 Guides** | Mixed Reality инструкции (HoloLens) |
| **Dynamics 365 Fraud Protection** | Защита от мошенничества в транзакциях |
| **Dynamics 365 Customer Insights** | Customer Data Platform (CDP), сегментация, AI |

---

## Power Platform — неразрывная часть экосистемы

Dynamics 365 тесно интегрирован с Power Platform:

| Компонент | Описание |
|---|---|
| **Power Apps** | Low-code разработка пользовательских приложений (canvas/model-driven) |
| **Power Automate** | Автоматизация бизнес-процессов и workflow (бывший Microsoft Flow) |
| **Power BI** | Бизнес-аналитика, дашборды, Embedded analytics |
| **Power Pages** | Создание внешних веб-порталов (бывший Power Apps Portals) |
| **Power Virtual Agents** | Чат-боты и Copilot-расширения |
| **Dataverse** | Единая платформа данных (бывший Common Data Service) |

> **Факт:** Все CRM-приложения D365 (Sales, Customer Service, Field Service, Marketing) работают на **Dataverse**. ERP-приложения (Finance, SCM) используют собственный движок данных, но синхронизируются с Dataverse через **Dual-write**.

---

## Модели лицензирования

### ERP (Finance & Operations)

| Модель | Описание |
|---|---|
| **Finance** | $180/пользователь/месяц (Full User), $30 (Team Member), $20 (Activity) |
| **Supply Chain Management** | $180/пользователь/месяц (Full User) |
| **Commerce** | $180/пользователь/месяц |
| **Human Resources** | $120/пользователь/месяц |

### CRM (Customer Engagement)

| Модель | Описание |
|---|---|
| **Sales Enterprise** | $95/пользователь/месяц |
| **Customer Service Enterprise** | $95/пользователь/месяц |
| **Field Service** | $95/пользователь/месяц |

> **Факт:** С 2019 года Microsoft ввела модель **"Attach" лицензирования** — при покупке первого приложения F&O (Finance или SCM) второе идёт с 20% скидкой. Это стимулирует использование обоих модулей.

---

## Common Data Model (CDM)

CDM — стандартизированная схема данных Microsoft, объединяющая все приложения экосистемы:

- **900+ стандартных сущностей** (Account, Contact, Product, Order, Invoice и др.)
- **Dataverse** — реализация CDM в облаке
- Обеспечивает совместимость D365, Power Platform и Azure Data Services
- Интеграция с **Microsoft Fabric** и **Azure Synapse** для аналитики

---

## История развития Axapta → D365 Finance & Operations

| Год | Событие |
|---|---|
| **1998** | Damgaard Data (Дания) выпускает Axapta 1.0 |
| **2000** | Microsoft покупает Damgaard (объединение с Navision в Microsoft Business Solutions) |
| **2002** | Axapta 3.0 — Multi-tier архитектура, AOS (Application Object Server) |
| **2006** | Dynamics AX 4.0 — Role Centers, Enterprise Portal |
| **2009** | Dynamics AX 2009 — Compliance, Globalization |
| **2012** | Dynamics AX 2012 — Model-Driven Architecture, Data Partitioning |
| **2016** | Dynamics 365 for Finance and Operations (облачная версия) |
| **2018** | Dynamics 365 for Finance and Operations → Dynamics 365 Finance & Operations |
| **2023** | Разделение на Finance и Supply Chain Management |

> **Факт:** Axapta изначально называлась "Damgaard Axapta" и была разработана датской компанией Damgaard Data, основанной братьями Эриком и Пребеном Дамгаард. Название происходит от слияния слов "Axa" (имя предыдущего продукта) и "apta" (от лат. "adaptare" — адаптироваться).

---

## Сравнение с конкурентами

| Критерий | D365 F&O | SAP S/4HANA | Oracle Fusion | NetSuite |
|---|---|---|---|---|
| **Целевой рынок** | Mid-Market + Enterprise | Enterprise | Enterprise | SMB + Mid-Market |
| **Облачность** | Cloud-first, есть on-prem | Cloud-first, on-prem возможен | Cloud | Cloud-only |
| **Язык разработки** | X++, C# | ABAP | Java, PL/SQL | SuiteScript (JS) |
| **Low-code** | Power Platform | SAP Build | VBCS | SuiteFlow |
| **AI** | Copilot (GPT-4) | Joule (SAP AI) | OCI AI | NetSuite AI |
| **Лицензирование** | User-based | User-based + FUE | User-based | User-based + modules |
| **Сильные стороны** | Интеграция с Microsoft 365, Excel, Power BI | Глубина manufacturing, global compliance | База данных, производительность | Малый бизнес, простота |
| **Слабые стороны** | Сложность F&O, X++ | Сложность внедрения, дороговизна | Слабая CRM | Ограниченная масштабируемость |

---

## Выбор приложения D365

```text
Нужно ERP для крупного бизнеса (>500 пользователей)?
├── Да → Dynamics 365 Finance + Supply Chain Management
└── Нет → Нужна CRM+ERP для малого/среднего бизнеса?
    ├── Да → Dynamics 365 Business Central
    └── Нет → Нужна только CRM?
        ├── Да → Dynamics 365 Sales / Customer Service / Field Service
        └── Нет → Нужна розничная торговля?
            └── Да → Dynamics 365 Commerce
```

---

## Полезные ссылки

- [Microsoft Dynamics 365 Official](https://dynamics.microsoft.com/)
- [Dynamics 365 Finance & Operations Documentation](https://learn.microsoft.com/en-us/dynamics365/finance/)
- [Dynamics 365 Community](https://community.dynamics.com/)
- [Dynamics 365 Blog](https://cloudblogs.microsoft.com/dynamics365/)
- [Power Platform Documentation](https://learn.microsoft.com/en-us/power-platform/)
- [Dynamics 365 Release Plans](https://learn.microsoft.com/en-us/dynamics365/release-plans/)

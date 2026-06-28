# Dynamics 365 Business Central — ERP для малого и среднего бизнеса

---

## Что такое Business Central

Dynamics 365 Business Central (BC) — облачное ERP-решение для малого и среднего бизнеса (SMB). Эволюция Microsoft Dynamics NAV (Navision), полностью переведённая на современный стек.

| Параметр | Значение |
|---|---|
| **Происхождение** | Navision (1987, Дания) → Dynamics NAV → Business Central (2018) |
| **Целевой рынок** | 10–500 сотрудников, SMB |
| **Развёртывание** | SaaS (Cloud), On-premises, Hybrid |
| **Стек** | AL Language (современный), C/AL (legacy), .NET Interop |
| **База данных** | SQL Server / Azure SQL DB |
| **Клиент** | Web Client (HTML5), Desktop App, Mobile, Teams |
| **Число клиентов** | 200 000+ организаций |
| **Ключевые конкуренты** | NetSuite, SAP Business One, Acumatica, Sage Intacct |

> **Факт:** Navision (Navigator + Vision) была основана в 1987 году в Дании Йеспером Балслёвом (Jesper Balslev) и Петером Бангом (Peter Bang). В 2002 году Microsoft купила Navision за $1.45 млрд.

---

## Архитектура Business Central

```text
┌─────────────────────────────────────────────────┐
│              CLIENTS (Web / Desktop / Mobile)     │
├─────────────────────────────────────────────────┤
│             NST (Navision Service Tier)           │
│  ┌─────────────────────────────────────────────┐ │
│  │         AL Runtime (современный)             │ │
│  │  ┌──────────┐ ┌──────────────┐ ┌─────────┐  │ │
│  │  │ Finance  │ │   Sales/Mkt  │ │  Purch.  │  │ │
│  │  ├──────────┤ ├──────────────┤ ├─────────┤  │ │
│  │  │Inventory │ │ Manufacturing│ │  Projects│  │ │
│  │  ├──────────┤ ├──────────────┤ ├─────────┤  │ │
│  │  │  Service │ │  Warehouse   │ │    HR    │  │ │
│  │  └──────────┘ └──────────────┘ └─────────┘  │ │
│  └─────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────┤
│          SQL Server / Azure SQL DB                │
└─────────────────────────────────────────────────┘
```

| Компонент | Описание |
|---|---|
| **NST (Service Tier)** | Сервер приложений — аналог AOS в F&O |
| **AL Language** | Современный язык разработки (с 2018), событийно-ориентированный |
| **C/AL** | Legacy язык (до 2018), клиент-серверный |
| **Extensions V2** | Изолированные расширения (без изменения базы) |
| **AppSource** | Маркетплейс приложений и расширений |

---

## Функциональные модули

### Финансы (Finance)

| Функция | Описание |
|---|---|
| **General Ledger** | План счетов, измерения (Dimensions), периоды, бюджеты |
| **AP/AR** | Счета к оплате/получению, Reconciliation |
| **Fixed Assets** | Основные средства, амортизация |
| **Cash Management** | Банковские счета, выверка |
| **Multi-Currency** | Полная мультивалютность |
| **Consolidation** | Консолидация нескольких компаний |
| **Intercompany** | Межфирменные проводки |
| **Cost Accounting** | Управленческий учёт |

### Продажи и маркетинг (Sales & Marketing)

| Функция | Описание |
|---|---|
| **Sales Orders** | Полный цикл Quote → Order → Ship → Invoice |
| **Pricing** | Гибкие ценовые группы, скидки, акции |
| **Campaigns** | Маркетинговые кампании |
| **Contacts** | Управление контактами и отношениями |
| **Opportunities** | Воронка продаж |

### Закупки (Purchasing)

| Функция | Описание |
|---|---|
| **Purchase Orders** | Заказы поставщикам |
| **Requisition** | Заявки на закупку |
| **Drop Shipment** | Прямая доставка от поставщика клиенту |
| **Return Orders** | Возвраты поставщикам |

### Склад (Inventory & Warehouse)

| Функция | Описание |
|---|---|
| **Item Tracking** | Серийные номера, партии, сроки годности (FEFO/FIFO) |
| **Warehouse Management** | Базовый склад (Bin, Pick, Put-away) |
| **Assembly** | Сборка (Kit, BOM) |
| **Item Charges** | Дополнительные расходы (фрахт, страховка, пошлина) |
| **Transfer Orders** | Перемещения между складами |

### Производство (Manufacturing)

| Функция | Описание |
|---|---|
| **BOM** | Спецификации (Production BOM, Assembly BOM) |
| **Routings** | Маршруты с операциями и центрами |
| **Production Orders** | Заказы на производство |
| **MRP** | Планирование потребностей |
| **Capacity Planning** | Планирование загрузки |
| **Subcontracting** | Субподряд |

### Проекты (Jobs / Projects)

| Функция | Описание |
|---|---|
| **Job Management** | Бюджетирование, учёт затрат |
| **Resource Management** | Планирование ресурсов |
| **Time Sheets** | Табели учёта времени |
| **WIP** | Расчёт незавершённого производства |

### Сервис (Service Management)

| Функция | Описание |
|---|---|
| **Service Orders** | Заказы на обслуживание |
| **Service Contracts** | Сервисные контракты |
| **Planning** | Планирование выездов |
| **Repair Management** | Учёт ремонтов |

---

## AL Language — современная разработка

AL пришёл на смену C/AL в 2018 году:

```al
// Пример AL — Table Extension
tableextension 50100 "Customer Extension" extends Customer
{
    fields
    {
        field(50100; "Loyalty Points"; Integer)
        {
            Caption = 'Loyalty Points';
            MinValue = 0;
            MaxValue = 10000;
        }
        field(50101; "Customer Segment"; Enum "Customer Segment")
        {
            Caption = 'Customer Segment';
        }
    }
}

// Codeunit
codeunit 50100 "Loyalty Management"
{
    procedure AddLoyaltyPoints(CustomerNo: Code[20]; Amount: Decimal)
    var
        Customer: Record Customer;
    begin
        Customer.Get(CustomerNo);
        if Customer.FindFirst() then begin
            Customer."Loyalty Points" += Amount;
            Customer.Modify(true);
            if Customer."Loyalty Points" >= 1000 then
                Customer."Customer Segment" := Customer."Customer Segment"::Platinum;
        end;
    end;
}
```

### AL vs C/AL

| Критерий | AL | C/AL |
|---|---|---|
| **Год внедрения** | 2018 | 1987 |
| **Среда разработки** | VS Code + AL Extension | C/SIDE (встроенный клиент) |
| **Source Control** | Git, Azure DevOps | Отсутствует |
| **Расширения** | Extensions V2 (изолированные) | Overlay (изменение базы) |
| **Синтаксис** | Современный, Pascal-based | Устаревший, Pascal-based |
| **Обновления** | Автоматические (SaaS) | Ручные, сложные |
| **Интеграция** | REST API, OData, SOAP | Только SOAP |

---

## Extensions V2 — изолированная кастомизация

```text
                   ┌──────────────────────┐
                   │     BASE APP          │
                   │  (Microsoft Code)     │
                   └──────────┬───────────┘
                              │
         ┌────────────────────┼────────────────────┐
         ▼                    ▼                    ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│   Extension A   │ │   Extension B   │ │   Extension C   │
│   (ISV / VAR)   │ │   (Customer)    │ │   (AppSource)   │
└─────────────────┘ └─────────────────┘ └─────────────────┘
```

| Свойство | Описание |
|---|---|
| **Изоляция** | Расширения не модифицируют базовый код |
| **AppSource** | Публикация и монетизация расширений |
| **Зависимости** | Extension может зависеть от другого Extension |
| **Версионирование** | Полное управление версиями |
| **Обновления** | Автоматическое обновление базового приложения без конфликтов |

---

## Сравнение Business Central vs Finance & Operations

| Критерий | Business Central | Finance & Operations |
|---|---|---|
| **Целевой рынок** | 10–500 сотрудников | 300–10 000+ сотрудников |
| **Язык** | AL | X++ |
| **IDE** | VS Code | Visual Studio / MorphX |
| **Сложность внедрения** | Недели/месяцы | 12–24 месяца |
| **Стоимость лицензии** | $70–100/польз/мес | $180/польз/мес |
| **Производство** | Базовое (дискретное) | Продвинутое (дискретное, процессное, Lean) |
| **Склад** | Базовый WMS | Продвинутый WHS |
| **Глобализация** | Базовая | 38+ стран, сложные налоги |
| **Мастер-планирование** | MRP | Planning Optimization (in-memory) |
| **Intercompany** | Базовое | Продвинутое (multi-level) |
| **Количество юрлиц** | Несколько | Десятки/сотни |

> **Факт:** Business Central не является «урезанной» версией F&O. Это самостоятельный продукт с собственной архитектурой, языком (AL) и парадигмой разработки. Выбор между BC и F&O зависит от размера бизнеса и сложности процессов, а не только от бюджета.

---

## Интеграции Business Central

| Тип интеграции | Технология |
|---|---|
| **REST API** | Встроенный OData V4 и REST API |
| **SOAP Web Services** | Для legacy-совместимости |
| **Power Platform** | Power Apps, Power Automate, Power BI |
| **Microsoft 365** | Teams, Excel, Outlook, SharePoint |
| **Dataverse** | Виртуальные таблицы (Virtual Tables) |
| **Azure Services** | Logic Apps, Service Bus, Event Grid |
| **EDI** | Электронный обмен документами |
| **Shopify / e-Commerce** | Готовые коннекторы |

---

## Миграция с NAV на Business Central

```text
Dynamics NAV (C/AL) ──→ Business Central (AL)
         │                       │
         ▼                       ▼
  C/SIDE Client           VS Code + AL
         │                       │
         ▼                       ▼
   Overlay Code           Extensions V2
         │                       │
         ▼                       ▼
   Manual Updates         Automatic Updates (SaaS)
```

| Шаг миграции | Описание |
|---|---|
| **1. Оценка** | Инвентаризация кода C/AL, выявление кастомизаций |
| **2. Конвертация** | C/AL → AL через инструменты Microsoft |
| **3. Рефакторинг** | Замена устаревших паттернов (overlay → events) |
| **4. Extensions** | Разделение монолита на модульные расширения |
| **5. Тестирование** | Полный регрессионный цикл |
| **6. Деплой** | SaaS (автоматически) или On-prem (ручной) |

---

## Полезные ресурсы

- [Business Central Documentation](https://learn.microsoft.com/en-us/dynamics365/business-central/)
- [AL for Business Central](https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/devenv-get-started)
- [Business Central on AppSource](https://appsource.microsoft.com/marketplace/apps?product=dynamics-365-business-central)
- [BC Container Helper](https://github.com/microsoft/navcontainerhelper) (PowerShell для dev-сред)
- [AL Go! for VS Code](https://github.com/microsoft/AL) — современная AL-разработка

# Практическое применение AI

---

## AI-агенты

### Что такое AI-агент

**AI-агент** — система, которая использует LLM для планирования, выполнения действий и взаимодействия с окружением.

```
User Goal
    │
    ▼
┌────────────────────────────────────────┐
│              AI Agent                  │
│  ┌──────────┐  ┌────────────────────┐  │
│  │  LLM     │  │  Tools             │  │
│  │  (brain) │  │  - Code Interpreter│  │
│  │          │  │  - Web Search      │  │
│  │          │  │  - File System     │  │
│  └────┬─────┘  │  - API Calls       │  │
│       │        └────────────────────┘  │
│       │  ┌────────────────────────┐    │
│       └──│  Memory (short/long)   │    │
│          └────────────────────────┘    │
└────────────────────────────────────────┘
    │
    ▼
  Result
```

### Agentic Patterns

| Паттерн | Описание | Пример |
|---------|----------|--------|
| **ReAct** | Reasoning + Acting (Chain of Thought + Tools) | "Подумай, затем выполни tool, затем подумай снова" |
| **Plan-and-Execute** | Сначала план, потом шаги | Разбить задачу на подзадачи |
| **Multi-agent** | Несколько агентов с разными ролями | Code Agent + Review Agent + Test Agent |
| **Reflection** | Проверка собственного вывода | "Проверь своё решение на ошибки" |

### Пример: Agentic RAG

```
User: "Какая текущая сумма заказов Пети?"
    │
Agent:
  1. Поиск "Петя" → user_id=42
  2. SQL: SELECT SUM(amount) FROM orders WHERE user_id=42
  3. Ответ: "Текущая сумма заказов Пети: 15 230 ₽"
```

**Факт:** Agentic RAG — следующий шаг после простого RAG. Агент сам решает, когда искать в векторной БД, когда в SQL, когда вызывать API.

---

## MLOps

### Жизненный цикл ML-модели

```
Data → Train → Evaluate → Deploy → Monitor → Retrain
  ↑                                          │
  └──────────────────────────────────────────┘
```

### Feature Store

Централизованное хранилище признаков для обучения и инференса.

```python
# Пример: Feast Feature Store
feature_store = FeatureStore()

# Определение признака
feature_view = FeatureView(
    name="user_transaction_features",
    entities=["user_id"],
    features=[
        Feature(name="total_transactions_7d", dtype=ValueType.INT64),
        Feature(name="avg_amount_30d", dtype=ValueType.FLOAT),
    ],
    source=BigQuerySource(...)
)

# Использование в инференсе
features = feature_store.get_online_features(
    feature_views=["user_transaction_features"],
    entity_rows=[{"user_id": 42}]
).to_dict()
```

### Model Registry

```
Experiment → Model Version → Stage (staging/production) → Deployment
```

**Инструменты:** MLflow, Weights & Biases, DVC, Kubeflow.

### Model Monitoring

| Метрика | Описание |
|---------|----------|
| **Data drift** | Изменение распределения входных данных |
| **Concept drift** | Изменение зависимости X→y |
| **Prediction drift** | Изменение распределения предсказаний |
| **Accuracy** | Снижение качества на fresh data |

```python
# Evidently AI — мониторинг drift
from evidently import ColumnMapping
from evidently.report import Report
from evidently.metric_preset import DataDriftPreset

report = Report(metrics=[DataDriftPreset()])
report.run(reference_data=train_data, current_data=new_data)
report.show()
```

### Batch vs Real-time Inference

| | Batch | Real-time |
|---|-------|-----------|
| **Когда** | Раз в N часов/день | На каждый запрос |
| **Задержка** | Минуты-часы | < 100ms |
| **Сложность** | Низкая (cron/spark) | Высокая (API + caching) |
| **Пример** | Ежедневные рекомендации | Fraud detection |

---

## Инструменты AI-разработки

### LLM Frameworks

| Инструмент | Назначение |
|------------|------------|
| **LangChain** | Фреймворк для LLM-приложений (chains, agents, RAG) |
| **LlamaIndex** | Data framework для RAG и индексации |
| **Semantic Kernel** (Microsoft) | .NET-фреймворк для AI-оркестрации |
| **Haystack** | NLP-пайплайны (search, QA, RAG) |

### Semantic Kernel (.NET)

```csharp
// Создание kernel
var kernel = Kernel.CreateBuilder()
    .AddAzureOpenAIChatCompletion("gpt-4", endpoint, apiKey)
    .Build();

// Определение плагина
public class DatabasePlugin
{
    [KernelFunction("get_user_orders")]
    [Description("Get total orders for a user")]
    public async Task<decimal> GetUserOrdersAsync(
        [Description("User ID")] int userId)
    {
        // SQL query
    }
}

// Вызов с agentic loop
var result = await kernel.InvokePromptAsync(
    "How many orders does user 42 have?",
    new KernelArguments { { "plugins", new DatabasePlugin() } });
```

### Vector Databases

| БД | Тип | Когда |
|----|-----|-------|
| **Qdrant** | Vector-only | Высокая производительность |
| **Pinecone** | Managed | Не хочу управлять инфрой |
| **Weaviate** | Vector + Object | Нужны гибридные запросы |
| **Pgvector** | PostgreSQL extension | Уже есть PostgreSQL |
| **Chroma** | Embedded | Dev/prototyping |

---

## Ограничения и риски

### Когда НЕ использовать AI

- ✅ **Правильная задача:** Классификация, генерация, суммаризация, поиск
- ❌ **Неправильная задача:** Точные расчёты, критичные к accuracy, конфиденциальные данные без защиты

### CAP-теорема для AI

```
Accuracy  ←→  Cost  ←→  Latency
Выберите любые два.
```

### Security и AI

| Угроза | Описание | Защита |
|--------|----------|--------|
| **Prompt Injection** | Злонамеренный промпт взламывает LLM | Input validation, guardrails |
| **Data Poisoning** | Отравление training data | Data provenance, validation |
| **Model Inversion** | Восстановление данных из модели | Differential privacy |
| **Supply Chain** | Уязвимости в open-source моделях | SBOM, model scanning |

---

## Чек-лист внедрения AI

- [ ] Бизнес-задача сформулирована и измерима
- [ ] Baseline (без AI) посчитан
- [ ] Данные доступны, размечены и качественны
- [ ] Метрики успеха определены (Accuracy? Cost saving? Time?)
- [ ] Выбрана архитектура (RAG / Fine-tuning / Prompt)
- [ ] MLOps пайплайн настроен (data → train → deploy → monitor)
- [ ] Мониторинг дрифта включён
- [ ] Security review проведён
- [ ] Cost per inference посчитан

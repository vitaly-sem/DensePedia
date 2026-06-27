# Мониторинг, логирование и observability

---

## Observability vs Monitoring

```
Monitoring  = знаем, что сломалось (known unknowns)
Observability = можем понять, почему сломалось (unknown unknowns)
```

**Observability** — свойство системы, позволяющее задавать **новые вопросы** о её состоянии без необходимости добавлять новый код.

### Три столпа observability

```
┌────────────────────────────────────────────────────┐
│                 Observability                      │
├──────────────┬──────────────┬──────────────────────┤
│   Metrics    │     Logs     │        Traces        │
│ (Prometheus) │   (ELK/Loki) │  (OpenTelemetry)     │
├──────────────┼──────────────┼──────────────────────┤
│ Численные    │ Структур.    │ Распределённые       │
│ показатели   │ записи       │ трейсы запросов      │
│ с тегами     │ событий      │ (span-based)         │
└──────────────┴──────────────┴──────────────────────┘
```

---

## Метрики

### RED-метрики (для микросервисов)

```
Rate     = количество запросов в секунду
Errors   = количество ошибок (5xx, 4xx)
Duration = время ответа (p50, p95, p99)
```

### USE-метрики (для инфраструктуры)

```
Utilization = насколько занят ресурс (%)
Saturation  = насколько ресурс перегружен (queue length)
Errors      = количество ошибок ресурса
```

### Пример Prometheus-метрик

```csharp
// Counter — только растёт
private static readonly Counter RequestCount = Metrics
    .CreateCounter("http_requests_total", "Total HTTP requests",
        new CounterConfiguration { LabelNames = new[] { "method", "endpoint", "status" } });

// Histogram — распределение
private static readonly Histogram RequestDuration = Metrics
    .CreateHistogram("http_request_duration_seconds", "HTTP request duration",
        new HistogramConfiguration
        {
            LabelNames = new[] { "method", "endpoint" },
            Buckets = new[] { 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5 }
        });

// Gauge — может расти и уменьшаться
private static readonly Gauge ActiveConnections = Metrics
    .CreateGauge("active_connections", "Number of active connections");
```

### Четыре золотых сигнала (Google SRE)

1. **Latency** — время ответа
2. **Traffic** — нагрузка на систему
3. **Errors** — явные + неявные ошибки (например, 200 OK с пустым телом)
4. **Saturation** — степень заполненности ресурсов

---

## SLI / SLO / SLA

```
SLA — юридическое обязательство (99.9% uptime)
SLO — цель, к которой стремимся (99.95% uptime)
SLI — фактическая метрика (99.97% uptime)
```

### Пример SLO для API

```
SLI = proportion of requests with latency < 200ms
SLO = 99.9% запросов за месяц < 200ms
Error Budget = 100% - 99.9% = 0.1% времени простоя
```

**Факт:** Google SRE использует **error budget** — если бюджет исчерпан, новые фичи не выкатываются, только reliability-работы.

---

## Логирование

### Структурированные логи

```json
{
  "@timestamp": "2025-06-27T20:00:00.000Z",
  "level": "ERROR",
  "message": "Database connection failed after 3 retries",
  "service": "order-service",
  "trace_id": "abc123def456",
  "exception": {
    "type": "Npgsql.NpgsqlException",
    "message": "Connection refused",
    "stacktrace": "..."
  },
  "metadata": {
    "db_instance": "orders-postgres-prod-01",
    "retry_attempt": 3,
    "timeout_ms": 5000
  }
}
```

### Serilog (C#)

```csharp
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .Enrich.WithProperty("Application", "OrderService")
    .Enrich.WithEnvironmentName()
    .WriteTo.Console(new JsonFormatter())
    .WriteTo.Seq("http://seq:5341")
    .CreateLogger();
```

### Лучшие практики

| Делать | Не делать |
|--------|-----------|
| Структурированные логи (JSON) | Писать текст в консоль |
| Уровни: Debug / Info / Warn / Error / Fatal | Ставить всё на Info |
| Include trace_id для корреляции | Логировать пароли/tokens |
| Асинхронная запись | Блокировать thread на записи |
| Централизованное хранилище | Хранить на локальном диске |

---

## Трейсинг (Distributed Tracing)

### OpenTelemetry

```csharp
// Настройка OpenTelemetry в ASP.NET Core
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddNpgsqlInstrumentation()
        .AddSource("OrderService")
        .SetSampler(new AlwaysOnSampler())
        .AddOtlpExporter(options =>
            options.Endpoint = new Uri("http://otel-collector:4317")));
```

### Span-based модель

```
[Root Span: POST /api/orders (2.3s)]
   ├── [Span: ValidateOrder (45ms)]
   ├── [Span: ReserveInventory (320ms)]
   │    └── [Span: SQL Query (280ms)]
   ├── [Span: ProcessPayment (1.2s)]
   │    └── [Span: HTTP POST /payment (1.1s)]
   └── [Span: SendNotification (180ms)]
        └── [Span: Publish to Queue (50ms)]
```

---

## Стек observability

### Prometheus + Grafana

```
Приложение → метрики (HTTP /metrics) → Prometheus (scrape) → Grafana (dashboard)
                                                              → AlertManager → Slack/PagerDuty
```

**Факт:** Prometheus — pull-based. PushGateway — только для batch-задач.

### ELK Stack (Elasticsearch, Logstash, Kibana)

```
Log → Filebeat → Logstash/Elastic Agent → Elasticsearch → Kibana
```

### Loki + Grafana

```
Log → Promtail → Loki (log aggregation) → Grafana
```

Лучше ELK для Kubernetes: не индексирует содержимое логов (only labels), работает дешевле.

### Datadog / New Relic / Grafana Cloud

SaaS-решения с единой платформой для метрик + логов + трейсов.

---

## Алертинг

### Правила Prometheus (PromQL)

```yaml
groups:
- name: api
  rules:
  - alert: HighErrorRate
    expr: |
      rate(http_requests_total{status=~"5.."}[5m])
      /
      rate(http_requests_total[5m]) > 0.05
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Error rate > 5% for {{ $labels.service }}"
      description: "5-minute error rate is {{ $value | humanizePercentage }}"

  - alert: HighLatency
    expr: |
      histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) > 1
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "p99 latency > 1s for {{ $labels.service }}"
```

### On-Call best practices

- **Никогда не будить на неважное** — каждая ночная тревога = выгорание
- **Runbook** — что делать при каждой тревоге
- **Escape hatch** — acknowledge, но не snooze навсегда
- **Post-mortem без blame** — системные проблемы, не люди

---

## Чек-лист observability

- [ ] RED-метрики (Rate, Errors, Duration) настроены для всех сервисов
- [ ] Dashboard'ы в Grafana для каждого сервиса
- [ ] Алерты с правильной severity и runbook'ами
- [ ] Централизованное логирование (структурированные логи)
- [ ] Distributed tracing (OpenTelemetry) для всех запросов
- [ ] SLI/SLO определены и мониторятся
- [ ] Error budget отслеживается
- [ ] On-call ротация с эскалацией
- [ ] Post-mortem после каждого инцидента

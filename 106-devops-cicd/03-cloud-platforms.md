# Облачные платформы и инфраструктура

---

## Big 3: AWS vs Azure vs GCP

| Критерий | AWS | Azure | GCP |
|----------|-----|-------|-----|
| **Рынок** | ~32% | ~23% | ~11% |
| **Сильные стороны** | Самый широкий спектр сервисов | Глубокая интеграция с .NET/Microsoft | Big Data, ML, Kubernetes (GKE) |
| **Kubernetes** | EKS | AKS | GKE (лучший managed K8s) |
| **Serverless** | Lambda | Functions | Cloud Functions, Cloud Run |
| **Базы данных** | RDS, Aurora, DynamoDB | SQL Database, Cosmos DB | Cloud SQL, Spanner, Bigtable |
| **CDN** | CloudFront | Azure CDN, Front Door | Cloud CDN |

### Факты для выбора

**AWS** — если стартап или продукт agnostic; самый богатый ecosystem.

**Azure** — если .NET-стек, Entra ID, гибридные сценарии (on-prem + cloud).

**GCP** — если data-heavy проекты, ML/AI, Kubernetes-native архитектура.

---

## Serverless

### AWS Lambda

```csharp
public class Function
{
    public async Task<string> Handler(SQSEvent evnt, ILambdaContext context)
    {
        foreach (var record in evnt.Records)
        {
            await ProcessMessage(record.Body);
        }
        return $"Processed {evnt.Records.Count} messages";
    }
}
```

**Факт:** Lambda может масштабироваться до **1000 concurrent executions** (по умолчанию).

### Azure Functions

```csharp
[FunctionName("ProcessOrder")]
public static async Task<IActionResult> Run(
    [HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequest req,
    [CosmosDB("Orders", "Items", ConnectionStringSetting = "CosmosDB")] IAsyncCollector<Order> orders,
    ILogger log)
{
    var order = await req.ReadFromJsonAsync<Order>();
    await orders.AddAsync(order);
    return new OkResult();
}
```

### Cloud Run (GCP)

```yaml
# Сервис на Cloud Run
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - image: gcr.io/my-project/myapp:1.0.0
        ports:
        - containerPort: 8080
        env:
        - name: DATABASE_URL
          value: "postgres://..."
```

**Факт:** Cloud Run платит только за время обработки запроса (до 100ms cold start).

---

## Infrastructure as Code — сравнение

### Terraform (HashiCorp)

```hcl
provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "rg" {
  name     = "rg-myapp"
  location = "westeurope"
}

resource "azurerm_postgresql_flexible_server" "db" {
  name                = "psql-myapp"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  sku_name            = "B_Standard_B1ms"
  storage_mb          = 32768
  version             = "16"
  administrator_login    = "dbadmin"
  administrator_password = var.db_password
}
```

### Pulumi

```csharp
using Pulumi;
using Pulumi.AzureNative.Resources;
using Pulumi.AzureNative.Storage;
using Pulumi.AzureNative.Storage.Inputs;

class MyStack : Stack
{
    public MyStack()
    {
        var rg = new ResourceGroup("rg-myapp");

        var storage = new StorageAccount("stmyapp", new StorageAccountArgs
        {
            ResourceGroupName = rg.Name,
            Kind = "StorageV2",
            Sku = new SkuArgs { Name = "Standard_LRS" }
        });

        this.ContainerName = storage.Name;
    }

    [Output] public Output<string> ContainerName { get; set; }
}
```

---

## Multi-Cloud стратегии

| Стратегия | Описание | Риски |
|-----------|----------|-------|
| **Изоляция** | Разные clouds для разных сервисов (Azure для .NET, GCP для ML) | Операционная сложность |
| **Резервирование** | Активный/пассивный DR в другом cloud'е | Data egress costs, eventual consistency |
| **Абстракция** | Абстрактный слой (Terraform, Crossplane) над провайдерами | Lowest common denominator |
| **Избегать** | Vendor lock-in через managed сервисы | Выход дорогой |

**Факт:** Data egress из AWS стоит $0.09/GB. Ежемесячный egress 10TB → $900+.

---

## Cloud Cost Optimization

### FinOps — три фазы

```
┌──────────────────────────────────────────────────┐
│           Inform (Visibility)                    │
│  Кто что тратит? Cost allocation tags, budgets   │
├──────────────────────────────────────────────────┤
│           Optimize (Efficiency)                  │
│  Reserved Instances, Spot instances, rightsizing │
├──────────────────────────────────────────────────┤
│           Operate (Continuous)                   │
│  Automation, policies, culture of cost awareness  │
└──────────────────────────────────────────────────┘
```

### Практики снижения затрат

| Практика | Экономия | Сложность |
|----------|----------|-----------|
| Reserved Instances (1yr) | ~30-40% | Низкая |
| Spot instances (batch) | ~60-90% | Средняя |
| Rightsizing (подбор SKU) | ~20-30% | Средняя |
| Auto-scaling (off-hours) | ~40-60% | Средняя |
| S3 lifecycle policies | ~50% (старые данные) | Низкая |
| Удаление неиспользуемых ресурсов | Variable | Высокая |

**Факт:** ~30% облачных ресурсов в среднем не используются (https://www.techrepublic.com/article/abandoned-cloud-resources/).

---

## Чек-лист облачной инфраструктуры

- [ ] Ресурсы тегированы (CostCenter, Environment, Owner)
- [ ] Настроены бюджеты и алерты по превышению
- [ ] Используются Reserved Instances / Savings Plans для стабильной нагрузки
- [ ] Non-production среды выключаются в нерабочее время
- [ ] Data egress оптимизирован (CDN, компрессия)
- [ ] Логи централизованы (Azure Log Analytics, CloudWatch, Stackdriver)
- [ ] RBAC настроен по принципу least privilege
- [ ] Все изменения через IaC (Terraform / Pulumi / Bicep)
- [ ] Disaster Recovery план протестирован

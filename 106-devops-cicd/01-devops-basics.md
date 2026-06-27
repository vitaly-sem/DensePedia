# Основы DevOps и CI/CD

---

## Что такое DevOps?

**DevOps** — это не должность, не инструмент и не команда. Это **культура**, объединяющая разработку (Dev) и эксплуатацию (Ops) для сокращения цикла поставки изменений при сохранении качества и надёжности.

### Три принципа (The Three Ways — Gene Kim)

```
┌─────────────────────────────────────────────────┐
│  Первый путь: Flow                              │
│  (Ускорение потока — от коммита до продакшена)  │
├─────────────────────────────────────────────────┤
│  Второй путь: Feedback                          │
│  (Быстрая обратная связь — мониторинг, алерты)  │
├─────────────────────────────────────────────────┤
│  Третий путь: Continuous Learning               │
│  (Эксперименты, блэйдорты, post-mortem без blame)│
└─────────────────────────────────────────────────┘
```

---

## CI/CD — Continuous Integration / Continuous Delivery

### Continuous Integration (CI)

Разработчики интеграции свои изменения в общую ветку **несколько раз в день**. Каждый коммит проходит:

- **Сборку** (`build`)
- **Юнит-тесты** (`test`)
- **Линтер / статический анализ** (`lint`)
- **Проверки безопасности** (`SAST`, `dependency scan`)

**Факт:** В Netflix CI-пайплайн выполняет **10 000+ тестов за 15 минут**.

### Continuous Delivery (CD)

Каждое прошедшее CI изменение готово к выкатке в продакшен, но выкатывается **вручную или по триггеру**.

### Continuous Deployment (CD — тот же акроним)

Каждое прошедшее CI изменение **автоматически** выкатывается в продакшен.

```
Код → CI (build + test) → Staging → Production
         ↑                    ↑           ↑
       feature              manual     auto (cont. deployment)
```

### Типовой CI/CD-пайплайн

```yaml
# Пример .gitlab-ci.yml
stages:
  - build
  - test
  - security
  - publish
  - deploy

build:
  stage: build
  script: dotnet build --configuration Release

test:
  stage: test
  script: dotnet test --configuration Release

security:
  stage: security
  script: dotnet list package --vulnerable

publish:
  stage: publish
  script: dotnet publish -c Release -o ./publish

deploy:
  stage: deploy
  script: kubectl apply -f k8s/
```

---

## GitOps

**GitOps** — практика, где **Git-репозиторий является единственным источником истины (single source of truth)** для инфраструктуры и приложений.

| Принцип | Описание |
|---------|----------|
| **Декларативность** | Всё состояние описано в манифестах (YAML, HCL) |
| **Версионирование** | Каждое изменение — Pull Request с review |
| **Автоматическая синхронизация** | Оператор (ArgoCD, Flux) применяет желаемое состояние |
| **Self-healing** | Отклонения от Git-состояния автоматически исправляются |

**Факт:** GitOps — стандарт де-факто для Kubernetes-инфраструктуры.

### ArgoCD vs Flux

| | ArgoCD | Flux |
|---|--------|------|
| UI | Богатый Web UI | CLI + Dashboard опционально |
| Sync | Pull-based, auto/prune | Pull-based, webhook/automation |
| SSO | Dex, OIDC, SAML | OIDC через GitHub |
| Multi-cluster | Нативный | Через Kustomize/Helm |
| Сообщество | CNCF graduated | CNCF graduated |

---

## Infrastructure as Code (IaC)

Управление инфраструктурой через код, а не через ручные операции.

### Типы IaC

| Подход | Инструменты | Описание |
|--------|-------------|----------|
| **Декларативный** | Terraform, Pulumi, ARM/Bicep | Описываем **что** должно быть; инструмент сам приводит к состоянию |
| **Императивный** | Ansible, Chef, Puppet | Описываем **как** привести к состоянию (шаги) |

### Terraform — основы

```hcl
resource "azurerm_resource_group" "main" {
  name     = "rg-myapp-prod"
  location = "westeurope"
}

resource "azurerm_kubernetes_cluster" "main" {
  name                = "aks-myapp-prod"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  dns_prefix          = "myapp-prod"

  default_node_pool {
    name       = "default"
    node_count = 3
    vm_size    = "Standard_D4s_v3"
  }

  identity {
    type = "SystemAssigned"
  }
}
```

**Факт:** Terraform State — критический артефакт. Хранить в удалённом backend'е (Azure Storage, S3 + DynamoDB) с блокировками.

---

## Практики DevOps

### Trunk-Based Development

- Все разработчики коммитят в `main` (или `trunk`) **несколько раз в день**
- Feature-флаги вместо долгоживущих веток
- Короткоживущие ветки (< 1 дня)

### Feature Flags

```csharp
if (featureFlags.IsEnabled("NewCheckoutFlow"))
{
    // новый код
}
else
{
    // старый код
}
```

**Инструменты:** LaunchDarkly, Unleash, Flagsmith, ConfigCat.

### Canary Deployments

Выкатка новой версии **на малый процент трафика** (5-10%) с автоматическим мониторингом ошибок и rollback'ом при аномалиях.

### Blue-Green Deployment

Два идентичных окружения (Blue = текущий, Green = новый). После валидации Green — переключение трафика.

```
        ┌─── Blue (v1) ───┐
User ───┤                  ├── production
        └─── Green (v2) ──┘
```

---

## Чек-лист DevOps-практик

- [ ] CI-пайплайн запускается на каждый PR/коммит
- [ ] Все артефакты версионированы и хранятся в registry
- [ ] Утверждён SLA/SLO для сервисов
- [ ] Настроен мониторинг (RED-метрики: Rate, Errors, Duration)
- [ ] Настроены алерты с эскалацией
- [ ] Есть runbook для инцидентов
- [ ] Post-mortem проводятся без blame
- [ ] Все изменения проходят code review
- [ ] IaC-код хранится в Git и проходит review
- [ ] Развёртывания автоматизированы

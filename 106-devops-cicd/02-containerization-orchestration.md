# Контейнеризация и оркестрация

---

## Docker — основы

**Docker** — платформа для упаковки, доставки и запуска приложений в контейнерах.

### Образ vs Контейнер

```
┌──────────────────────────┐
│       Docker Image       │  ← Read-only шаблон (слоистый)
│  ┌────────────────────┐  │
│  │      Layer 3       │  │  ← app code
│  ├────────────────────┤  │
│  │      Layer 2       │  │  ← dotnet runtime
│  ├────────────────────┤  │
│  │      Layer 1       │  │  ← OS base (alpine, ubuntu)
│  └────────────────────┘  │
└──────────────────────────┘
            ↓ docker run
┌──────────────────────────┐
│    Docker Container      │  ← Read-write layer + изолированный процесс
└──────────────────────────┘
```

### Dockerfile — best practices (.NET)

```dockerfile
# 1. Многоступенчатая сборка (multi-stage build)
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["MyApp.csproj", "."]
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app

FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine AS runtime
WORKDIR /app
EXPOSE 8080
COPY --from=build /app .
USER app
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

**Best practices:**
- Использовать **multi-stage build** — итоговый образ минимального размера
- Указывать **явную версию** тега (не `latest`)
- Запускать от **non-root** пользователя (`USER app`)
- **Healthcheck** обязательно

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

### .dockerignore

```
**/.classpath
**/.dockerignore
**/.git
**/.gitignore
**/.vs/
**/bin/
**/obj/
**/node_modules/
```

---

## Kubernetes — оркестратор контейнеров

Kubernetes (K8s) — платформа для автоматизации развёртывания, масштабирования и управления контейнерами.

### Архитектура

```
┌─────────────────────────────────────────────────┐
│                 Control Plane                   │
│  ┌──────────┐ ┌──────────┐ ┌────────────────┐  │
│  │  API     │ │ Scheduler│ │ Controller     │  │
│  │  Server  │ │          │ │ Manager        │  │
│  └────┬─────┘ └──────────┘ └────────────────┘  │
│       │                                          │
│  ┌────┴─────┐                                   │
│  │   etcd   │  ← cluster state store             │
│  └──────────┘                                   │
└─────────────────────────────────────────────────┘
         │
         │ kubectl / API
         ▼
┌─────────────────────────────────────────────────┐
│               Worker Node 1                     │
│  ┌──────────┐ ┌──────────┐ ┌────────────────┐  │
│  │  kubelet │ │  kube-   │ │    Pods         │  │
│  │          │ │  proxy   │ │  ┌────┐┌────┐  │  │
│  └──────────┘ └──────────┘ │  │App1││App2│  │  │
│                            │  └────┘└────┘  │  │
│  ┌──────────────────────┐ │  ┌────┐         │  │
│  │   Container Runtime  │ │  │App3│         │  │
│  │   (containerd)       │ │  └────┘         │  │
│  └──────────────────────┘ └─────────────────┘  │
└─────────────────────────────────────────────────┘
```

### Основные объекты

| Объект | Описание |
|--------|----------|
| **Pod** | Минимальная единица — 1+ контейнеров с общим storage/network |
| **Deployment** | Декларативное управление Pod'ами (replicas, rollout, rollback) |
| **Service** | Стабильный endpoint (ClusterIP, NodePort, LoadBalancer) |
| **Ingress** | HTTP/HTTPS-маршрутизация извне |
| **ConfigMap / Secret** | Конфигурация и чувствительные данные |
| **PersistentVolumeClaim** | Запрос на хранение данных |
| **HorizontalPodAutoscaler** | Автоматическое масштабирование |

### Пример Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myregistry.azurecr.io/myapp:1.0.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
```

### Requests vs Limits — почему это важно

```
Requests = гарантированный минимум (scheduler)
Limits  = максимум (cgroups throttling)
```

**Факт:** Превышение memory limit → **OOMKill**. Превышение CPU limit → **throttling** (не убивает).

### Service-типы

| Тип | Видимость | Использование |
|-----|-----------|---------------|
| **ClusterIP** | Только внутри кластера | Внутренние сервисы |
| **NodePort** | На IP узла через порт 30000-32767 | Dev/тестирование |
| **LoadBalancer** | Внешний LB от cloud provider | Продакшен |

---

## Helm — пакетный менеджер K8s

**Chart** — набор шаблонизированных YAML-манифестов.

```
myapp-chart/
├── Chart.yaml          # метаданные (name, version, dependencies)
├── values.yaml         # значения по умолчанию
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── _helpers.tpl    # вспомогательные функции
│   └── hpa.yaml
└── charts/             # dependency charts
```

```yaml
# values.yaml
replicaCount: 3
image:
  repository: myregistry.azurecr.io/myapp
  tag: "1.0.0"
  pullPolicy: IfNotPresent
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"
ingress:
  enabled: true
  host: app.example.com
```

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
```

---

## Service Mesh

**Service Mesh** — выделенный инфраструктурный слой для управления сервис-ту-сервис коммуникацией.

| Задача | Без Mesh | С Mesh (Istio) |
|--------|----------|----------------|
| mTLS | В каждом сервисе | Автоматически (sidecar) |
| Traffic splitting | Nginx/Istio Gateway | VirtualService + DestinationRule |
| Observability | Сбор вручную | Автоматические метрики + трейсы |
| Retry / Timeout | В коде | На уровне конфига |

### Istio

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp
  http:
  - match:
    - headers:
        x-canary:
          exact: "true"
    route:
    - destination:
        host: myapp
        subset: v2
      weight: 100
  - route:
    - destination:
        host: myapp
        subset: v1
      weight: 90
    - destination:
        host: myapp
        subset: v2
      weight: 10
```

---

## Container Security

| Уровень | Практики |
|---------|----------|
| **Image** | Scan уязвимостей (Trivy, Snyk), minimal base (distroless), подпись (Cosign) |
| **Runtime** | Read-only rootfs, drop capabilities, seccomp, AppArmor |
| **Network** | NetworkPolicies, mTLS, egress filtering |
| **Registry** | Private registry, image pull secrets, vulnerability scanning |

```yaml
# Пример Pod Security Context
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  capabilities:
    drop: ["ALL"]
  readOnlyRootFilesystem: true
```

---

## Чек-лист

- [ ] Все образы версионированы (semver) + digest
- [ ] Multi-stage build — минимальный размер образа
- [ ] Non-root пользователь в контейнере
- [ ] Healthcheck / liveness + readiness probes
- [ ] Resource requests + limits заданы для всех контейнеров
- [ ] PodDisruptionBudget настроен
- [ ] NetworkPolicy ограничивает трафик
- [ ] Secrets не в values.yaml, а через External Secrets / Vault
- [ ] Установлены HorizontalPodAutoscaler
- [ ] Настроен pod anti-affinity для HA
- [ ] Логи собираются в централизованное хранилище
- [ ] Метрики + алерты настроены

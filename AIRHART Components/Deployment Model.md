# Deployment Model

> Container-based deployment architecture with Kubernetes orchestration, service mesh, and infrastructure scaling

## Overview

AIRHART uses **containerized microservices** deployed on **Kubernetes** clusters with **Rancher** management. The deployment model emphasizes high availability, horizontal scaling, and infrastructure as code.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ AIRHART Deployment Architecture                             │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐ │
│  │ Azure Cloud / On-Premises Kubernetes Cluster         │ │
│  │                                                       │ │
│  │  ┌─────────────┐  ┌──────────────┐  ┌─────────────┐│ │
│  │  │ Namespace:  │  │ Namespace:   │  │ Namespace:  ││ │
│  │  │ aodb        │  │ querystream  │  │ adapters    ││ │
│  │  │             │  │              │  │             ││ │
│  │  │ - API       │  │ - API        │  │ - Generic   ││ │
│  │  │ - Worker    │  │ - SignalR    │  │ - Atlantis  ││ │
│  │  │ - DB        │  │ - Projection │  │ - SITA      ││ │
│  │  └─────────────┘  └──────────────┘  └─────────────┘│ │
│  │                                                       │ │
│  │  Shared Services:                                    │ │
│  │  ┌──────────┐ ┌──────────┐ ┌───────────────┐       │ │
│  │  │ Kafka    │ │ Redis    │ │ PostgreSQL    │       │ │
│  │  │ (3 nodes)│ │ (Cluster)│ │ (Primary+     │       │ │
│  │  │          │ │          │ │  Replicas)    │       │ │
│  │  └──────────┘ └──────────┘ └───────────────┘       │ │
│  └───────────────────────────────────────────────────────┘ │
│                                                             │
│  Ingress Layer:                                             │
│  ┌──────────────────────────────────────────────────────┐ │
│  │ NGINX Ingress Controller → TLS Termination           │ │
│  │ - airhart-api.example.com → AODB API                │ │
│  │ - querystream.example.com → QueryStream API         │ │
│  └──────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## Kubernetes Structure

### Namespaces

```yaml
# AODB namespace
apiVersion: v1
kind: Namespace
metadata:
  name: aodb
  labels:
    component: write-side
    monitoring: enabled

---
# QueryStream namespace
apiVersion: v1
kind: Namespace
metadata:
  name: querystream
  labels:
    component: read-side
    monitoring: enabled

---
# Adapters namespace
apiVersion: v1
kind: Namespace
metadata:
  name: adapters
  labels:
    component: integration
    monitoring: enabled
```

### AODB Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aodb-api
  namespace: aodb
spec:
  replicas: 3  # Horizontal scaling
  selector:
    matchLabels:
      app: aodb-api
  template:
    metadata:
      labels:
        app: aodb-api
        version: v1.5.2
    spec:
      containers:
      - name: aodb-api
        image: airhart.azurecr.io/aodb-api:1.5.2
        ports:
        - containerPort: 8080
          name: http
        - containerPort: 9090
          name: metrics
        env:
        - name: ASPNETCORE_ENVIRONMENT
          value: "Production"
        - name: ConnectionStrings__PostgreSQL
          valueFrom:
            secretKeyRef:
              name: aodb-secrets
              key: db-connection-string
        - name: Kafka__BootstrapServers
          value: "kafka-0.kafka-headless:9092,kafka-1.kafka-headless:9092,kafka-2.kafka-headless:9092"
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "2000m"
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        volumeMounts:
        - name: config
          mountPath: /app/config
          readOnly: true
      volumes:
      - name: config
        configMap:
          name: aodb-config

---
apiVersion: v1
kind: Service
metadata:
  name: aodb-api
  namespace: aodb
spec:
  selector:
    app: aodb-api
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: metrics
    port: 9090
    targetPort: 9090
  type: ClusterIP
```

### QueryStream Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: querystream-api
  namespace: querystream
spec:
  replicas: 5  # More replicas for read-heavy workload
  selector:
    matchLabels:
      app: querystream-api
  template:
    metadata:
      labels:
        app: querystream-api
    spec:
      containers:
      - name: querystream-api
        image: airhart.azurecr.io/querystream-api:1.5.2
        ports:
        - containerPort: 8080
          name: http
        - containerPort: 8081
          name: signalr
        env:
        - name: DocumentDb__ConnectionString
          valueFrom:
            secretKeyRef:
              name: querystream-secrets
              key: documentdb-connection
        - name: Redis__ConnectionString
          valueFrom:
            secretKeyRef:
              name: querystream-secrets
              key: redis-connection
        resources:
          requests:
            memory: "1Gi"
            cpu: "1000m"
          limits:
            memory: "4Gi"
            cpu: "3000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10

---
apiVersion: v1
kind: Service
metadata:
  name: querystream-api
  namespace: querystream
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
spec:
  selector:
    app: querystream-api
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: signalr
    port: 8081
    targetPort: 8081
  sessionAffinity: ClientIP  # Required for SignalR
  type: LoadBalancer
```

### Adapter Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: adapter-atlantis
  namespace: adapters
spec:
  replicas: 2
  selector:
    matchLabels:
      app: adapter-atlantis
  template:
    metadata:
      labels:
        app: adapter-atlantis
    spec:
      containers:
      - name: adapter
        image: airhart.azurecr.io/adapter-atlantis:1.3.0
        env:
        - name: Atlantis__Endpoint
          value: "https://atlantis.external.com/api"
        - name: Atlantis__ApiKey
          valueFrom:
            secretKeyRef:
              name: atlantis-secrets
              key: api-key
        - name: Kafka__BootstrapServers
          value: "kafka-headless:9092"
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
```

## Ingress Configuration

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: airhart-ingress
  namespace: aodb
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "100"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - airhart-api.example.com
    secretName: airhart-api-tls
  rules:
  - host: airhart-api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: aodb-api
            port:
              number: 80

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: querystream-ingress
  namespace: querystream
  annotations:
    nginx.ingress.kubernetes.io/websocket-services: "querystream-api"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - querystream.example.com
    secretName: querystream-tls
  rules:
  - host: querystream.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: querystream-api
            port:
              number: 80
```

## ConfigMaps & Secrets

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aodb-config
  namespace: aodb
data:
  appsettings.json: |
    {
      "Logging": {
        "LogLevel": {
          "Default": "Information",
          "Microsoft": "Warning"
        }
      },
      "BusinessRules": {
        "EnableFlightLegEnrichment": true,
        "EnableVisitEnrichment": true
      },
      "Kafka": {
        "ProducerConfig": {
          "Acks": "All",
          "EnableIdempotence": true,
          "MaxInFlight": 5
        }
      }
    }

---
apiVersion: v1
kind: Secret
metadata:
  name: aodb-secrets
  namespace: aodb
type: Opaque
stringData:
  db-connection-string: "Host=postgres-primary;Database=aodb;Username=aodb_user;Password=<PASSWORD>"
```

## Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: aodb-api-hpa
  namespace: aodb
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: aodb-api
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 minutes before scaling down
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0  # Scale up immediately
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: querystream-api-hpa
  namespace: querystream
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: querystream-api
  minReplicas: 5
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  - type: Pods
    pods:
      metric:
        name: signalr_connections
      target:
        type: AverageValue
        averageValue: "1000"  # Scale at 1000 connections per pod
```

## Persistent Storage

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: kafka-data-0
  namespace: kafka
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 500Gi
  storageClassName: premium-ssd

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
  namespace: aodb
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Ti
  storageClassName: premium-ssd
```

## Infrastructure as Code

### Terraform (Azure)

```hcl
# Azure Kubernetes Service
resource "azurerm_kubernetes_cluster" "airhart" {
  name                = "airhart-aks"
  location            = azurerm_resource_group.airhart.location
  resource_group_name = azurerm_resource_group.airhart.name
  dns_prefix          = "airhart"
  kubernetes_version  = "1.28.3"

  default_node_pool {
    name                = "default"
    node_count          = 5
    vm_size             = "Standard_D8s_v3"
    enable_auto_scaling = true
    min_count           = 3
    max_count           = 10
    os_disk_size_gb     = 100
    type                = "VirtualMachineScaleSets"
  }

  identity {
    type = "SystemAssigned"
  }

  network_profile {
    network_plugin    = "azure"
    load_balancer_sku = "standard"
  }

  tags = {
    Environment = "Production"
    Component   = "AIRHART"
  }
}

# Azure Container Registry
resource "azurerm_container_registry" "airhart" {
  name                = "airhartacr"
  resource_group_name = azurerm_resource_group.airhart.name
  location            = azurerm_resource_group.airhart.location
  sku                 = "Premium"
  admin_enabled       = false

  georeplications {
    location = "West Europe"
  }
}

# PostgreSQL Flexible Server
resource "azurerm_postgresql_flexible_server" "aodb" {
  name                   = "airhart-postgres"
  resource_group_name    = azurerm_resource_group.airhart.name
  location               = azurerm_resource_group.airhart.location
  version                = "15"
  administrator_login    = "aodb_admin"
  administrator_password = var.postgres_password
  storage_mb             = 1048576  # 1 TB
  sku_name              = "GP_Standard_D8s_v3"

  high_availability {
    mode                      = "ZoneRedundant"
    standby_availability_zone = "2"
  }

  backup_retention_days = 35
}
```

### Helm Charts

```yaml
# values.yaml for AODB Helm chart
replicaCount: 3

image:
  repository: airhart.azurecr.io/aodb-api
  tag: "1.5.2"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: airhart-api.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: airhart-api-tls
      hosts:
        - airhart-api.example.com

resources:
  requests:
    memory: "512Mi"
    cpu: "500m"
  limits:
    memory: "2Gi"
    cpu: "2000m"

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

postgresql:
  enabled: false
  external:
    host: airhart-postgres.postgres.database.azure.com
    database: aodb
    username: aodb_user

kafka:
  enabled: false
  external:
    bootstrapServers: "kafka-0:9092,kafka-1:9092,kafka-2:9092"
```

## CI/CD Pipeline

```yaml
# Azure DevOps Pipeline
trigger:
  branches:
    include:
    - main
    - develop

pool:
  vmImage: 'ubuntu-latest'

stages:
- stage: Build
  jobs:
  - job: BuildAndPush
    steps:
    - task: Docker@2
      inputs:
        command: buildAndPush
        repository: airhart.azurecr.io/aodb-api
        tags: |
          $(Build.BuildId)
          latest

- stage: DeployDev
  dependsOn: Build
  condition: eq(variables['Build.SourceBranch'], 'refs/heads/develop')
  jobs:
  - deployment: DeployToKubernetes
    environment: 'dev'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: KubernetesManifest@0
            inputs:
              action: deploy
              manifests: |
                k8s/aodb-deployment.yaml
                k8s/aodb-service.yaml
              containers: |
                airhart.azurecr.io/aodb-api:$(Build.BuildId)

- stage: DeployProd
  dependsOn: Build
  condition: eq(variables['Build.SourceBranch'], 'refs/heads/main')
  jobs:
  - deployment: DeployToProduction
    environment: 'production'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: HelmDeploy@0
            inputs:
              command: upgrade
              chartType: FilePath
              chartPath: helm/aodb
              releaseName: aodb
              overrideValues: 'image.tag=$(Build.BuildId)'
```

## Monitoring & Observability

```yaml
# Prometheus ServiceMonitor
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: aodb-api
  namespace: aodb
spec:
  selector:
    matchLabels:
      app: aodb-api
  endpoints:
  - port: metrics
    path: /metrics
    interval: 30s

---
# Grafana Dashboard ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: aodb-dashboard
  namespace: monitoring
data:
  aodb-overview.json: |
    {
      "dashboard": {
        "title": "AODB Overview",
        "panels": [
          {
            "title": "Request Rate",
            "targets": [
              {
                "expr": "rate(http_requests_total{namespace=\"aodb\"}[5m])"
              }
            ]
          }
        ]
      }
    }
```

## Related Patterns

- **[[Design Principles]]** - Architectural principles
- **Microservices Pattern** - Service decomposition
- **Circuit Breaker Pattern** - Resilience
- **Blue-Green Deployment** - Zero-downtime deployments

---

**Tags:** #airhart #deployment #kubernetes #infrastructure #devops
**Created:** 2025-12-15
**Related:** [[Design Principles]], [[AODB Pipeline]], [[QueryStream]]

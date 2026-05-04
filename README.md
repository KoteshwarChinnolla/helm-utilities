# helm-utilities

A collection of production-ready, reusable Helm charts for deploying microservices and their supporting infrastructure on Kubernetes. Designed for plug-and-play usage — just configure your `values.yaml` and deploy.

---

## 📦 Repository Structure

```
helm-utilities/
├── Any_service/             # Generic microservice chart (works for any service)
├── certified_gateway/       # Istio Gateway + TLS certificate management
├── kafka-helm-chart/        # Apache Kafka (KRaft mode) + Kafka UI
├── redis/                   # Redis with Sentinel HA setup
├── storage-class/           # StorageClass definitions
├── Argo_CD/                 # ArgoCD GitOps application manifest
└── charts/                  # Packaged .tgz chart files + Helm index
```

---

## 🚀 Charts Overview

### 1. `Any_service` — Generic Microservice Chart

The most flexible chart in this repo. Designed to deploy **any backend microservice** without modifying templates — just supply your own `values.yaml`.

**What it provisions:**
- `Deployment` with configurable replicas, image, resources, probes, node selectors, and tolerations
- `Service` (NodePort or ClusterIP)
- `HorizontalPodAutoscaler` (HPA) for autoscaling
- `ServiceAccount` with custom RBAC rules
- `ConfigMap` for environment variables
- `HTTPRoute` for Istio Gateway API traffic routing (with retries and timeouts)
- `ArgoCD Application` manifest for GitOps-based continuous deployment
- `Namespace` creation

**Key features:**
- Plain env vars (`env`) and secret-backed env vars (`envFromSecret`) are both supported
- Full liveness and readiness probe configuration
- Gateway `HTTPRoute` with retry policy, timeout, and hostname binding
- ArgoCD integration pointing to your Helm chart repo and microservice repo
- Node selector and toleration support for mixed-arch clusters

**Example `values.yaml`:**

```yaml
replicaCount: 1

microservice:
  name: employee
  namespace: hrms
  port: 8090

image:
  repository: 208940303379.dkr.ecr.ap-south-2.amazonaws.com/hrms/employee
  pullPolicy: IfNotPresent
  tag: "v1.3.9"

env:
  SPRING_SERVLET_MULTIPART_MAX_REQUEST_SIZE: 100MB
  CLOUD_AWS_REGION_STATIC: ap-south-2
  CLOUD_AWS_S3_BUCKET: techlifeprofiles

envFromSecret:
  SPRING_DATASOURCE_URL:
    name: postgres-credentials
    key: POSTGRES_URL
  SPRING_DATASOURCE_USERNAME:
    name: postgres-credentials
    key: POSTGRES_USER
  SPRING_DATASOURCE_PASSWORD:
    name: postgres-credentials
    key: POSTGRES_PASSWORD
  CLOUD_AWS_CREDENTIALS_ACCESSKEY:
    name: employee-aws-access
    key: aws-access-key
  CLOUD_AWS_CREDENTIALS_SECRETKEY:
    name: employee-aws-access
    key: aws-secret-key

nodeSelector:
  kubernetes.io/arch: amd64

tolerations:
  - key: "arch"
    operator: "Equal"
    value: "x86"
    effect: "PreferNoSchedule"

resources:
  limits:
    cpu: 1000m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 128Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

livenessProbe:
  path: "actuator/health/liveness"
  initialDelaySeconds: 180
  periodSeconds: 10

readinessProbe:
  path: "actuator/health/readiness"
  initialDelaySeconds: 150
  periodSeconds: 10

service:
  type: NodePort
  port: 80
  nodePort: 30081

gateway_service:
  namespace: istio-gateway

httprouts:
  hostnames: ["hrms.anasolconsultancyservices.com"]
  path: "/api/employee"
  timeout_request: "2s"
  retry_codes: [500, 502, 503, 504]
  retry_attempts: 3
  retry_backoff: "1s"

serviceAccount:
  annotations:
    example.com/annotation: "value"
  rules:
    - apiGroups: [""]
      resources: ["pods", "endpoints", "services", "secrets", "configmaps"]
      verbs: ["get", "list", "watch"]

argo_cd:
  helm_url: "https://koteshwarchinnolla.github.io/helm-utilities/charts"
  chart_name: hrms-microservices
  chart_version: 0.1.0
  path_to_values: employee/values.yaml
  microservice_repo_url: "https://github.com/KoteshwarChinnolla/TECHLIFE-BACKEND"
  branch: main
  destination_server: https://kubernetes.default.svc
```

**Deploy:**
```bash
helm upgrade --install employee ./Any_service \
  -f employee/values.yaml \
  --namespace hrms --create-namespace
```

---

### 2. `certified_gateway` — Istio Gateway + TLS Certificate

Manages your Istio Gateway API entry point and automates TLS certificate issuance via cert-manager and Let's Encrypt (DNS-01 challenge using AWS Route53).

**What it provisions:**
- `Gateway` resource (Istio Gateway API) with configurable HTTP, HTTPS, and TCP listeners
- `Certificate` resource (cert-manager) for wildcard or specific domain TLS
- `ClusterIssuer` backed by Let's Encrypt ACME with Route53 DNS validation

**Key features:**
- Toggle gateway and certificate on/off independently via `enabled` flags
- Multi-protocol listeners: HTTP (80), HTTPS (443), TCP (PostgreSQL 5432, Redis 6378)
- Wildcard certificate support (e.g., `*.yourdomain.com`)
- Route53 credentials managed via Kubernetes Secret

**Example `values.yaml`:**

```yaml
project:
  name: hrms

gateway_service:
  enabled: true
  namespace: istio-gateway
  name: gateway
  hostname: "hrms.anasolconsultancyservices.com"
  isthio_injection: "disable"
  listeners:
    - name: hrms-http-listener
      hostname: "hrms.anasolconsultancyservices.com"
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: All
    - name: hrms-https-listener
      hostname: "hrms.anasolconsultancyservices.com"
      port: 443
      protocol: HTTPS
      tls:
        certificateRefs:
          - kind: Secret
            name: cirtificate-hrms-secret
      allowedRoutes:
        namespaces:
          from: All
    - name: postgress-listener
      port: 5432
      protocol: TCP
      allowedRoutes:
        namespaces:
          from: All
    - name: redis-listener
      port: 6378
      protocol: TCP
      allowedRoutes:
        namespaces:
          from: All

certificate:
  enabled: false
  name: certificate-hrms
  secretName: cirtificate-hrms-secret
  dnsNames:
    - "*.anasolconsultancyservices.com"
  email: chinnollakoteshwar@gmail.com
  issuer:
    name: letsencrypt
    kind: ClusterIssuer
    server: https://acme-v02.api.letsencrypt.org/directory
  route53:
    region: us-east-1
    secretName: route53-secret
    accessKeyIdKey: access-key-id
    secretAccessKeyKey: secret-access-key
```

**Deploy:**
```bash
helm upgrade --install gateway ./certified_gateway \
  -f gateway/values.yaml \
  --namespace istio-gateway --create-namespace
```

---

### 3. `kafka-helm-chart` — Apache Kafka (KRaft Mode)

Deploys a production-style Kafka cluster using **KRaft mode** (no ZooKeeper), with separate controller and broker node pools and a Kafka UI for visual management.

**What it provisions:**
- Kafka **controller** StatefulSet (metadata management)
- Kafka **broker** StatefulSet (data handling, configurable replica count)
- ConfigMap with Kafka configuration
- Init script ConfigMap for cluster setup
- `ServiceAccount` for Kafka pods
- Kafka UI deployment with NodePort access

**Key features:**
- Independent scaling of controllers and brokers
- Persistent volume support via configurable StorageClass and sizes
- Kafka UI accessible via NodePort for easy cluster inspection
- KRaft mode — no ZooKeeper dependency

**Example `values.yaml`:**

```yaml
controllerReplicas: 1
brokerReplicas: 3

storageClassName: standard
storageSize_controller: 1Gi
storageSize_broker: 5Gi

serviceAccountName: kafka-service-account
serviceAccount:
  automount: true
  annotations: {}

kafka_ui:
  replicas: 1
  resources:
    requests:
      memory: "124Mi"
      cpu: "100m"
    limits:
      memory: "500Mi"
      cpu: "500m"
  service:
    port: 9094
    targetPort: 8080
    nodePort: 30004
```

**Deploy:**
```bash
helm upgrade --install kafka ./kafka-helm-chart \
  -f kafka/values.yaml \
  --namespace kafka --create-namespace
```

---

### 4. `redis` — Redis with Sentinel High Availability

Deploys a Redis cluster in **master-replica + Sentinel** topology for automatic failover and high availability.

**What it provisions:**
- Redis **replica** StatefulSet (1 master + N replicas via Redis replication)
- Redis **Sentinel** StatefulSet for leader election and failover
- ConfigMap-based Redis configuration per node
- Persistent volumes for both replicas and sentinels
- `ServiceAccount` for Redis pods
- Services for Redis client access and Sentinel coordination

**Key features:**
- Configurable replica count with minimum-replicas-to-write safety
- Password authentication for both master and replica connections
- AOF (append-only file) persistence enabled
- Separate storage sizing for replica and sentinel nodes
- Sentinel quorum configurable (default: 2)

**Example `values.yaml`:**

```yaml
storageClass: "gp2"

serviceAccount:
  create: true
  automount: true
  annotations: {}
  name: "redis-service-account"

replica:
  name: redis
  count: 3
  image: redis:7.4-alpine
  port: 6379
  replicaStorage: 1Gi
  resources:
    requests:
      memory: "64Mi"
      cpu: "250m"
    limits:
      memory: "128Mi"
      cpu: "500m"
  Config:
    bind: "0.0.0.0"
    protected-mode: "no"
    port: 6379
    dir: "/data"
    masterauth: "RedisPasswordMaster"
    min-replicas-to-write: 2
    requirepass: "RedisPasswordMaster"
    appendonly: "yes"

masterPassword: RedisPasswordMaster

sentinel:
  name: sentinel
  image: redis:7.4-alpine
  port: 5000
  quorum: 2
  sentinelStorage: 1Gi
```

**Deploy:**
```bash
helm upgrade --install redis ./redis \
  -f redis/values.yaml \
  --namespace redis --create-namespace
```

**Uninstall:**
```bash
helm uninstall redis --namespace hrms
```

---

### 5. `storage-class` — StorageClass

Defines the Kubernetes `StorageClass` used by Kafka and Redis for persistent volume claims. Apply this before deploying stateful workloads.

```bash
kubectl apply -f storage-class/storageclass.yaml
```

---

### 6. `Argo_CD` — GitOps Application Manifest

Contains a pre-built ArgoCD `Application` manifest (`helm-chart-update.yaml`) that wires your Helm chart repo to your microservice source repo for automated GitOps deployments. The same configuration is also embedded into each microservice via the `argo_cd` block in `Any_service/values.yaml`.

---

## 🗂️ Packaged Charts (Helm Repo)

The `charts/` directory serves as a **Helm chart repository**, hosted at:

```
https://koteshwarchinnolla.github.io/helm-utilities/charts
```

| Chart | Version | Description |
|---|---|---|
| `hrms-microservices` | 0.1.0 | Generic microservice chart (Any_service) |
| `certified_gateway` | 0.1.1 | Istio gateway + TLS certificate |
| `kafka-helm-chart` | 0.1.0 | Kafka KRaft cluster + UI |
| `redis` | 0.1.0 | Redis Sentinel HA |

Add the repo:
```bash
helm repo add helm-utilities https://koteshwarchinnolla.github.io/helm-utilities/charts
helm repo update
```

Install directly from the repo:
```bash
helm install employee helm-utilities/hrms-microservices -f values.yaml
```

---

## ⚙️ Prerequisites

| Tool | Purpose |
|---|---|
| Kubernetes 1.25+ | Target cluster |
| Helm 3.x | Chart installation |
| Istio (Gateway API CRDs) | For `Any_service` HTTPRoute and `certified_gateway` |
| cert-manager | For TLS certificate issuance in `certified_gateway` |
| ArgoCD | For GitOps deployment via `Argo_CD` manifest |
| AWS Route53 access | For DNS-01 TLS challenge in `certified_gateway` |

---

## 🔐 Secrets Management

This repo expects certain Kubernetes Secrets to exist in the cluster before deploying:

| Secret Name | Used By | Keys |
|---|---|---|
| `postgres-credentials` | Any_service | `POSTGRES_URL`, `POSTGRES_USER`, `POSTGRES_PASSWORD` |
| `employee-aws-access` | Any_service | `aws-access-key`, `aws-secret-key` |
| `route53-secret` | certified_gateway | `access-key-id`, `secret-access-key` |

Create secrets manually or via your secrets manager (e.g., AWS Secrets Manager, External Secrets Operator) before running helm install.

---

## 🌐 Cluster Setup (kind)

A `kind-config.yaml` is included for local development clusters:

```bash
kind create cluster --config kind-config.yaml
```

---

## 📄 License

See [LICENSE](./LICENSE) for details.

---

## 👤 Author

**Koteshwar Chinnolla**
- GitHub: [KoteshwarChinnolla](https://github.com/KoteshwarChinnolla)
- Chart Repo: [helm-utilities](https://koteshwarchinnolla.github.io/helm-utilities/charts)

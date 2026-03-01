# Microservices Observability Stack

[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.27%2B-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![Prometheus](https://img.shields.io/badge/Prometheus-2.45%2B-E6522C?logo=prometheus&logoColor=white)](https://prometheus.io/)
[![Grafana](https://img.shields.io/badge/Grafana-10.0%2B-F46800?logo=grafana&logoColor=white)](https://grafana.com/)
[![Loki](https://img.shields.io/badge/Loki-2.9%2B-F5A623?logo=grafana&logoColor=white)](https://grafana.com/oss/loki/)
[![Go](https://img.shields.io/badge/Go-1.21-00ADD8?logo=go&logoColor=white)](https://go.dev/)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

A production-grade Kubernetes observability stack built on top of Google's [microservices-demo](https://github.com/GoogleCloudPlatform/microservices-demo). Provides full-stack visibility — metrics, logs, and dashboards — with custom application instrumentation for the `productcatalogservice`.

---

## Table of Contents

- [Architecture](#architecture)
- [Key Features](#key-features)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Production Deployment](#production-deployment)
- [Custom Instrumentation](#custom-instrumentation)
- [Dashboard](#dashboard)
- [Verification](#verification)
- [Configuration Reference](#configuration-reference)
- [Troubleshooting](#troubleshooting)
- [Security Considerations](#security-considerations)
- [Contributing](#contributing)
- [License](#license)

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                    │
│                                                         │
│  ┌──────────────── hipster-shop namespace ───────────┐  │
│  │  productcatalogservice (:8080 gRPC, :8888 metrics) │  │
│  │  + 10 other microservices (frontend, cart, …)     │  │
│  └────────────────────────────────────────────────────┘  │
│                          │ scrape /metrics               │
│  ┌──────────────── monitoring namespace ─────────────┐  │
│  │  Prometheus  ◄── ServiceMonitor (30s interval)    │  │
│  │  Grafana     ←── dashboards (drdroid-dashboard)   │  │
│  │  Loki        ←── Promtail (pod log shipping)      │  │
│  └────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

| Layer | Component | Role |
|-------|-----------|------|
| Orchestration | Kubernetes (Kind) | Container scheduling & service discovery |
| Application | microservices-demo (Hipster Shop) | 11-service e-commerce workload |
| Metrics | Prometheus + kube-prometheus-stack | Collection, storage, alerting |
| Visualization | Grafana | Dashboards & alerting UI |
| Logs | Loki + Promtail | Log aggregation & querying |
| Instrumentation | Go `promhttp` in productcatalogservice | Custom process & runtime metrics |

---

## Key Features

- **Custom Application Metrics** — Process memory, goroutines, and GC pause times from `productcatalogservice` via a dedicated `/metrics` endpoint.
- **Kubernetes Infrastructure Metrics** — Pod CPU/memory and node-level metrics via `kube-prometheus-stack`.
- **Centralized Logging** — Structured log aggregation from all pods via Loki + Promtail.
- **Pre-built Grafana Dashboard** — 7-panel dashboard covering application, infrastructure, and log observability (importable from `drdroid-dashboard.json`).
- **Automatic Service Discovery** — Prometheus discovers the metrics endpoint through a `ServiceMonitor` CRD — no static scrape config required.

---

## Prerequisites

Ensure the following tools are installed and available in your `PATH` before proceeding:

| Tool | Minimum Version | Install |
|------|----------------|---------|
| Docker | 20.10+ | [docs.docker.com](https://docs.docker.com/get-docker/) |
| kubectl | 1.27+ | [kubernetes.io](https://kubernetes.io/docs/tasks/tools/) |
| Kind | 0.20+ | [kind.sigs.k8s.io](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) |
| Helm | 3.12+ | [helm.sh](https://helm.sh/docs/intro/install/) |

> **Note:** Any CNCF-conformant Kubernetes cluster (EKS, GKE, AKS, k3s, minikube) can be used in place of Kind.

---

## Quick Start

### 1. Create a local cluster

```bash
kind create cluster --name observability
kubectl config use-context kind-observability
```

### 2. Create namespaces

```bash
kubectl create namespace hipster-shop
kubectl create namespace monitoring
```

### 3. Deploy the microservices

```bash
# Clone Google's microservices-demo if not already present
git clone https://github.com/GoogleCloudPlatform/microservices-demo.git

kubectl apply -f microservices-demo/release/kubernetes-manifests.yaml -n hipster-shop
```

### 4. Add Helm repositories

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

### 5. Install the monitoring stack

```bash
# kube-prometheus-stack (Prometheus + Grafana + Alertmanager)
helm install kps prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.adminPassword="${GRAFANA_ADMIN_PASSWORD:-changeme}" \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false

# Loki log aggregation (reuses existing Grafana instance)
helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set grafana.enabled=false \
  --set promtail.enabled=true
```

### 6. Apply the ServiceMonitor

```bash
kubectl apply -f productcatalogservice-servicemonitor.yaml
```

### 7. Import the Grafana dashboard

```bash
# Port-forward Grafana
kubectl port-forward svc/kps-grafana 3000:80 -n monitoring &

# Import dashboard via API
curl -s -u "admin:${GRAFANA_ADMIN_PASSWORD:-changeme}" \
  -X POST http://localhost:3000/api/dashboards/import \
  -H "Content-Type: application/json" \
  -d "{\"dashboard\": $(cat drdroid-dashboard.json), \"overwrite\": true, \"folderId\": 0}"
```

### 8. Access Grafana

Open [http://localhost:3000](http://localhost:3000) and log in with `admin` / `<GRAFANA_ADMIN_PASSWORD>`.

---

## Production Deployment

### Resource Limits

Always set resource `requests` and `limits` for monitoring components to protect cluster stability:

```bash
helm upgrade kps prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.resources.requests.cpu=200m \
  --set prometheus.prometheusSpec.resources.requests.memory=512Mi \
  --set prometheus.prometheusSpec.resources.limits.cpu=1000m \
  --set prometheus.prometheusSpec.resources.limits.memory=2Gi \
  --set grafana.resources.requests.cpu=100m \
  --set grafana.resources.requests.memory=128Mi
```

### Persistent Storage

Enable persistent volumes so metrics and dashboards survive pod restarts:

```bash
helm upgrade kps prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=20Gi \
  --set grafana.persistence.enabled=true \
  --set grafana.persistence.size=5Gi
```

### High Availability

For production clusters, run multiple replicas of Grafana and use Thanos or Cortex for Prometheus HA:

```bash
helm upgrade kps prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.replicas=2
```

### TLS / Ingress

Expose Grafana behind an Ingress controller with TLS rather than using `kubectl port-forward`:

```yaml
# Example Ingress (cert-manager + nginx-ingress)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana
  namespace: monitoring
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts: [grafana.example.com]
      secretName: grafana-tls
  rules:
    - host: grafana.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: kps-grafana
                port:
                  number: 80
```

### Alerting

Configure Alertmanager to route critical alerts to your notification channel (Slack, PagerDuty, email):

```bash
helm upgrade kps prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set alertmanager.config.global.slack_api_url="https://hooks.slack.com/services/YOUR/WEBHOOK"
```

---

## Custom Instrumentation

### productcatalogservice metrics endpoint

`productcatalogservice` exposes Prometheus metrics on port `8888` via a lightweight HTTP server added to `server.go`:

```go
go func() {
    http.Handle("/metrics", promhttp.Handler())
    log.Infof("Starting metrics server on :8888")
    if err := http.ListenAndServe(":8888", nil); err != nil {
        log.Errorf("Metrics server error: %v", err)
    }
}()
```

**Exposed metrics:**

| Metric | Type | Description |
|--------|------|-------------|
| `process_resident_memory_bytes` | Gauge | Resident set size in bytes |
| `process_virtual_memory_bytes` | Gauge | Virtual memory size in bytes |
| `go_goroutines` | Gauge | Number of active goroutines |
| `go_gc_duration_seconds` | Summary | GC stop-the-world pause durations |
| `go_threads` | Gauge | Number of OS threads created |

### ServiceMonitor

The `productcatalogservice-servicemonitor.yaml` file registers the metrics endpoint with Prometheus Operator so that Prometheus automatically discovers and scrapes it every 30 seconds — no manual scrape config required.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: productcatalogservice-monitor
  namespace: hipster-shop
  labels:
    release: kps          # must match kube-prometheus-stack release name
spec:
  selector:
    matchLabels:
      app: productcatalogservice
  endpoints:
    - port: metrics
      interval: 30s
```

> If your Helm release name differs from `kps`, update the `release` label accordingly.

---

## Dashboard

The `drdroid-dashboard.json` file contains a ready-to-import Grafana dashboard with seven panels:

| # | Panel | Data Source |
|---|-------|-------------|
| 1 | Product Catalog — Resident Memory | Prometheus |
| 2 | Product Catalog — Virtual Memory | Prometheus |
| 3 | Product Catalog — Active Goroutines | Prometheus |
| 4 | Product Catalog — GC 99th Percentile Pause | Prometheus |
| 5 | All Pods — CPU Usage Rate | Prometheus |
| 6 | All Pods — Memory Usage | Prometheus |
| 7 | Application Logs — All Microservices | Loki |

Import via **Grafana UI**: Dashboards → Import → Upload JSON file → select `drdroid-dashboard.json`.

---

## Verification

### Check Prometheus targets

```bash
kubectl port-forward svc/kps-kube-prometheus-stack-prometheus 9090:9090 -n monitoring
```

Open [http://localhost:9090/targets](http://localhost:9090/targets) and verify `productcatalogservice-metrics` shows **UP**.

### Query custom metrics

```promql
# Resident memory
process_resident_memory_bytes{job="productcatalogservice-metrics"}

# GC rate
rate(go_gc_duration_seconds_count{job="productcatalogservice-metrics"}[5m])

# Active goroutines
go_goroutines{job="productcatalogservice-metrics"}
```

### Query logs

```logql
# All hipster-shop logs
{namespace="hipster-shop"}

# Product catalog logs only
{namespace="hipster-shop", pod=~"productcatalogservice-.*"}
```

### Validate metrics endpoint directly

```bash
POD=$(kubectl get pod -n hipster-shop -l app=productcatalogservice -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n hipster-shop "$POD" -- wget -qO- http://localhost:8888/metrics | head -20
```

---

## Configuration Reference

| Parameter | Default | Description |
|-----------|---------|-------------|
| `GRAFANA_ADMIN_PASSWORD` | `changeme` | Grafana admin password (set via env var or Helm value) |
| Prometheus scrape interval | `30s` | Configurable per `ServiceMonitor` `.spec.endpoints[].interval` |
| Prometheus retention | `10d` | Set via `--set prometheus.prometheusSpec.retention=30d` |
| Loki retention | `744h` (31 days) | Set via `--set loki.config.table_manager.retention_period=720h` |
| Metrics port | `8888` | Port exposed by `productcatalogservice` for `/metrics` |

---

## Troubleshooting

### Metrics not appearing in Prometheus

```bash
# 1. Confirm the ServiceMonitor exists
kubectl get servicemonitor -n hipster-shop

# 2. Confirm productcatalogservice pods are running
kubectl get pods -n hipster-shop -l app=productcatalogservice

# 3. Verify the /metrics endpoint is reachable inside the cluster
POD=$(kubectl get pod -n hipster-shop -l app=productcatalogservice -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n hipster-shop "$POD" -- wget -qO- http://localhost:8888/metrics | grep process_

# 4. Check Prometheus is discovering the target
kubectl port-forward svc/kps-kube-prometheus-stack-prometheus 9090:9090 -n monitoring
# Open http://localhost:9090/targets
```

### Logs not appearing in Loki

```bash
# 1. Check Loki and Promtail pods are healthy
kubectl get pods -n monitoring -l app=loki
kubectl get pods -n monitoring -l app=promtail

# 2. Check Promtail logs for shipping errors
kubectl logs -n monitoring -l app=promtail --tail=50

# 3. Verify Loki data source in Grafana
# Grafana → Configuration → Data Sources → Loki → Test
```

### Grafana shows "No data"

- Confirm the Loki data source URL matches the service: `http://loki:3100`
- Verify the Prometheus data source URL: `http://kps-kube-prometheus-stack-prometheus:9090`
- Check that the dashboard time range covers a period when metrics were being generated.

### Kind cluster out of resources

```bash
# Check node resource usage
kubectl top nodes

# Describe a pod that is Pending
kubectl describe pod <pod-name> -n hipster-shop
```

---

## Security Considerations

- **Change default credentials** — replace the default `changeme` Grafana password with a strong, unique secret. Store it in a Kubernetes `Secret` and reference it in Helm:

  ```bash
  # Create the secret first
  kubectl create secret generic grafana-admin-secret \
    --namespace monitoring \
    --from-literal=admin-password="$(openssl rand -base64 24)"

  # Reference the secret in Helm (instead of --set grafana.adminPassword)
  helm install kps prometheus-community/kube-prometheus-stack \
    --namespace monitoring \
    --set grafana.admin.existingSecret=grafana-admin-secret \
    --set grafana.admin.passwordKey=admin-password
  ```
- **Restrict network access** — do not expose Prometheus, Grafana, or Loki directly on public IPs. Use an authenticated Ingress or a VPN.
- **RBAC** — `kube-prometheus-stack` creates the necessary RBAC resources; avoid granting `cluster-admin` to monitoring service accounts.
- **TLS** — terminate TLS at the Ingress layer (see [Production Deployment → TLS / Ingress](#tls--ingress)).
- **Image scanning** — regularly scan all container images for CVEs using tools such as Trivy or Grype.
- **Secrets rotation** — rotate Grafana admin passwords and any webhook tokens on a regular schedule.

---

## Contributing

Contributions are welcome. Please follow these steps:

1. Fork the repository and create a feature branch: `git checkout -b feature/your-change`
2. Make your changes with clear, descriptive commits.
3. Ensure any new Kubernetes manifests pass `kubectl apply --dry-run=client -f <file>`.
4. Open a Pull Request with a clear description of the problem and solution.

---

## Project Structure

```
.
├── drdroid-dashboard.json                  # Grafana dashboard (7 panels)
├── productcatalogservice-servicemonitor.yaml  # Prometheus ServiceMonitor CRD
└── README.md                               # This file
```

---

## Tech Stack

| Component | Purpose | Version |
|-----------|---------|---------|
| Kubernetes | Container orchestration | 1.27+ |
| Kind | Local cluster provisioning | 0.20+ |
| Prometheus | Metrics collection & storage | 2.45+ |
| Grafana | Visualization & dashboards | 10.0+ |
| Loki | Log aggregation | 2.9+ |
| Promtail | Log shipping agent | 2.9+ |
| Go | `productcatalogservice` runtime | 1.21 |
| Helm | Package management | 3.12+ |

---

## License

This project builds on [Google Cloud Platform's microservices-demo](https://github.com/GoogleCloudPlatform/microservices-demo), which is licensed under the [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0).

Additions and modifications in this repository are also released under the Apache License 2.0.

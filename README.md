# Microservices Observability Stack - DrDroid Assignment

A complete Kubernetes-based observability solution for Google's [microservices-demo](https://github.com/GoogleCloudPlatform/microservices-demo) with custom application instrumentation, centralized logging, and real-time dashboards.

## 🎯 Overview

This project demonstrates a production-ready observability stack for microservices:
- **Kubernetes Cluster** - Deployed via Kind (Kubernetes in Docker)
- **microservices-demo** - Hipster Shop microservices across 11 services
- **Prometheus** - Metrics collection and storage
- **Grafana** - Visualization and dashboarding
- **Loki** - Distributed log aggregation
- **Custom Instrumentation** - Process-level metrics from productcatalogservice
- **Load Testing** - k6 traffic generation

## ✨ Key Features

- ✅ **Custom Application Metrics** - Process memory, goroutines, GC metrics from productcatalogservice
- ✅ **Kubernetes Metrics** - Pod CPU/memory, node metrics via Kube-Prometheus
- ✅ **Centralized Logging** - Real-time logs aggregated via Loki
- ✅ **Production Dashboard** - 7-panel Grafana dashboard with all key metrics
- ✅ **Continuous Traffic** - k6 load testing driving realistic metrics

## 📊 Dashboard Contents

**Grafana Dashboard:** `http://localhost:3000/d/18a7bba7-4437-4e84-a564-50fc39199ae1/drdroid-microservices-observability`

### Application Metrics (Custom Instrumentation)
1. Product Catalog - Resident Memory Usage
2. Product Catalog - Virtual Memory Usage  
3. Product Catalog - Active Go Routines
4. Product Catalog - GC 99th Percentile Pause

### Infrastructure Metrics
5. Kubernetes - All Pods CPU Usage Rate
6. Kubernetes - All Pods Memory Usage

### Observability
7. Application Logs - All Microservices (via Loki)

## 🚀 Quick Start

### Prerequisites
- Docker & Docker Compose (or Podman)
- kubectl
- Kind cluster (or any Kubernetes cluster)
- Helm 3

### Setup (5 minutes)

```bash
# 1. Create Kind cluster
kind create cluster --name observability

# 2. Create namespaces
kubectl create namespace hipster-shop
kubectl create namespace monitoring

# 3. Deploy microservices
kubectl apply -f microservices-demo/release/kubernetes-manifests.yaml -n hipster-shop

# 4. Add Helm repos
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# 5. Install monitoring stack
helm install kps prometheus-community/kube-prometheus-stack \
  -n monitoring \
  --set grafana.adminPassword=admin123

helm install loki grafana/loki-stack \
  -n monitoring \
  --set grafana.enabled=false

# 6. Port forward Grafana
kubectl port-forward svc/kps-grafana 3000:80 -n monitoring

# 7. Access dashboard
# URL: http://localhost:3000
# Login: admin / admin123
```

## 🔧 Custom Instrumentation

### productcatalogservice Metrics

The productcatalogservice exposes Prometheus metrics on port 8888:

```go
// In server.go (lines 130-137)
go func() {
    http.Handle("/metrics", promhttp.Handler())
    log.Infof("Starting metrics server on :8888")
    if err := http.ListenAndServe(":8888", nil); err != nil {
        log.Errorf("Metrics server error: %v", err)
    }
}()
```

**Available Metrics:**
- `process_resident_memory_bytes` - Process resident memory
- `process_virtual_memory_bytes` - Process virtual memory  
- `go_goroutines` - Active goroutine count
- `go_gc_duration_seconds` - Garbage collection pause durations
- `go_threads` - Active thread count

### ServiceMonitor Configuration

Prometheus automatically discovers and scrapes metrics via ServiceMonitor CRD at 30-second intervals.

## 📈 Load Testing

Generate realistic traffic using k6:

```bash
# Install k6
# Run load tests
k6 run load-test.js
```

Metrics will be visible in Grafana within seconds.

## 📁 Project Structure

```
.
├── microservices-demo/          # Google's microservices demo
│   ├── src/                     # Service source code
│   │   └── productcatalogservice/   # Custom instrumented service
│   ├── kubernetes-manifests/    # K8s deployment specs
│   └── release/                 # Release manifests
├── drdroid-dashboard.json       # Grafana dashboard definition
├── DASHBOARD_SETUP_GUIDE.md     # Detailed setup documentation
└── README.md                    # This file
```

## 🔍 Verification

### Check Prometheus Targets
```bash
kubectl port-forward svc/kps-kube-prometheus-stack-prometheus 9090:9090 -n monitoring
# Visit http://localhost:9090/targets
# Verify: productcatalogservice-metrics target is UP
```

### Query Custom Metrics
```promql
process_resident_memory_bytes{job="productcatalogservice-metrics"}
rate(go_gc_duration_seconds_count{job="productcatalogservice-metrics"}[5m])
go_goroutines{job="productcatalogservice-metrics"}
```

### Check Logs in Grafana
```logql
{namespace="hipster-shop"}
{pod=~"productcatalogservice-.*"}
```

## 📚 Documentation

- **DASHBOARD_SETUP_GUIDE.md** - Comprehensive setup and troubleshooting
- **drdroid-dashboard.json** - Complete dashboard configuration

## 💻 Tech Stack

| Component | Purpose | Version |
|-----------|---------|---------|
| Kubernetes | Container orchestration | 1.27+ (Kind) |
| Prometheus | Metrics collection | 2.45+ |
| Grafana | Visualization | 10.0+ |
| Loki | Log aggregation | 2.9+ |
| k6 | Load testing | Latest |
| Go | productcatalogservice | 1.21 |

## 🎓 Learning Outcomes

This project demonstrates:
- Kubernetes observability architecture
- Prometheus custom metrics instrumentation
- Grafana dashboard design
- Distributed logging with Loki
- Load testing and metrics correlation
- Production-grade monitoring stack

## ⚙️ Troubleshooting

### Metrics not appearing in Prometheus?
```bash
# Check ServiceMonitor
kubectl get servicemonitor -n monitoring

# Check pod logs
kubectl logs -f deployment/productcatalogservice -n hipster-shop

# Verify metrics endpoint
kubectl exec -it pod/productcatalogservice-xxx -n hipster-shop -- curl localhost:8888/metrics
```

### Logs not showing in Loki?
```bash
# Check Loki pods
kubectl get pods -n monitoring | grep loki

# Verify promtail is running
kubectl get pods -n hipster-shop | grep promtail
```

## 📝 License

Based on [Google Cloud Platform's microservices-demo](https://github.com/GoogleCloudPlatform/microservices-demo)

---

**Submission Date:** 2024  
**Status:** ✅ Complete
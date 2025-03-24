## ✅ **1. Prometheus Configuration Design**  

### **💡 Objective:**  
Prometheus will serve as the primary monitoring solution for the Trade Distribution System, collecting real-time metrics from:  
✅ Redis (RTS, RIS, RTSS, RTRS)  
✅ DuckDB  
✅ ClickHouse  
✅ Backend  
✅ Rust Trade Engine  
✅ System-Level Metrics (CPU, memory, disk usage, network)  

The configuration must support:  
✅ Low latency (scrape intervals as low as 1s)  
✅ High retention (14–30 days)  
✅ Secure access  
✅ Efficient memory and storage usage  

---

### **🔎 Key Configuration Files**  
We will define three key configuration files:  

| File | Purpose | Location |
|-------|---------|----------|
| `prometheus.yml` | Core Prometheus configuration file | `/etc/prometheus/prometheus.yml` |
| `targets.yml` | Defines targets for scraping (Redis, DuckDB, ClickHouse, backend) | `/etc/prometheus/targets.yml` |
| `rules.yml` | Alerting and aggregation rules | `/etc/prometheus/rules.yml` |

---

### **📂 Directory Structure**  
```plaintext
/prometheus
├── prometheus.yml
├── targets.yml
├── rules.yml
├── data/               # Metric storage (TSDB)
└── alerts/             # Alerting rules
```

---

### **1.1 Core Configuration (`prometheus.yml`)**
This file defines how Prometheus scrapes data, handles storage, and retention.  

#### **Basic Setup:**  
```yaml
global:
  scrape_interval: 1s               # Scrape data every 1 second
  evaluation_interval: 1s           # Evaluate rules every 1 second
  external_labels:
    source: "trade-system"          # Adds label to distinguish metrics
    env: "production"               # Environment tag

# Rule files for alerting
rule_files:
  - "/etc/prometheus/rules.yml"

# Storage configuration
storage:
  retention_time: 30d               # Keep data for 30 days
  retention_size: 50GB              # Keep up to 50GB of metrics
  path: "/prometheus/data"

# Scrape configurations
scrape_configs:
  - job_name: "redis"
    scrape_interval: 1s
    static_configs:
      - targets:
          - "redis-trade-state:6379"
          - "redis-instance-state:6380"
          - "redis-signal-stream:6381"
          - "redis-result-stream:6382"
    metrics_path: "/metrics"
    scheme: "http"

  - job_name: "duckdb"
    scrape_interval: 5s
    static_configs:
      - targets: 
          - "localhost:1234"
    metrics_path: "/metrics"
    scheme: "http"

  - job_name: "clickhouse"
    scrape_interval: 5s
    static_configs:
      - targets:
          - "clickhouse-node:8123"
    metrics_path: "/metrics"
    scheme: "http"

  - job_name: "backend"
    scrape_interval: 1s
    static_configs:
      - targets:
          - "backend:8000"
    metrics_path: "/metrics"
    scheme: "http"

  - job_name: "rust-trade-engine"
    scrape_interval: 1s
    static_configs:
      - targets:
          - "trade-engine:9000"
    metrics_path: "/metrics"
    scheme: "http"
```

---

### **1.2 Target Definition (`targets.yml`)**  
Instead of hardcoding targets, we can define them dynamically in a separate file for easier updates.  

```yaml
# Redis targets
- targets:
    - "redis-trade-state:6379"
    - "redis-instance-state:6380"
    - "redis-signal-stream:6381"
    - "redis-result-stream:6382"
  labels:
    service: "redis"

# DuckDB target
- targets:
    - "localhost:1234"
  labels:
    service: "duckdb"

# ClickHouse target
- targets:
    - "clickhouse-node:8123"
  labels:
    service: "clickhouse"

# Backend target
- targets:
    - "backend:8000"
  labels:
    service: "backend"

# Trade Engine target
- targets:
    - "trade-engine:9000"
  labels:
    service: "rust-trade-engine"
```

---

### **1.3 Alert Rules (`rules.yml`)**  
Define alerting thresholds to catch anomalies or failures.  

✅ High Memory Usage: Trigger alert if backend memory > 80%  
✅ Slow Sync Latency: Trigger alert if sync to ClickHouse > 200ms  
✅ Redis Down: Trigger alert if Redis instance stops responding  

```yaml
groups:
  - name: "trade-system"
    interval: 1s
    rules:
      - alert: HighMemoryUsage
        expr: process_resident_memory_bytes > 0.8 * 1024 * 1024 * 1024 # 80% of 1GB
        for: 5s
        labels:
          severity: critical
        annotations:
          summary: "High Memory Usage Detected"
          description: "Memory usage exceeds 80%"

      - alert: SlowSync
        expr: avg_over_time(sync_latency_seconds[10s]) > 0.2
        for: 5s
        labels:
          severity: warning
        annotations:
          summary: "High Sync Latency Detected"
          description: "Sync latency exceeds 200ms"

      - alert: RedisDown
        expr: up{service="redis"} == 0
        for: 5s
        labels:
          severity: critical
        annotations:
          summary: "Redis Instance Down"
          description: "Redis instance is not responding"
```

---

### **1.4 Retention Strategy**  
We need to balance data retention vs. storage size.  

✅ Retain last **30 days** of data  
✅ Limit retention to **50 GB**  
✅ Drop data older than 30 days  
✅ If retention exceeds limit → Drop oldest records first  

---

### **1.5 Scraping Strategy**  
| Service | Scrape Interval | Method | Reason |
|---------|-----------------|--------|--------|
| Redis | 1s | HTTP `/metrics` | High-frequency trade state changes |
| DuckDB | 5s | HTTP `/metrics` | Lower volume, less volatility |
| ClickHouse | 5s | HTTP `/metrics` | Historical, low latency |
| Backend | 1s | HTTP `/metrics` | Low-latency trade sync |
| Rust Trade Engine | 1s | HTTP `/metrics` | High-frequency trade execution |

---

### **1.6 Storage Strategy**  
- **Retention:**  
   ✅ 30 days OR 50GB (whichever comes first)  
- **Compression:**  
   ✅ Snappy compression enabled  
- **Memory Usage:**  
   ✅ Cache size = 500MB  
- **Storage Location:**  
   ✅ `/prometheus/data`  

---

### **1.7 Resource Allocation**  
| Resource | Allocation |
|----------|------------|
| **CPU** | 2 cores |
| **Memory** | 2 GB |
| **Disk** | 50 GB |
| **Scrape Load** | ~50 MB/s |

---

### **1.8 External Monitoring**  
✅ Set up Grafana for Prometheus visualization.  
✅ Create a Grafana dashboard for key trade metrics.  
✅ Add Prometheus health checks to backend monitoring.  

---

### **🚧 Potential Risks**  
| Risk | Mitigation |
|------|------------|
| High scrape load | Increase Prometheus CPU/memory |  
| Storage overflow | Drop oldest data first |  
| Redis overload | Use Redis clustering |  
| Slow scrape response | Increase scrape interval |  

---

### **🛡️ Configuration Best Practices**  
✅ Keep `prometheus.yml` minimal — offload target config to `targets.yml`.  
✅ Group alerts by severity (info, warning, critical).  
✅ Always use labels for service tagging.  
✅ Protect the Prometheus UI using HTTPS and authentication.  

---

## ✅ **Next Step:**  
1. Finalize Prometheus resource allocation.  
2. Write the Docker configuration for Prometheus setup.  
3. Configure Grafana with the correct scrape metrics.  
4. Start testing sync rates and scrape efficiency.  

---

## 🚀 **Shall we proceed to Schema Design next?** 😎
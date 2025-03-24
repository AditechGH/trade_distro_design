## ✅ **Prometheus Architectural Design – High Availability (HA) Design**  
This section outlines the **High Availability (HA)** strategy for Prometheus, focusing on ensuring continuous data collection, processing, and querying even in the event of node failures or network interruptions. 

The goal is to implement a multi-instance, load-balanced, and failover-capable Prometheus setup to support real-time state tracking and trade execution without data loss or downtime.

---

## 🚀 **1. HA Goals**  
The high-availability design will aim to:  
✅ Ensure continuous data collection from Redis, DuckDB, and Trade Engine  
✅ Minimize data loss during node failures  
✅ Distribute the load across multiple Prometheus instances  
✅ Implement failover and data replication for redundancy  
✅ Ensure consistent querying across Prometheus nodes  

---

## 🏆 **2. HA Architecture Overview**  
The HA design will feature the following components:  

### ✅ **2.1. Two Prometheus Nodes**  
- **Prometheus-1** → Primary Prometheus instance  
- **Prometheus-2** → Secondary Prometheus instance (replica)  

Both instances will scrape the same set of targets (Redis, DuckDB, Trade Engine) but only one instance will handle alerts and recording rules.

### ✅ **2.2. Load Balancer**  
A load balancer will distribute requests between Prometheus instances:  
- Ensures traffic is evenly balanced between nodes  
- Directs traffic to the active instance in case of a failure  

### ✅ **2.3. Thanos (Optional)**  
- **Thanos** will be used for long-term storage and horizontal scaling.  
- Thanos will consolidate data from both Prometheus nodes for unified query access.  
- Thanos will provide deduplication across Prometheus replicas.  

---

## 🌐 **3. High Availability Topology**  
```
+-------------------------+              +-------------------------+
|      Prometheus-1       |              |      Prometheus-2       |
|      (Primary)          |              |      (Secondary)        |
|  - Scraping Redis       |              |  - Scraping Redis       |
|  - Scraping DuckDB      |              |  - Scraping DuckDB      |
|  - Scraping Trade Engine|              |  - Scraping Trade Engine|
|  - Alerting Active      |              |  - Alerting Inactive    |
+-------------------------+              +-------------------------+
         |                                      |
         +----------------+    +----------------+
                          |    |
             +------------+    +------------+
             | Load Balancer (HAProxy)       |
             +------------+------------------+
                          |
         +------------------------------------+
         |         Thanos Querier (Optional)  |
         +------------------------------------+
```

---

## 🛠️ **4. Load Balancer Configuration**  
We will use **HAProxy** or **NGINX** to distribute traffic between Prometheus-1 and Prometheus-2:  
✅ HAProxy will route external requests to Prometheus nodes based on availability  
✅ Round-robin strategy for balancing query load  
✅ Sticky sessions enabled to ensure query consistency  

### ✅ **Example HAProxy Config:**  
```haproxy
frontend prometheus
    bind *:9090
    default_backend prometheus_nodes

backend prometheus_nodes
    balance roundrobin
    server prometheus1 10.0.0.1:9090 check
    server prometheus2 10.0.0.2:9090 check
```

---

## 🔄 **5. Data Redundancy Strategy**  
Prometheus does **not support clustering** natively — we’ll achieve redundancy using **parallel scraping** and **Thanos**.  

### ✅ **5.1. Parallel Scraping:**  
Both Prometheus-1 and Prometheus-2 will scrape the same targets (Redis, DuckDB, Trade Engine):  
- Ensures that no data is lost if one Prometheus instance fails.  
- Scraping is lightweight, so parallel scraping adds minimal overhead.  

### ✅ **5.2. WAL Replication:**  
- Prometheus will store transaction logs in the WAL (Write-Ahead Log).  
- WAL data will be synchronized to both instances using shared disk or rsync.  
- On restart, WAL will automatically restore the last state.  

**Example rsync Config:**  
```bash
rsync -avz /var/lib/prometheus/wal prometheus-2:/var/lib/prometheus/
```

---

## 🔥 **6. Thanos for HA and Data Deduplication**  
Thanos will act as the global query layer and storage backend:  
✅ Thanos Querier → Aggregates data from Prometheus-1 and Prometheus-2  
✅ Thanos Sidecar → Attached to each Prometheus instance to sync data to Thanos  
✅ Thanos Store → Stores historical data for long-term access  
✅ Thanos Compactor → Compacts blocks to reduce storage size  

### ✅ **Thanos Config Example:**  
`sidecar.yaml` (for each Prometheus instance):  
```yaml
objstore:
  config:
    bucket: "prometheus-backups"
    endpoint: "https://s3-compatible-storage.local"
    access_key: "ACCESS_KEY"
    secret_key: "SECRET_KEY"
```

---

## 📡 **7. Alerting Redundancy**  
Only **one Prometheus instance** should fire alerts to avoid duplication:  
- Enable alerting on Prometheus-1  
- Disable alerting on Prometheus-2  

**Example Config (on Prometheus-2):**  
```yaml
alerting:
  enabled: false
```

---

## 🔐 **8. Data Consistency Strategy**  
Since Prometheus does not support clustering, data consistency is managed by Thanos:  
✅ Thanos handles data deduplication across instances  
✅ External queries are routed through Thanos for consistent results  
✅ Retain WAL data to ensure state recovery after restart  

---

## 🧠 **9. Health Checks and Failover**  
### ✅ **9.1. Prometheus Instance Health Check:**  
Use the `/-/healthy` endpoint for health checks:  
```bash
curl http://localhost:9090/-/healthy
```

### ✅ **9.2. HAProxy Health Check:**  
Configure HAProxy to check Prometheus health every 2 seconds:  
```haproxy
server prometheus1 10.0.0.1:9090 check interval 2s fall 3 rise 2
```

### ✅ **9.3. Automatic Failover Strategy:**  
| Failure Scenario | Action |  
|------------------|--------|  
| Prometheus-1 down | HAProxy routes traffic to Prometheus-2 |  
| Prometheus-2 down | HAProxy routes traffic to Prometheus-1 |  
| Thanos down | Failover to local Prometheus data |  
| Redis down | Trigger alert → Attempt reconnect |  

---

## 🌍 **10. Scaling Strategy**  
✅ Add more Prometheus nodes → Increase scraping and query capacity  
✅ Horizontal scaling → Add Thanos components for distributed querying  

---

## ⚡ **11. Example Final Config (Summary):**  
**Prometheus Config (on Prometheus-1):**  
```yaml
global:
  scrape_interval: 1s
  evaluation_interval: 1s

storage:
  tsdb:
    retention: 15d
    wal_segment_size: 64MB

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - 'localhost:9093'
```

**Prometheus Config (on Prometheus-2):**  
```yaml
alerting:
  enabled: false
```

**Thanos Sidecar Config:**  
```yaml
objstore:
  config:
    bucket: "prometheus-backups"
    endpoint: "https://s3-compatible-storage.local"
    access_key: "ACCESS_KEY"
    secret_key: "SECRET_KEY"
```

---

## ✅ **12. Edge Case Handling**  
| Failure Type | Handling Strategy |  
|--------------|-------------------|  
| Prometheus Crash | Automatic failover using HAProxy |  
| Thanos Crash | Use local Prometheus data until Thanos is back online |  
| WAL Corruption | Restore from latest snapshot |  
| Load Spike | Increase scrape interval or reduce data volume |  

---

## ✅ **Summary**  
| Component | Status |  
|-----------|--------|  
| Prometheus HA | ✅ Implemented with 2 nodes |  
| Load Balancer | ✅ HAProxy configured |  
| Thanos | ✅ Enabled for deduplication |  
| Alerting Redundancy | ✅ Enabled only on Prometheus-1 |  
| Failover Strategy | ✅ Configured using health checks |  
| Scaling Strategy | ✅ Horizontal scaling supported |  

## ✅ **Prometheus Monitoring Design for ClickHouse**  
Now that we’ve defined the Prometheus schema for Redis and DuckDB, let’s focus on defining the **system and trade metrics** for ClickHouse. Since ClickHouse is handling high-throughput trade data and analytical queries, monitoring will cover both:  

1. **System-Level Metrics** – Performance, resource usage, and operational health of the ClickHouse server.  
2. **Trade-Level Metrics** – Trade execution, sync performance, and query success/failure rates.  

---

## 🚀 **1. System-Level Metrics**  
ClickHouse exposes system-level metrics through the `/metrics` HTTP endpoint, which Prometheus will scrape at **1-second intervals**. These metrics are crucial for ensuring ClickHouse is operating within healthy performance limits and can handle the load.  

### ✅ **1.1. CPU Usage**  
- Monitor the percentage of CPU utilization.  
- High CPU usage over a sustained period can lead to latency spikes and dropped queries.  

| Metric | Description | Threshold | Action |
|--------|-------------|-----------|--------|
| `clickhouse_cpu_usage` | % of CPU used by ClickHouse | > 80% for 10 sec | Notify and reduce load |

---

### ✅ **1.2. Memory Usage**  
- Track the total memory used by ClickHouse vs. configured memory limit.  
- High memory usage can lead to query timeouts or out-of-memory errors.  

| Metric | Description | Threshold | Action |
|--------|-------------|-----------|--------|
| `clickhouse_memory_usage_bytes` | Total memory consumed by ClickHouse (in bytes) | > 80% of memory limit | Notify and restart instance if needed |
| `clickhouse_memory_limit_bytes` | Configured memory limit for ClickHouse | - | - |
| `clickhouse_memory_fragmentation_ratio` | Ratio of allocated to used memory | > 1.05 | Notify and defragment memory |

---

### ✅ **1.3. Disk Usage**  
- Monitor disk consumption and fragmentation.  
- Full disks can cause ClickHouse to fail new write operations.  

| Metric | Description | Threshold | Action |
|--------|-------------|-----------|--------|
| `clickhouse_disk_usage_bytes` | Total disk space used by ClickHouse | > 90% of disk limit | Notify and auto-clean |
| `clickhouse_disk_limit_bytes` | Configured disk size limit | - | - |
| `clickhouse_disk_fragmentation_ratio` | Ratio of allocated to used disk space | > 1.10 | Notify and defragment disk |

---

### ✅ **1.4. Query Performance**  
- Monitor how long ClickHouse takes to execute queries.  
- Long query times could indicate memory pressure, lock contention, or disk I/O problems.  

| Metric | Description | Threshold | Action |
|--------|-------------|-----------|--------|
| `clickhouse_query_latency_ms` | Average query execution time | > 500 ms | Notify and investigate |
| `clickhouse_query_latency_p99` | 99th percentile query time | > 1000 ms | Notify and restart if needed |
| `clickhouse_query_throughput` | Number of queries per second | - | - |

---

### ✅ **1.5. Connections**  
- Track open client connections and connection errors.  

| Metric | Description | Threshold | Action |
|--------|-------------|-----------|--------|
| `clickhouse_active_connections` | Number of active client connections | > 100 | Notify and increase connection limit |
| `clickhouse_connection_errors_total` | Number of failed connection attempts | > 5 in 10 seconds | Notify and throttle new connections |

---

### ✅ **1.6. Uptime**  
- Monitor ClickHouse uptime to catch unexpected crashes or restarts.  

| Metric | Description | Threshold | Action |
|--------|-------------|-----------|--------|
| `clickhouse_uptime_seconds` | Time since ClickHouse started | < 60 sec (unexpected restart) | Notify and investigate |

---

## 📊 **2. Trade-Level Metrics**  
Monitoring trade-level metrics is crucial for tracking trade execution performance, detecting anomalies, and reconciling state mismatches between DuckDB and ClickHouse.  

### ✅ **2.1. Trade Execution Success/Failure**  
- Monitor success and failure rates for trade insertions into ClickHouse.  
- High failure rates may indicate schema mismatches or connectivity issues.  

| Metric | Description | Threshold | Action |
|--------|-------------|-----------|--------|
| `clickhouse_trade_insert_success_total` | Total number of successful trade inserts | - | - |
| `clickhouse_trade_insert_failure_total` | Total number of failed trade inserts | > 5 per second | Notify and investigate |
| `clickhouse_trade_insert_latency_ms` | Time to insert a trade into ClickHouse | > 100 ms | Notify and investigate |

---

### ✅ **2.2. Trade Sync Status**  
- Monitor the sync performance between DuckDB and ClickHouse.  
- Failed syncs should be logged and retried using exponential backoff.  

| Metric | Description | Threshold | Action |
|--------|-------------|-----------|--------|
| `clickhouse_trade_sync_success_total` | Total successful trade syncs | - | - |
| `clickhouse_trade_sync_failure_total` | Total failed trade syncs | > 3 per minute | Notify and retry with backoff |
| `clickhouse_trade_sync_latency_ms` | Time to complete a trade sync | > 200 ms | Notify and investigate |

---

### ✅ **2.3. Query Success/Failure**  
- Monitor query success rates and latency.  
- Failures should trigger retries (if applicable).  

| Metric | Description | Threshold | Action |
|--------|-------------|-----------|--------|
| `clickhouse_query_success_total` | Total number of successful queries | - | - |
| `clickhouse_query_failure_total` | Total number of failed queries | > 5 in 30 seconds | Notify and investigate |
| `clickhouse_query_timeout_total` | Number of queries that timed out | > 2 in 10 seconds | Notify and increase timeout if needed |

---

### ✅ **2.4. Broker and Trade State Sync**  
- Track how long it takes to sync broker and trade state data to ClickHouse.  

| Metric | Description | Threshold | Action |
|--------|-------------|-----------|--------|
| `clickhouse_broker_sync_latency_ms` | Time to sync broker state | > 300 ms | Notify and retry |
| `clickhouse_trade_state_sync_latency_ms` | Time to sync trade state | > 300 ms | Notify and retry |

---

### ✅ **2.5. Trade Reconciliation**  
- Monitor the accuracy of trade reconciliation data between ClickHouse and DuckDB.  

| Metric | Description | Threshold | Action |
|--------|-------------|-----------|--------|
| `clickhouse_trade_reconciliation_mismatch_total` | Number of mismatched trades during reconciliation | > 5 per minute | Notify and retry |

---

## 🚦 **3. Alerting Design**  
The following alerts will be configured in Prometheus to monitor ClickHouse state:  

| Alert Type | Trigger | Action |
|------------|---------|--------|
| **High CPU Usage** | `clickhouse_cpu_usage > 80% for 10 sec` | Notify and reduce load |
| **High Memory Usage** | `clickhouse_memory_usage > 80% for 10 sec` | Notify and restart |
| **High Disk Usage** | `clickhouse_disk_usage > 90%` | Notify and auto-clean |
| **High Latency** | `clickhouse_query_latency_ms > 500 ms` | Notify and investigate |
| **Sync Failures** | `clickhouse_trade_sync_failure_total > 3 per minute` | Notify and retry |
| **Unexpected Restart** | `clickhouse_uptime_seconds < 60 sec` | Notify and investigate |

---

## 🔥 **4. Schema Structure Summary**  
| Category | Number of Metrics | Key Metrics |
|----------|------------------|-------------|
| **System (CPU, Memory, Disk, Connections)** | 12 | CPU, Memory, Disk, Connection, Uptime |
| **Trade Execution** | 6 | Sync latency, trade insert failures, query failures |
| **Reconciliation** | 3 | State mismatch, failed reconciliation |

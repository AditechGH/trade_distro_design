## ✅ **DuckDB Monitoring Design**  
This is the **monitoring design** for DuckDB. Monitoring is essential for ensuring the stability, performance, and reliability of the data layer, especially since DuckDB acts as both a local cache and a data buffer for syncing with ClickHouse.

---

## 🚀 **1. Overview**  
### ✅ **Goals:**  
- Ensure that DuckDB performance is stable under high load.  
- Detect failed sync attempts and retry issues.  
- Monitor query latency, transaction throughput, and data integrity.  
- Set up automatic alerting for sync failures, high memory usage, and unexpected query latencies.  

---

## 📡 **2. Monitoring Sources**  
- **Direct Monitoring:**  
   → DuckDB does not have a native Prometheus exporter. We will set up a Python-based monitoring agent that reads from DuckDB and exposes metrics to Prometheus over HTTP.  

- **Indirect Monitoring:**  
   → Prometheus will scrape the monitoring agent at regular intervals (every 1s).  
   → Monitoring data will be retained for historical analysis in ClickHouse.  

---

## 🖥️ **3. Metrics Design**  

### ✅ **3.1. Query Latency**
Monitor the average execution time for SELECT, INSERT, UPDATE, and DELETE queries.  
- If latency increases → potential bottlenecks in DuckDB engine.  
- Track separately for SELECT and INSERT operations.  

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `duckdb_query_latency_ms` | `Histogram` | query_type | Query execution time in milliseconds |
| `duckdb_select_latency_ms` | `Histogram` | table_name | SELECT query latency |
| `duckdb_insert_latency_ms` | `Histogram` | table_name | INSERT query latency |
| `duckdb_update_latency_ms` | `Histogram` | table_name | UPDATE query latency |
| `duckdb_delete_latency_ms` | `Histogram` | table_name | DELETE query latency |

Example:  
```prometheus
duckdb_query_latency_ms{query_type="SELECT"} 25.5
duckdb_select_latency_ms{table_name="trades"} 18.2
```

---

### ✅ **3.2. Query Throughput**
Monitor how many queries DuckDB is processing per second.  
- High throughput = good performance under load.  
- Low throughput + high latency → possible bottleneck.  

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `duckdb_queries_per_sec` | `Gauge` | query_type | Number of queries processed per second |
| `duckdb_select_queries_per_sec` | `Gauge` | table_name | SELECT queries per second |
| `duckdb_insert_queries_per_sec` | `Gauge` | table_name | INSERT queries per second |
| `duckdb_update_queries_per_sec` | `Gauge` | table_name | UPDATE queries per second |
| `duckdb_delete_queries_per_sec` | `Gauge` | table_name | DELETE queries per second |

Example:  
```prometheus
duckdb_queries_per_sec{query_type="INSERT"} 45
duckdb_select_queries_per_sec{table_name="trades"} 30
```

---

### ✅ **3.3. Memory Usage**
Monitor memory consumption by DuckDB.  
- High memory usage → risk of out-of-memory errors.  
- Ensure that memory fragmentation remains low for efficient query execution.  

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `duckdb_memory_used_bytes` | `Gauge` | instance_id | Total memory used by DuckDB (in bytes) |
| `duckdb_memory_limit_bytes` | `Gauge` | instance_id | Configured memory limit |
| `duckdb_memory_fragmentation_ratio` | `Gauge` | instance_id | Ratio of allocated to used memory |

Example:  
```prometheus
duckdb_memory_used_bytes{instance_id="mt5-1"} 128000000
duckdb_memory_limit_bytes{instance_id="mt5-1"} 256000000
duckdb_memory_fragmentation_ratio{instance_id="mt5-1"} 1.05
```

---

### ✅ **3.4. Disk Usage**
Monitor how much disk space DuckDB is consuming.  
- High disk usage → possible data growth or fragmentation issues.  
- If disk usage approaches the limit → potential data corruption risk.  

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `duckdb_disk_usage_bytes` | `Gauge` | instance_id | Total disk usage by DuckDB |
| `duckdb_max_disk_usage_bytes` | `Gauge` | instance_id | Configured max disk usage |
| `duckdb_disk_fragmentation_ratio` | `Gauge` | instance_id | Disk fragmentation ratio |

Example:  
```prometheus
duckdb_disk_usage_bytes{instance_id="mt5-1"} 100000000
duckdb_max_disk_usage_bytes{instance_id="mt5-1"} 200000000
duckdb_disk_fragmentation_ratio{instance_id="mt5-1"} 1.02
```

---

### ✅ **3.5. Connection Pool**
Monitor active and failed connections to DuckDB.  
- High connection failures → possible network or memory issues.  
- Active connections should remain stable.  

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `duckdb_active_connections` | `Gauge` | instance_id | Active DuckDB connections |
| `duckdb_failed_connections` | `Counter` | instance_id | Number of failed connection attempts |

Example:  
```prometheus
duckdb_active_connections{instance_id="mt5-1"} 12
duckdb_failed_connections{instance_id="mt5-1"} 3
```

---

### ✅ **3.6. Sync Performance**
Monitor the sync between DuckDB and ClickHouse.  
- Failed syncs should be retried automatically.  
- Track retry attempts and backoff intervals.  

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `duckdb_sync_success_count` | `Counter` | instance_id | Successful sync attempts |
| `duckdb_sync_failure_count` | `Counter` | instance_id | Failed sync attempts |
| `duckdb_sync_retry_attempts` | `Counter` | instance_id | Total retry attempts |
| `duckdb_sync_latency_ms` | `Gauge` | instance_id | Average sync time in milliseconds |

Example:  
```prometheus
duckdb_sync_success_count{instance_id="mt5-1"} 50
duckdb_sync_failure_count{instance_id="mt5-1"} 3
duckdb_sync_latency_ms{instance_id="mt5-1"} 120
```

---

### ✅ **3.7. Transaction Handling**
Monitor transactional consistency and performance.  
- Track transaction success and failure rates.  
- High failure rate → possible locking issues.  

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `duckdb_transaction_success_count` | `Counter` | instance_id | Total successful transactions |
| `duckdb_transaction_failure_count` | `Counter` | instance_id | Total failed transactions |
| `duckdb_transaction_latency_ms` | `Gauge` | instance_id | Average transaction execution time |

Example:  
```prometheus
duckdb_transaction_success_count{instance_id="mt5-1"} 25
duckdb_transaction_failure_count{instance_id="mt5-1"} 2
duckdb_transaction_latency_ms{instance_id="mt5-1"} 80
```

---

## 🚨 **4. Alerts**  
| Alert Type | Trigger | Action |
|------------|---------|--------|
| High Query Latency | `duckdb_query_latency_ms > 100ms` | Notify + increase memory |
| High Memory Usage | `duckdb_memory_used_bytes > 90% of limit` | Notify + auto-optimize |
| High Disk Usage | `duckdb_disk_usage_bytes > 90% of limit` | Notify + cleanup |
| Failed Sync Attempt | `duckdb_sync_failure_count > 5 in 10 mins` | Notify + retry |
| Transaction Failure Rate | `duckdb_transaction_failure_count > 10% of attempts` | Notify + rollback |
| Connection Failure | `duckdb_failed_connections > 5 in 5 mins` | Notify + reconnect |

---

## ✅ **5. Prometheus Integration**  
1. Configure Python-based DuckDB monitoring agent to expose metrics via `/metrics`.  
2. Set Prometheus to scrape `/metrics` endpoint every 1s.  
3. Set alert thresholds using `Prometheus AlertManager`.  
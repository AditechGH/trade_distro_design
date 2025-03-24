## ✅ **Revised Redis Schema Definition**  
This is the revised Redis schema definition, combining the updated **RTSS** (Signal Stream), **RIS** (Instance State), and **RTS** (Trade State) sections, with monitoring metrics and event-based alerting included.

---

## 🚀 **1. Trade State Layer (RTS)**
### ✅ **1.1. Latency**
We need to track the time it takes for:  
- Trade state updates (balance, equity, position, leverage)  
- Trade execution state changes  
- Retrieval of trade state data from Redis  

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `latency_us` | `Gauge` | trade_id | Microseconds taken to update state data |
| `retrieval_latency_us` | `Gauge` | trade_id | Microseconds taken to retrieve state data |
| `set_latency_us` | `Gauge` | trade_id | Microseconds taken to set state data in Redis |

Example:  
```prometheus
latency_us{trade_id="abc123"} 105
retrieval_latency_us{trade_id="abc123"} 45
set_latency_us{trade_id="abc123"} 60
```

---

### ✅ **1.2. Throughput**  
Throughput measures how many trade state updates Redis is handling per second.  
| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `commands_processed_per_sec` | `Gauge` | instance_id | Total Redis commands processed per second |
| `trade_state_updates_per_sec` | `Gauge` | trade_id | Number of trade state updates processed per second |

Example:  
```prometheus
commands_processed_per_sec{instance_id="mt5-1"} 350
trade_state_updates_per_sec{trade_id="abc123"} 5
```

---

### ✅ **1.3. Memory Usage**  
Redis operates in-memory, so memory usage should be closely monitored to prevent overflow or eviction issues.  

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `used_memory` | `Gauge` | instance_id | Total memory used by Redis (in bytes) |
| `maxmemory` | `Gauge` | instance_id | Configured maximum memory limit |
| `mem_fragmentation_ratio` | `Gauge` | instance_id | Ratio of allocated memory to physical memory used |

Example:  
```prometheus
used_memory{instance_id="mt5-1"} 2048000
maxmemory{instance_id="mt5-1"} 4000000
mem_fragmentation_ratio{instance_id="mt5-1"} 1.2
```

---

### ✅ **1.4. CPU Load**  
High CPU load can lead to increased latency and reduced throughput.  

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `used_cpu_sys` | `Gauge` | instance_id | Total CPU time spent by the Redis server in system mode |
| `used_cpu_user` | `Gauge` | instance_id | Total CPU time spent by the Redis server in user mode |
| `used_cpu_sys_children` | `Gauge` | instance_id | Total CPU time consumed by child processes (system) |
| `used_cpu_user_children` | `Gauge` | instance_id | Total CPU time consumed by child processes (user) |

Example:  
```prometheus
used_cpu_sys{instance_id="mt5-1"} 12.5
used_cpu_user{instance_id="mt5-1"} 25.2
used_cpu_sys_children{instance_id="mt5-1"} 2.1
used_cpu_user_children{instance_id="mt5-1"} 3.5
```

---

### ✅ **1.5. Keyspace Hit/Miss Ratio**  
This shows how frequently Redis is able to serve data from memory without needing to fetch it from storage (if applicable).  

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `keyspace_hits` | `Counter` | instance_id | Number of keyspace hits |
| `keyspace_misses` | `Counter` | instance_id | Number of keyspace misses |
| `hit_ratio` | `Gauge` | instance_id | Ratio of hits to total key requests |

Example:  
```prometheus
keyspace_hits{instance_id="mt5-1"} 4500
keyspace_misses{instance_id="mt5-1"} 250
hit_ratio{instance_id="mt5-1"} 0.94
```

---

### ✅ **1.6. Expiry Strategy**  
| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `expired_keys` | `Counter` | instance_id | Number of keys that have expired |
| `evicted_keys` | `Counter` | instance_id | Number of keys evicted due to memory pressure |

Example:  
```prometheus
expired_keys{instance_id="mt5-1"} 20
evicted_keys{instance_id="mt5-1"} 5
```

---

## 🌐 **2. Instance State Layer (RIS)**
### ✅ **2.1. State Transitions**
RIS will publish events for each state transition and performance threshold breach:  

| Event | Description |
|-------|-------------|
| `instance_started` | Instance started successfully |
| `instance_failed` | Instance failed |
| `instance_recovered` | Instance recovered after failure |
| `instance_stopped` | Instance stopped |
| `instance_degraded` | Instance health degraded |
| `network_issue` | Network latency issues |
| `socket_failure` | Failure in socket connection |

---

### ✅ **2.2. Alerting**
| Alert Type | Trigger | Action |
|------------|---------|--------|
| High CPU Usage | `cpu_usage > 80%` for more than 10 seconds | Notify and attempt auto-recovery |
| High Memory Usage | `memory_usage > 80%` for more than 10 seconds | Notify and attempt auto-recovery |
| Network Failure | `network_latency > 500ms` for more than 5 seconds | Notify and attempt reconnection |
| Instance Crash | Transition to FAILED | Attempt auto-recovery |
| High Queue Length | `queue_length > 100` | Notify and throttle trade processing |
| High Load Average | `load_average > 2.0` | Attempt to redistribute load |
| Socket Failure | `socket_connections = 0` | Attempt reconnect and notify |
| Recovery Failure | `recovery_attempts > 3` | Stop instance and notify |

---

## 📡 **3. Signal Stream Layer (RTSS)**
### ✅ **Metrics Design**
| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `redis_tss_signals_received_total` | `Counter` | signal_id, action | Total number of trade signals received |
| `redis_tss_signals_processed_total` | `Counter` | signal_id, action | Total number of processed signals |
| `redis_tss_signals_failed_total` | `Counter` | signal_id, action | Total number of failed signals |
| `redis_tss_signal_processing_latency` | `Gauge` | signal_id | Average processing time for signals |
| `redis_tss_stream_size` | `Gauge` | stream_id | Current size of Redis trade signal stream |
| `redis_tss_pending_signals` | `Gauge` | stream_id | Number of pending signals |
| `redis_tss_dropped_signals_total` | `Counter` | signal_id | Number of dropped signals |
| `redis_tss_connection_errors_total` | `Counter` | stream_id | Number of Redis connection errors |

Example:  
```prometheus
redis_tss_signals_received_total{signal_id="sig001", action="buy"} 1
redis_tss_signal_processing_latency{signal_id="sig001"} 12.3
redis_tss_stream_size{stream_id="trade_signals"} 350
```

---

## 🔎 **4. Result Stream Layer (RTRS)**
✅ Retain only last **10,000 results**  
✅ Remove signals after **execution** or after **30 seconds** if not processed.  

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `redis_trade_result` | `Counter` | result_id, status | Trade result count |
| `redis_trade_slippage` | `Gauge` | result_id | Slippage per trade execution |
| `redis_execution_time` | `Gauge` | result_id | Trade execution time |

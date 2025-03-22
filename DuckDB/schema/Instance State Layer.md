# **Instance State Layer – Schema Definition**  
The **Instance State Layer** will store the real-time state and health of each MetaTrader 5 (MT5) instance. Since this layer handles dynamic instance data, it will reside entirely in **Redis** for fast access and low latency.  

---

## ✅ **Purpose of Instance State Layer:**
1. Track the current state of each MT5 instance (e.g., running, stopped).  
2. Monitor instance-level resource usage (e.g., CPU, memory).  
3. Track health status and uptime for monitoring and failover purposes.  
4. Provide real-time instance state to Redis for quick recovery and failover handling.  

---

## 🚀 **Tables in the Instance State Layer**  
| Table Name | Purpose | State Type | Storage |
|------------|---------|------------|---------|
| **instance_state** | Stores the current state of each MT5 instance | Temporary | Redis |
| **instance_metrics** | Stores resource usage (CPU, memory) per instance | Temporary | Redis |
| **instance_health** | Stores health status and uptime per instance | Temporary | Redis |  

---

## 🔎 **1. `instance_state` Table**
### ✅ **Purpose:**
- Track the current state of each instance (e.g., running, stopped, restarting).  
- Store state transitions and last known state.  
- Update in real-time as the instance state changes.  

### ✅ **Schema Design (SQL):**
```sql
CREATE TABLE instance_state (
    instance_id UUID PRIMARY KEY,                -- Unique ID for the instance
    broker_id UUID NOT NULL,                     -- Associated broker ID
    state STRING NOT NULL,                       -- 'running', 'stopped', 'crashed'
    launch_mode STRING NOT NULL,                 -- 'auto', 'manual'
    active_connections INT DEFAULT 0,            -- Number of active connections
    last_updated_at TIMESTAMP NOT NULL           -- Last update time
);
```

---

### ✅ **Redis Adaptation:**
1. **Key:** `instance:{instance_id}`  
2. **Value Type:** Redis Hash  
3. **Fields:** Stored as key-value pairs  

### ✅ Example Redis Command:
```bash
HSET instance:550e8400-e29b-41d4-a716-446655440000 \
    broker_id "776e4400-e29b-41d4-a716-446655440001" \
    state "running" \
    launch_mode "auto" \
    active_connections 4 \
    last_updated_at "2025-03-22T10:15:30Z"
```

---

### ✅ **Example Retrieval:**
```bash
HGETALL instance:550e8400-e29b-41d4-a716-446655440000
```
**Output:**  
```bash
1) "broker_id"
2) "776e4400-e29b-41d4-a716-446655440001"
3) "state"
4) "running"
5) "launch_mode"
6) "auto"
7) "active_connections"
8) "4"
9) "last_updated_at"
10) "2025-03-22T10:15:30Z"
```

---

### ✅ **Example State Update:**
```bash
HSET instance:550e8400-e29b-41d4-a716-446655440000 state "stopped"
```

---

### ✅ **Example State Transitions:**
| Initial State | Transition | Final State |
|---------------|------------|-------------|
| `stopped` → `running` | Instance launched | `running` |
| `running` → `stopped` | User shut down the instance | `stopped` |
| `running` → `crashed` | Unexpected crash | `crashed` |
| `crashed` → `running` | Instance recovered | `running` |

---

### ✅ **Indexes for Fast Lookups:**
Use **Redis Sets** to track instance state for fast lookups:  
1. Create a set for each state:  
```bash
SADD instance:state:running "550e8400-e29b-41d4-a716-446655440000"
SADD instance:state:stopped "550e8400-e29b-41d4-a716-446655440001"
```

2. Get all running instances:  
```bash
SMEMBERS instance:state:running
```

3. Remove from set upon state change:  
```bash
SREM instance:state:running "550e8400-e29b-41d4-a716-446655440000"
SADD instance:state:stopped "550e8400-e29b-41d4-a716-446655440000"
```

---

## 🔎 **2. `instance_metrics` Table**
### ✅ **Purpose:**
- Track real-time resource usage for each instance (CPU, memory).  
- Help monitor instance health and diagnose performance issues.  
- Provide input to the monitoring layer (e.g., Prometheus).  

### ✅ **Schema Design (SQL):**
```sql
CREATE TABLE instance_metrics (
    instance_id UUID PRIMARY KEY,                -- Instance reference
    cpu_usage FLOAT NOT NULL,                    -- CPU usage in percentage
    memory_usage FLOAT NOT NULL,                 -- Memory usage in MB
    network_usage FLOAT NOT NULL,                -- Network usage in KBps
    disk_usage FLOAT NOT NULL,                   -- Disk usage in MB
    last_updated_at TIMESTAMP NOT NULL           -- Last update time
);
```

---

### ✅ **Redis Adaptation:**
1. **Key:** `instance_metrics:{instance_id}`  
2. **Value Type:** Redis Hash  

### ✅ Example Redis Command:
```bash
HSET instance_metrics:550e8400-e29b-41d4-a716-446655440000 \
    cpu_usage 45.5 \
    memory_usage 256.3 \
    network_usage 123.4 \
    disk_usage 500.7 \
    last_updated_at "2025-03-22T10:15:30Z"
```

---

### ✅ **Example Retrieval:**
```bash
HGETALL instance_metrics:550e8400-e29b-41d4-a716-446655440000
```

---

### ✅ **Example State Update:**
```bash
HSET instance_metrics:550e8400-e29b-41d4-a716-446655440000 cpu_usage 38.2
```

---

## 🔎 **3. `instance_health` Table**
### ✅ **Purpose:**
- Monitor instance uptime and health status.  
- Track failed restarts, errors, and recovery attempts.  
- Provide data for the monitoring module.  

### ✅ **Schema Design (SQL):**
```sql
CREATE TABLE instance_health (
    instance_id UUID PRIMARY KEY,                -- Instance reference
    uptime_seconds INT NOT NULL,                 -- Total uptime in seconds
    restart_count INT DEFAULT 0,                 -- Number of restarts
    health_status STRING NOT NULL,               -- 'healthy', 'warning', 'critical'
    last_error STRING,                           -- Last error message
    last_updated_at TIMESTAMP NOT NULL           -- Last update time
);
```

---

### ✅ **Redis Adaptation:**
1. **Key:** `instance_health:{instance_id}`  
2. **Value Type:** Redis Hash  

### ✅ Example Redis Command:
```bash
HSET instance_health:550e8400-e29b-41d4-a716-446655440000 \
    uptime_seconds 3600 \
    restart_count 2 \
    health_status "healthy" \
    last_error "" \
    last_updated_at "2025-03-22T10:15:30Z"
```

---

### ✅ **Example Retrieval:**
```bash
HGETALL instance_health:550e8400-e29b-41d4-a716-446655440000
```

---

### ✅ **Example State Update:**
```bash
HSET instance_health:550e8400-e29b-41d4-a716-446655440000 health_status "warning"
```

---

## 🏆 **Summary of Instance State Layer Schema**
| Table | Purpose | State Type | Storage Location |
|-------|---------|------------|------------------|
| **instance_state** | Instance status and state | Temporary | Redis |
| **instance_metrics** | Resource usage | Temporary | Redis |
| **instance_health** | Health monitoring and uptime | Temporary | Redis |  

---

## ✅ **Design Strengths:**
✅ Real-time state and metrics → Fast lookup and update in Redis.  
✅ Health and metrics available for monitoring → Tied into Prometheus.  
✅ State transitions and sync behavior = Clean and consistent.  
✅ Supports failover and automatic restart handling.  


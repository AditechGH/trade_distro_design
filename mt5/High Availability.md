# ✅ **MT5 High Availability Design**  
*(Architectural Design for High Availability)*  

---

## **1. Overview**  
The High Availability (HA) strategy for the MT5 Trade Engine focuses on maintaining continuous uptime, ensuring failover protection, and minimizing downtime during network or instance failures.  

Since MT5 is the heart of the trade execution system, HA must be implemented across the following key components:  
- **Trade Engine (Rust)** – Ensuring continuous execution of trade signals and state updates.  
- **MT5 Connection Layer** – Ensuring continuous broker connectivity.  
- **Redis Layer** – High availability for trade signals, state updates, and trade results.  
- **DuckDB Layer** – Fast recovery and state retention.  
- **Backend Layer** – Reliable backend connectivity to Redis, MT5, and ClickHouse.  
- **Broker Connectivity** – Continuous connection with failover handling.  

---

## **2. HA Scope**  
| Component | HA Type | Strategy | Failover Strategy | Recovery Time Objective (RTO) | Recovery Point Objective (RPO) |
|-----------|---------|----------|-------------------|------------------------------|------------------------------|
| **Trade Engine (Rust)** | Active-Passive | Cold Standby | Restart Engine | 3 seconds | 0 seconds |
| **Redis** | Replication + Sentinel | Active-Standby | Automatic failover with Sentinel | 1 second | 0 seconds |
| **DuckDB** | Snapshot | Event-based and Batch-based | Restore from snapshot | 5 seconds | 0.5 seconds |
| **Backend (Python)** | Horizontal Scaling | Auto Restart | Restart service | 3 seconds | 0 seconds |
| **Broker Connection** | Multi-connection fallback | Reconnection attempt | Reconnect via retry | 5 seconds | 0 seconds |
| **MT5 Instance State** | Redis Sync + Snapshot | Restore from Redis | Restore state | 1 second | 0 seconds |
| **Trade Results** | Redis + DuckDB + ClickHouse Sync | Redis Retry + ClickHouse Backoff | Restore from Redis → DuckDB → ClickHouse | 1 second | 0 seconds |

---

## **3. HA Strategy**  
### ✅ **3.1. Trade Engine (Rust) HA Strategy**  
- The Trade Engine (Rust) will run as an **active-passive instance**.  
- Only one active instance will process trade signals at any time.  
- If the active instance fails → Backup instance takes over automatically.  

**Failover Strategy:**  
- Use **Rust-based heartbeat** between active and passive instances.  
- If active instance fails → Passive instance will detect failure and take over.  
- Upon restart →  
   - Restore last known state from Redis RTS.  
   - Resume pending trades from Redis RTSS.  

**Trade Engine Health Check:**  
- Heartbeat sent every **1 second** to Redis.  
- If no heartbeat for **3 seconds** → Failover initiated.  

---

### ✅ **3.2. MT5 Connection HA Strategy**  
MT5 connectivity will use a **multi-connection fallback** strategy:  
- Establish primary connection with the broker.  
- If the primary connection fails → Attempt reconnection every **3 seconds**.  
- After **5 failed attempts** →  
   - Switch to secondary connection (if available).  
   - If no secondary connection → Notify user.  

**Connection Recovery:**  
1. Restore open position state from Redis RTS.  
2. Resend pending trades from Redis RTSS.  
3. Restore broker state from Redis RIS.  

---

### ✅ **3.3. Redis HA Strategy**  
Redis will use **Sentinel-based HA** with automatic failover.  
- Redis will run in a **primary-replica setup**.  
- Redis Sentinel will monitor the primary Redis instance:  
   - If Redis primary fails →  
     - Sentinel promotes a replica to primary.  
     - Redis client reconnects automatically to new primary.  

**Redis Sentinel Configuration:**  
```bash
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 3000
sentinel failover-timeout mymaster 5000
sentinel parallel-syncs mymaster 1
```

**Redis Recovery Flow:**  
1. Promote replica to primary (automatic).  
2. Resync Redis data from new primary.  
3. Redis AOF will recover unprocessed trades.  
4. Redis snapshots will recover trade state.  

---

### ✅ **3.4. DuckDB HA Strategy**  
DuckDB will be configured for:  
- High-frequency snapshotting (every 1 minute).  
- Immediate state recovery from the last known snapshot.  
- Trade data stored in DuckDB → Synced to ClickHouse in real-time.  

**DuckDB HA Configuration:**  
- Snapshot stored in `/var/lib/duckdb/backups/`.  
- Restore from last known snapshot in case of failure.  
- Failed syncs will be retried automatically with backoff logic.  

**DuckDB Sync Recovery:**  
- Rust engine writes to DuckDB →  
- If sync to ClickHouse fails →  
   - Store in `failed_syncs` table in DuckDB.  
   - Backend retries every **5 seconds**.  

---

### ✅ **3.5. Backend HA Strategy**  
Backend will run as a horizontally scalable microservice:  
- Auto-restart on failure.  
- Auto-recover connection to Redis and DuckDB.  
- Stateless design → No dependency on persistent state.  
- Use Kubernetes (or systemd) for automated restart.  

**Backend Recovery Flow:**  
1. Restart backend service.  
2. Restore Redis connection.  
3. Restore DuckDB connection.  
4. Resume Redis RTSS processing.  

**Python Service Restart Command:**  
```bash
sudo systemctl restart backend
```

---

### ✅ **3.6. MT5 Trade Results HA Strategy**  
Trade results will follow a three-tier storage model:  
1. **Redis RTRS** – Fast, real-time trade execution.  
2. **DuckDB** – Event-based storage (in-memory + disk).  
3. **ClickHouse** – Long-term trade result storage.  

**Failover Strategy:**  
- Redis stores trade result until processed.  
- If Redis fails → Restore from last Redis snapshot.  
- If Redis snapshot corrupted → Restore from DuckDB.  
- If DuckDB corrupted → Restore from ClickHouse.  

---

### ✅ **3.7. Broker Connectivity HA Strategy**  
- If broker connection drops →  
   - Retry every **3 seconds**.  
   - After **5 failed attempts** →  
     - Attempt recovery.  
     - After **10 failed attempts** → Notify user.  
- If no recovery after **10 attempts** →  
   - Temporarily suspend trade signals.  

**Recovery Priority:**  
1. Restore connection.  
2. Restore last known trade state.  
3. Resume trade signals.  

---

## **4. HA Infrastructure Setup**  
### ✅ **4.1. Redis HA Infrastructure**  
- Primary-Replica setup.  
- Redis Sentinel for failover.  
- Use `volatile-lru` for stream eviction.  

**Redis Cluster Configuration:**  
```bash
maxmemory-policy volatile-lru
tcp-backlog 1024
tcp-keepalive 300
```

---

### ✅ **4.2. MT5 Instance HA Infrastructure**  
- MT5 will use multi-session configuration:  
   - Primary connection + backup connection.  
   - Session monitoring every **1 second**.  

---

### ✅ **4.3. DuckDB HA Infrastructure**  
- Use snapshots for recovery.  
- Store last **10 snapshots** in `/var/lib/duckdb/backups/`.  
- Rust handles concurrent access and data integrity.  

---

### ✅ **4.4. Backend HA Infrastructure**  
- Use Python’s `FastAPI` for backend service.  
- Service auto-restart on failure.  
- Monitor using systemd.  

**systemd Configuration:**  
```bash
[Service]
Restart=always
RestartSec=3
```

---

## **5. HA Testing**  
### ✅ **5.1. Redis Failure Test**  
1. Kill Redis primary → Sentinel should promote replica.  
2. Restore Redis connection.  

### ✅ **5.2. DuckDB Failure Test**  
1. Crash DuckDB process → Restart and restore from snapshot.  

### ✅ **5.3. Backend Failure Test**  
1. Kill backend service → Restart using systemd.  
2. Restore Redis and DuckDB connection.  

### ✅ **5.4. MT5 Failure Test**  
1. Kill MT5 connection → Attempt reconnection every 3 seconds.  
2. Resume trade execution from Redis RTS.  

### ✅ **5.5. Broker Connection Failure Test**  
1. Kill broker connection → Attempt reconnect.  
2. Suspend trade signals after 10 failed attempts.  

---

## ✅ **MT5 High Availability → COMPLETE**  
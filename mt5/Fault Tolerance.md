# ✅ **MT5 Fault Tolerance Design**  
*(Architectural Design for Fault Tolerance)*  

---

## **1. Overview**  
The MT5 fault tolerance design focuses on ensuring system resilience under failure conditions — including component failure, network disconnection, data corruption, and broker-side issues. The goal is to handle failures gracefully, recover state automatically, and minimize system downtime.  

### **Key Fault Tolerance Components:**  
1. **Trade Engine (Rust)** – Ensure stateful recovery and redundancy in trade signal processing.  
2. **MT5 Connection Layer** – Handle broker disconnections and network failures.  
3. **Redis Layer** – Ensure state retention and failover using Redis Sentinel.  
4. **DuckDB Layer** – Ensure recovery from failed inserts and corrupted files.  
5. **Backend Layer** – Handle backend request failures and sync retries.  
6. **Broker Connectivity** – Ensure multi-broker redundancy.  

---

## **2. Fault Scenarios**  
### 🔴 **2.1. Trade Engine Failure**  
**Failure Type:**  
- Trade engine crashes unexpectedly.  
- Trade engine enters a deadlock state.  

**Handling Strategy:**  
- Use systemd to auto-restart the trade engine.  
- Recover last state from Redis RTS and Redis RTSS.  
- If Redis data is lost → Recover state from DuckDB snapshot.  

**Detection:**  
- Trade engine health check via heartbeat every **1 second**.  
- If no heartbeat for **3 seconds** → Trigger failover to backup instance.  

**Recovery Strategy:**  
- Restart using systemd.  
- Restore last known state from Redis RTS.  
- Replay any pending trade signals from Redis RTSS.  

---

### 🔴 **2.2. MT5 Connection Failure**  
**Failure Type:**  
- Connection to broker lost.  
- Broker temporarily unavailable.  
- Network failure on the broker side.  

**Handling Strategy:**  
- Retry connection every **3 seconds** (up to 10 attempts).  
- If broker connection is still down → Switch to backup connection.  
- If all brokers fail → Suspend trade processing and alert user.  

**Detection:**  
- Heartbeat to broker every **3 seconds**.  
- Connection timeout after **5 seconds**.  

**Recovery Strategy:**  
1. Attempt reconnection → Retry every **3 seconds**.  
2. If connection to all brokers fails → Suspend trading.  
3. Restore connection → Replay missed trade signals from Redis RTSS.  

---

### 🔴 **2.3. Redis Failure**  
**Failure Type:**  
- Redis primary node failure.  
- Redis data corruption.  
- Connection failure to Redis instance.  

**Handling Strategy:**  
- Redis Sentinel will promote a replica to primary.  
- If all Redis nodes fail → Restore from latest snapshot.  
- If data corruption occurs → Clear Redis cache and restore from DuckDB.  

**Detection:**  
- Redis Sentinel monitors Redis every **500ms**.  
- Sentinel detects failure → Promotes backup node within **1 second**.  

**Recovery Strategy:**  
1. Promote new primary node using Sentinel.  
2. Resync Redis data from DuckDB and ClickHouse.  
3. Restart Redis client connections.  

---

### 🔴 **2.4. DuckDB Failure**  
**Failure Type:**  
- DuckDB data corruption.  
- Failed write operation.  
- File I/O error.  

**Handling Strategy:**  
- If write operation fails → Store failed data in `failed_syncs` table.  
- Retry writes automatically with exponential backoff.  
- If corruption detected → Restore from snapshot.  

**Detection:**  
- DuckDB health check every **1 second**.  
- Transaction failure logged to `failed_syncs` table.  

**Recovery Strategy:**  
1. Restore last known snapshot.  
2. Resume pending sync from `failed_syncs` table.  
3. Resync data to ClickHouse.  

---

### 🔴 **2.5. Backend Failure**  
**Failure Type:**  
- Python backend crashes.  
- Data processing failure.  
- Backend connectivity loss to Redis or DuckDB.  

**Handling Strategy:**  
- Restart using systemd.  
- Restore Redis and DuckDB connections.  
- Resume processing trade signals from Redis RTSS.  

**Detection:**  
- Backend health check every **3 seconds**.  
- Connection failure triggers automatic restart.  

**Recovery Strategy:**  
1. Restart backend service using systemd.  
2. Restore Redis connection.  
3. Restore DuckDB connection.  
4. Resume processing signals from Redis RTSS.  

---

### 🔴 **2.6. Broker-Side Failure**  
**Failure Type:**  
- Broker rejects order.  
- Broker returns an error.  
- Market closed.  

**Handling Strategy:**  
- If broker returns **TRADE_RETCODE_REQUOTE** → Retry with new price.  
- If broker returns **TRADE_RETCODE_MARKET_CLOSED** → Store order in queue.  
- If broker returns **TRADE_RETCODE_NO_MONEY** → Notify user and suspend trade.  

**Detection:**  
- Capture broker error code using MT5 API.  

**Recovery Strategy:**  
1. If market closed → Retry after market opens.  
2. If price change → Retry with updated price.  
3. If invalid order → Notify user and stop trade.  

---

### 🔴 **2.7. Data Loss or Corruption**  
**Failure Type:**  
- Data corruption in Redis.  
- Data loss in DuckDB or Redis.  
- Network packet loss.  

**Handling Strategy:**  
- Redis AOF + snapshot recovery.  
- DuckDB snapshot recovery.  
- ClickHouse sync validation.  

**Detection:**  
- Redis corruption → Triggered by checksum validation.  
- DuckDB corruption → Triggered by failed I/O operation.  

**Recovery Strategy:**  
1. Clear corrupted Redis data.  
2. Restore from latest Redis snapshot.  
3. Restore from latest DuckDB snapshot.  

---

## **3. Fault Tolerance Strategy**  
| Component | Fault Tolerance Type | Strategy | Failover Strategy | Recovery Time Objective (RTO) | Recovery Point Objective (RPO) |
|-----------|-----------------------|----------|-------------------|------------------------------|------------------------------|
| **Trade Engine (Rust)** | Stateful Failover | Backup Instance | Restart instance | 3 seconds | 0 seconds |
| **MT5 Connection** | Multi-Broker Fallback | Retry + Backup | Reconnect or switch broker | 5 seconds | 0 seconds |
| **Redis** | Sentinel + AOF | Auto Failover | Promote replica | 1 second | 0 seconds |
| **DuckDB** | Snapshot + Backoff | Restore + Retry | Restore from snapshot | 5 seconds | 0.5 seconds |
| **Backend (Python)** | Stateless Restart | Restart on Failure | Auto Restart | 3 seconds | 0 seconds |
| **Broker** | Retry + Fallback | Retry + Reprice | Reconnect or stop order | 5 seconds | 0 seconds |

---

## **4. Recovery Flow**  
### ✅ **Trade Engine Recovery:**  
1. Restart Rust engine.  
2. Restore trade state from Redis RTS.  
3. Resume trade signal processing from Redis RTSS.  

### ✅ **MT5 Connection Recovery:**  
1. Attempt reconnection.  
2. Restore open trades from Redis.  
3. Resume trade signals.  

### ✅ **Redis Recovery:**  
1. Sentinel promotes replica.  
2. Redis clients reconnect to new primary.  
3. Replay signals from Redis RTSS.  

### ✅ **DuckDB Recovery:**  
1. Restore from latest snapshot.  
2. Retry failed syncs from `failed_syncs` table.  
3. Resume ClickHouse sync.  

### ✅ **Backend Recovery:**  
1. Restart backend.  
2. Restore Redis and DuckDB connection.  
3. Resume signal processing.  

### ✅ **Broker Recovery:**  
1. Retry connection.  
2. Restore state from Redis RTS.  
3. Retry trade signals.  

---

## **5. Fault Injection Testing**  
| Component | Test Type | Expected Outcome |
|-----------|-----------|------------------|
| **Trade Engine** | Kill Process | Restart trade engine + state recovery |
| **MT5 Connection** | Disconnect Broker | Reconnect to broker + replay signals |
| **Redis** | Kill Primary | Promote replica + restore data |
| **DuckDB** | Corrupt Data | Restore snapshot + retry failed sync |
| **Backend** | Kill Process | Restart backend + restore connection |
| **Broker** | Invalid Order | Retry or notify user |

---

## ✅ **MT5 Fault Tolerance → COMPLETE**  
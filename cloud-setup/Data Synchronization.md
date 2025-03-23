## **Architectural Design Document: Data Synchronization for Trade Distribution System**  

---

### **1. Overview**  
This document outlines the architectural design for the data synchronization mechanism between DuckDB (local) and ClickHouse (cloud). The design covers the data flow, handling of edge cases, concurrency, security, and failover mechanisms.  

The goal of the data synchronization system is to achieve **low-latency, fault-tolerant, and consistent** data ingestion into ClickHouse while ensuring that trades are processed in order, without duplication, and with minimal impact on system performance.  

---

### **2. System Components**  
| Component | Description | Location |
|-----------|-------------|----------|
| **Rust (Trade Engine)** | Handles trade execution and state updates | Local |
| **DuckDB** | Local storage for trade data and state | Local |
| **Backend (Python)** | Processes trade data and syncs it to ClickHouse | Cloud |
| **ClickHouse** | High-performance, columnar database for historical and analytical data | Cloud |
| **Redis** | Optional buffer for backpressure and fault recovery | Local |
| **Prometheus** | Monitors system performance and sync health | Local |
| **Grafana** | Visualization of system performance | Local |

---

### **3. Data Synchronization Flow**  

#### ✅ **Happy Path:**  
1. **Trade Execution**  
   - A trade is executed in the Rust-based trade engine.  
   - Rust assigns a unique `UUID` to the trade.  

2. **Write to DuckDB**  
   - Trade data is written into DuckDB (local) using Rust.  
   - Write is wrapped in a transaction to ensure atomicity.  

3. **Notify Backend**  
   - After successful write to DuckDB, the UUID and trade metadata are sent to the backend via TCP (primary) or HTTP (fallback).  

4. **Backend Processes Data**  
   - Backend receives the trade UUID and retrieves trade data from DuckDB.  
   - Data is validated and transformed as needed.  

5. **Write to ClickHouse**  
   - Backend writes data into ClickHouse using ClickHouse's native client over TCP.  
   - Write is batched to minimize network round trips.  

6. **Acknowledge Success**  
   - If ClickHouse confirms the write → Backend clears the trade from memory buffer.  

---

#### ✅ **Edge Case 1: Backend Failure (Network or Server Failure)**  
1. Rust sends UUID to backend.  
2. Backend connection fails after exponential backoff attempts.  
3. UUID is stored in `failed_syncs` table in DuckDB.  
4. Backend runs a scheduled job to retry failed syncs periodically.  

---

#### ✅ **Edge Case 2: ClickHouse Insert Failure**  
1. Backend receives UUID from Rust.  
2. Backend processes data but fails to insert into ClickHouse (e.g., ClickHouse is down).  
3. Backend stores the trade in an **in-memory buffer**.  
4. Backend retries in-memory buffer every 500ms until success.  
5. If backoff threshold is exceeded → Move to `failed_syncs` table in DuckDB.  

---

#### ✅ **Edge Case 3: Partial Fills and Race Conditions**  
1. Trade execution includes multiple fills.  
2. If backend receives a fill before the full trade state is resolved →  
   - Backend stores partial fill state in Redis.  
   - Once full fill is complete → Backend aggregates and inserts to ClickHouse.  

---

#### ✅ **Edge Case 4: Backend Crash During Sync**  
1. Backend receives UUID and starts sync.  
2. Backend crashes before ClickHouse confirmation.  
3. On backend restart:  
   - Backend checks memory buffer and `failed_syncs` table.  
   - Backend retries unconfirmed trades automatically.  

---

#### ✅ **Edge Case 5: Network Partition Between Backend and ClickHouse**  
1. Network partition occurs during sync.  
2. Backend detects failure and applies exponential backoff.  
3. Backend routes trades to Redis or DuckDB until ClickHouse reconnects.  
4. Backend sync resumes once network stabilizes.  

---

### **4. Security Design**  

#### **(a) TLS and mTLS Protection**  
- All communication between:  
   ✅ Rust ↔ Backend  
   ✅ Backend ↔ ClickHouse  
   ✅ Backend ↔ Redis  
…should use **TLS**.  
- For backend ↔ ClickHouse, use **mTLS** for two-way validation.  

---

#### **(b) Replay Protection**  
- UUID, timestamp, and nonce included in every request.  
- Backend rejects requests with:  
   - Out-of-order timestamps  
   - Duplicate UUIDs  
   - Expired nonce (older than 5 seconds)  

---

#### **(c) JWT and HMAC Signature Validation**  
- Each request is signed with an HMAC key.  
- Backend validates signature before processing trade data.  

---

#### **(d) ClickHouse Access Control**  
- Create a dedicated ClickHouse user for backend sync:  
```xml
<users>
    <backend_user>
        <password>secure-password</password>
        <networks>
            <ip>10.x.x.x</ip> <!-- Backend IP -->
        </networks>
        <access_management>0</access_management>
        <readonly>0</readonly>
        <allow_ddl>0</allow_ddl>
    </backend_user>
</users>
```  
- Backend user should have **INSERT-ONLY** permissions.  

---

#### **(e) Rate-Limiting and Connection Throttling**  
- Backend should enforce connection limits:  
   ✅ Max 100 connections/second  
   ✅ Max 50 MB/s data ingestion  
   ✅ Auto-throttle on high memory usage (>80%)  

---

### **5. Concurrency and Locking**  
- Rust assigns a unique UUID to every trade.  
- Backend acquires a lock on UUID before processing:  
```python
if trade_uuid not in lock:
    lock.add(trade_uuid)
    try:
        # Process trade
    finally:
        lock.remove(trade_uuid)
```
- Prevents duplicate processing during race conditions.  

---

### **6. Backpressure Handling**  
1. Backend memory buffer = 1000 trades max.  
2. If buffer exceeds limit:  
   ✅ Offload to Redis.  
   ✅ Redis TTL = 60 seconds.  
   ✅ After 60 seconds → Move to `failed_syncs`.  

---

### **7. Data Consistency**  
- Use the same schema in DuckDB and ClickHouse.  
- Ensure time zones are consistent (UTC).  
- Use `INSERT INTO ... ON DUPLICATE KEY UPDATE` pattern to avoid duplicates.  

---

### **8. Performance Optimization**  
✅ Batch inserts into ClickHouse → 100–500 trades per batch.  
✅ Use ClickHouse’s native client for lower overhead.  
✅ Rust uses connection pooling with DuckDB for better concurrency.  
✅ Prometheus monitors latency and throughput.  

---

### **9. Monitoring and Alerting**  
| Metric | Description | Threshold | Action |
|--------|-------------|-----------|--------|
| **Sync Latency** | Time to sync trade to ClickHouse | >100ms | Alert |
| **Failed Syncs** | Trades in `failed_syncs` table | >10 trades | Retry immediately |
| **Memory Usage** | Backend memory buffer usage | >80% | Throttle sync |
| **Trade Rate** | Trades/sec | >200 trades/sec | Throttle incoming trades |
| **Network Latency** | Backend ↔ ClickHouse latency | >500ms | Backoff | 

---

### **10. Failure and Recovery**  
- **Backend Crash** → On restart, backend scans `failed_syncs` table and retries.  
- **ClickHouse Down** → Backend buffers trades in Redis and retries after 5 seconds.  
- **Network Partition** → Exponential backoff with retry threshold.  
- **Duplicate UUID** → Backend ignores duplicate keys in ClickHouse.  

---

### **11. Future Scalability**  
✅ Horizontal scaling of backend by running multiple instances.  
✅ Increase ClickHouse node count for better ingestion speed.  
✅ Add Redis replication to improve resilience.  

---

## 🚀 **Conclusion**
This design ensures:  
✅ High-performance, low-latency sync to ClickHouse.  
✅ Guaranteed data consistency and transactional integrity.  
✅ Secure and resilient sync process.  
✅ Automatic recovery from failure scenarios.  
## ✅ **EA Architectural Design Document: Fault Tolerance Design**  
**Purpose:**  
This document defines the fault tolerance strategy for the Expert Advisor (EA). The goal is to ensure that the EA can continue functioning without data loss or disruption in case of hardware failures, network issues, software bugs, or service interruptions. Since the EA handles both trade execution and tick data streaming, fault tolerance is critical for real-time operations and maintaining consistent trade state.  

---

## **1. Objectives**  
The key objectives for fault tolerance are:  
✅ Prevent order loss or trade execution failure during partial system failures.  
✅ Ensure no tick data loss even if UDP/TCP fails temporarily.  
✅ Minimize downtime and data corruption during crashes or network issues.  
✅ Ensure consistent state across Redis, DuckDB, and ClickHouse.  
✅ Prevent order duplication and avoid orphaned trades.  

---

## **2. Scope of Fault Tolerance**  
### ✅ **2.1. Components Covered**  
The following components are part of the fault tolerance design:  

| **Component** | **Fault Mode** | **Mitigation Strategy** |
|--------------|----------------|-------------------------|
| **Trade Execution** | Order execution failure | Retry with backoff and failover to secondary strategy |
| **Tick Data Streaming** | Packet loss, connection failure | UDP/TCP fallback, auto-reconnect, and buffering |
| **Redis State (RTS, RIS)** | Redis crash or connection loss | Redis Sentinel and data redundancy |
| **DuckDB** | Data corruption or file locking | Write-Ahead Logging (WAL) and backup recovery |
| **ClickHouse Sync** | Network failure or sync failure | Backoff and retry from `failed_syncs` table |
| **EA Process** | Process crash or forced exit | Windows Service auto-restart |
| **Configuration Data** | JSON file corruption or mismatch | Restore from backup |  
| **Session State** | Session loss due to Redis failure | Auto-reconnect using Redis Sentinel |

---

### ✅ **2.2. Components NOT Covered**  
❌ Historical tick data loss beyond 24 hours is acceptable.  
❌ UDP packet loss (since TCP is the fallback).  
❌ Order loss due to catastrophic broker-level failure is out of scope.  

---

## **3. Fault Tolerance Strategy**  
### ✅ **3.1. Trade Execution Fault Tolerance**  
Since trade execution is the most critical part of the EA, the fault tolerance design is focused on minimizing execution failure risk and handling order state consistency.  

**Fault Scenarios:**  
- Order execution failure due to network issue  
- Order rejected by broker  
- Price slippage or market closure  

**Strategy:**  
1. If order fails → Retry immediately with exponential backoff:  
   - 1s → 2s → 4s → 8s → 16s  
2. If order fails after 5 attempts →  
   - Save in `failed_syncs` table in DuckDB  
   - Retry every 30 seconds (up to 5 attempts)  
3. If trade fails due to invalid price or volume → Notify user immediately  
4. If Redis is unavailable → Save trade state in DuckDB → Retry after 2 seconds  
5. If Redis failure persists → Fallback to ClickHouse state → Sync after recovery  

**Tuning:**  
✅ Max retry attempts → **5**  
✅ Backoff interval → **1s → 2s → 4s → 8s → 16s**  
✅ Max ClickHouse sync delay → **30 seconds**  
✅ Redis fallback threshold → **10 attempts**  

---

### ✅ **3.2. Tick Data Fault Tolerance**  
Tick data is streamed continuously using UDP and TCP, so data loss is possible if network connectivity is unstable.  

**Fault Scenarios:**  
- UDP packet loss  
- TCP connection failure  
- Tick data buffer overflow  

**Strategy:**  
1. UDP sends data first → If UDP fails → Fallback to TCP  
2. If TCP fails → Attempt reconnect every **2 seconds** (max **10 attempts**)  
3. If both UDP and TCP fail → Stop streaming → Alert user  
4. Buffer always holds the latest tick data → No overflow risk  
5. If connection recovers → Buffer starts sending from the latest tick  
6. Historical data is sent directly over TCP (not from buffer)  

**Tuning:**  
✅ UDP to TCP fallback → Enabled  
✅ Reconnect interval → **2 seconds**  
✅ Max reconnect attempts → **10**  
✅ Buffer size → **Latest tick only**  

---

### ✅ **3.3. Redis State Fault Tolerance**  
Redis handles both trade state (RTS) and instance state (RIS). Redis failure could lead to data loss or inconsistent state.  

**Fault Scenarios:**  
- Redis master failure  
- Redis state corruption  
- Redis cluster partitioning  

**Strategy:**  
1. Redis Sentinel will monitor Redis health.  
2. If Redis master fails → Sentinel promotes a new master within **3 seconds**.  
3. All EA instances will automatically switch to the new master.  
4. Redis AOF (Append-Only File) enabled for state recovery.  
5. If Redis corruption → Restore from RDB snapshot (daily).  
6. If Redis failover fails → Sync Redis state from DuckDB → Redis.  

**Tuning:**  
✅ Sentinel failover time → **<3 seconds**  
✅ AOF recovery time → **<5 seconds**  
✅ RDB snapshot frequency → **Daily**  

---

### ✅ **3.4. DuckDB Fault Tolerance**  
DuckDB is local, so fault tolerance focuses on preventing corruption and data loss.  

**Fault Scenarios:**  
- Data corruption  
- File lock failure  
- WAL failure  

**Strategy:**  
1. Use WAL for atomic transactions and recovery.  
2. If DuckDB fails → Restart and restore from last WAL state.  
3. If WAL is corrupted → Restore from backup (last successful state).  
4. If backup fails → Recreate from Redis/ClickHouse state.  

**Tuning:**  
✅ WAL enabled → Yes  
✅ WAL size limit → **256 MB**  
✅ Backup frequency → **500ms**  

---

### ✅ **3.5. ClickHouse Sync Fault Tolerance**  
ClickHouse sync issues can lead to order execution state loss or inconsistent historical data.  

**Fault Scenarios:**  
- Network failure  
- Backend crash  
- Write failure  

**Strategy:**  
1. If ClickHouse sync fails → Retry every **5 seconds** (max **3 attempts**).  
2. If sync fails after 3 attempts → Save state in `failed_syncs` table in DuckDB.  
3. Retry from `failed_syncs` table every **30 seconds**.  

**Tuning:**  
✅ Retry interval → **5 seconds**  
✅ Max retry attempts → **3**  
✅ Backoff interval → **30 seconds**  

---

### ✅ **3.6. EA Process Fault Tolerance**  
If the EA process crashes or exits unexpectedly, it must be restored immediately.  

**Fault Scenarios:**  
- Process crash  
- Forced exit  

**Strategy:**  
1. EA runs as a **Windows Service**.  
2. If EA crashes → Restart immediately using `sc.exe`.  
3. If restart fails **3 times** → Stop service and alert user.  

**Tuning:**  
✅ Restart interval → **1 second**  
✅ Max restart attempts → **3**  
✅ Cooldown time → **30 seconds**  

---

### ✅ **3.7. Session State Fault Tolerance**  
Session state is stored in Redis, but Redis failure could disrupt the session.  

**Fault Scenarios:**  
- Session loss  
- Redis connection failure  

**Strategy:**  
1. If Redis session state fails → Attempt reconnect every **2 seconds**.  
2. If session state recovery fails after 5 attempts → Start a new session.  

**Tuning:**  
✅ Redis Sentinel enabled → Yes  
✅ Reconnect interval → **2 seconds**  
✅ Max recovery attempts → **5**  

---

## **4. Testing Strategy**  
| **Test** | **Goal** | **Pass Criteria** |
|---------|----------|------------------|
| UDP to TCP Fallback Test | Ensure TCP handles UDP failure | TCP fallback within 100 ms |
| Redis Failover Test | Ensure Sentinel promotes a new master | New master within 3 sec |
| EA Restart Test | Ensure EA restarts after crash | Restart within 1 sec |
| ClickHouse Sync Test | Ensure retry strategy works | Sync successful after retry |
| Session State Test | Ensure session recovery works | New session started if needed |

---

## **5. Summary of Fault Tolerance Goals**  
| **Component** | **Recovery Goal** |
|--------------|-------------------|
| Trade Execution | <1 sec |
| Tick Stream | <100 ms |
| Redis State | <3 sec |
| DuckDB | <3 sec |
| ClickHouse Sync | <5 sec |
| EA Process | <1 sec |
| Session State | <3 sec |

---

✅ **Next: Implementation Strategy Phase 1: Setup and Configuration?**
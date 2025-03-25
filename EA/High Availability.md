## ✅ **EA Architectural Design Document: High Availability Design**  
**Purpose:**  
This document defines the high availability (HA) strategy for the Expert Advisor (EA). Since the EA handles both trade execution and tick data streaming, it must remain highly available even during system failures, network issues, or hardware faults. The HA design will cover:  
- Ensuring EA process uptime  
- Reducing failover time  
- Auto-recovery from network failures  
- Supporting multiple EA instances with minimal contention  

---

## **1. Objectives**  
The key objectives for high availability are:  
✅ Ensure EA uptime of **99.9%** (less than **8.76 hours** of downtime per year).  
✅ Support auto-recovery from failures without manual intervention.  
✅ Prevent order duplication or missed trades during failovers.  
✅ Maintain consistent tick data streaming without gaps.  
✅ Support running multiple EA instances concurrently without conflicts.  
✅ Ensure low-latency recovery from network and socket failures.  

---

## **2. Scope of High Availability**  
### ✅ **2.1. Components Covered**
The following components are part of the high availability design:  

| **Component** | **HA Requirement** | **Failure Mode** | **Recovery Mode** |
|--------------|--------------------|------------------|------------------|
| **Trade Execution** | Critical | Order failure or partial execution | Retry with exponential backoff |
| **Tick Data Streaming** | Critical | UDP/TCP disconnection or data loss | Auto-reconnect and data refresh |
| **Redis State** | Critical | Redis instance down or network failure | Redis Sentinel auto-failover |
| **DuckDB** | Moderate | Data loss or corruption | Restore from last successful backup |
| **ClickHouse Sync** | High | Sync failure or backend issue | Retry from `failed_syncs` table |
| **Configuration Data** | High | Corrupted JSON or config mismatch | Restore from last valid JSON backup |
| **Session State** | Moderate | Session state loss | Restore from Redis RDB snapshot |

---

### ✅ **2.2. Components NOT Covered**
The following components are **NOT** covered under HA strategy:  
❌ **Tick Data** → Tick stream data is transient and will not be backed up.  
❌ **Historical Data** → Historical data loss (beyond 24-hour window) is acceptable.  
❌ **UDP Data Loss** → Temporary packet loss is acceptable; TCP will compensate.  

---

## **3. High Availability Strategy**  
### ✅ **3.1. EA Process Resilience**
The EA process will be configured to automatically restart on failure.  
**Strategy:**  
1. EA will be launched as a **Windows Service** using `sc.exe`.  
2. If the EA process crashes or exits unexpectedly → Service restarts within **1 second**.  
3. If restart fails **3 times** within **30 seconds** → Stop service and alert the user.  
4. Logging of crash details will be enabled for troubleshooting.  

**Tuning:**  
✅ Restart interval → **1 second**  
✅ Max retries → **3 attempts**  
✅ Cooldown time → **30 seconds**  
✅ Logging → Enabled  

---

### ✅ **3.2. Trade Execution High Availability**
Since trade execution is the most critical part of the EA, the design will prioritize real-time recovery:  
**Strategy:**  
1. If order fails → Retry immediately (up to **5 attempts**).  
2. Use exponential backoff for retries:  
   - 1s → 2s → 4s → 8s → 16s  
3. If order fails after 5 attempts → Save in `failed_syncs` (DuckDB) → Retry after **1 minute**.  
4. If Redis is down → Save trade state in DuckDB → Retry every **2 seconds**.  
5. If Redis fails after 10 attempts → Switch Redis instance using Redis Sentinel.  

**Tuning:**  
✅ Exponential backoff → Yes  
✅ Retry max attempts → **5**  
✅ Redis failover threshold → **10 attempts**  
✅ Redis Sentinel → Enabled  

---

### ✅ **3.3. Tick Data Streaming High Availability**
Tick data is streamed continuously using UDP and TCP:  
**Strategy:**  
1. UDP sends data first → Followed by TCP.  
2. If UDP stream is lost → Fall back to TCP.  
3. If TCP fails → Attempt reconnect every **2 seconds** (max 10 attempts).  
4. If both UDP and TCP fail → Stop streaming → Alert user.  
5. Buffer resets automatically after each reconnect.  

**Tuning:**  
✅ UDP to TCP fallback → Yes  
✅ Reconnect interval → **2 seconds**  
✅ Max reconnect attempts → **10**  
✅ Buffer size → **Latest tick only**  

---

### ✅ **3.4. Redis State HA**
Redis Sentinel will handle Redis failover and auto-recovery.  
**Strategy:**  
1. Deploy Redis Sentinel across multiple Redis nodes.  
2. Redis Sentinel will monitor Redis instances.  
3. If Redis instance goes down → Sentinel will promote a new master within **3 seconds**.  
4. All EA instances will automatically switch to the new master.  
5. Redis state (RTS, RIS) will be recovered from AOF logs.  

**Tuning:**  
✅ Sentinel failure detection → **3 seconds**  
✅ Redis replication mode → Asynchronous  
✅ Redis AOF replay → Enabled  
✅ Max Redis state loss → **100ms**  

---

### ✅ **3.5. DuckDB HA**
DuckDB is local and does not support clustering. Therefore, HA will be based on recovery:  
**Strategy:**  
1. Use WAL (Write-Ahead Logging) to prevent data corruption.  
2. Create backup every **500ms**.  
3. If DuckDB instance fails → Restart and restore from last WAL state.  
4. If DuckDB fails beyond recovery → Recreate from Redis/ClickHouse.  

**Tuning:**  
✅ Backup interval → **500ms**  
✅ WAL size limit → **256 MB**  
✅ Max recovery time → **3 seconds**  

---

### ✅ **3.6. ClickHouse Sync HA**
If ClickHouse sync fails, the backend will retry automatically:  
**Strategy:**  
1. If sync fails → Retry every **5 seconds** (up to 3 attempts).  
2. If retry fails → Save state in `failed_syncs` table in DuckDB.  
3. Retry `failed_syncs` every **1 minute**.  

**Tuning:**  
✅ Retry interval → **5 seconds**  
✅ Max retry attempts → **3**  
✅ Backoff interval → **1 minute**  

---

### ✅ **3.7. Session State HA**
Session state will be stored in Redis (RIS).  
**Strategy:**  
1. If Redis connection fails → Attempt reconnect every **2 seconds**.  
2. If session state recovery fails after 5 attempts → Start a new session.  

**Tuning:**  
✅ Redis Sentinel → Yes  
✅ Max recovery attempts → **5**  
✅ Fallback → Start new session  

---

## **4. Failover Strategy**  
| **Component** | **Failure Mode** | **Failover Action** | **Recovery Time** |
|--------------|------------------|---------------------|------------------|
| EA Process | Process crash | Restart using Windows Service | <1 sec |
| Trade Execution | Order failure | Retry with backoff | <1 sec |
| UDP Stream | Packet loss | Switch to TCP | <100 ms |
| TCP Stream | Connection loss | Reconnect with backoff | <2 sec |
| Redis State | Redis master down | Redis Sentinel promotes new master | <3 sec |
| DuckDB | Corruption or crash | Restart and restore WAL | <3 sec |
| ClickHouse Sync | Sync failure | Retry from `failed_syncs` | <5 sec |
| Session State | Redis connection loss | Redis Sentinel failover | <3 sec |

---

## **5. Testing Strategy**
| **Test** | **Goal** | **Pass Criteria** |
|---------|----------|------------------|
| EA Restart Test | Ensure EA restarts after crash | Restart in <1 sec |
| Trade Execution Test | Ensure retry strategy works | Trade executed within 5 attempts |
| Redis Failover Test | Ensure Sentinel promotes new master | New master promoted in <3 sec |
| DuckDB Recovery Test | Ensure WAL recovery works | Recovery in <3 sec |
| Tick Stream Fallback Test | Ensure TCP handles UDP failure | TCP fallback in <100 ms |
| ClickHouse Sync Recovery Test | Ensure retry works | Sync successful after retry |

---

## **6. Summary of High Availability Goals**  
| **Component** | **Uptime Goal** | **Failover Target** |
|--------------|------------------|---------------------|
| EA Process | 99.9% | <1 sec |
| Trade Execution | 99.9% | <1 sec |
| Tick Stream | 99.9% | <100 ms |
| Redis State | 99.9% | <3 sec |
| DuckDB | 99.9% | <3 sec |
| ClickHouse Sync | 99.9% | <5 sec |
| Session State | 99.9% | <3 sec |

---

✅ **Next: Proceed to Fault Tolerance Design?**
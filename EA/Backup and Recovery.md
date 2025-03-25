## ✅ **EA Architectural Design Document: Backup and Recovery**  
**Purpose:**  
This document defines the backup and recovery strategy for the Expert Advisor (EA). Since the EA is critical for both trade execution and tick data handling, ensuring that it can quickly recover from failures, reboots, and crashes without data loss is essential. The backup strategy will focus on:  
- Protecting trade execution data  
- Ensuring tick data consistency  
- Guaranteeing high availability and fast recovery  

---

## **1. Objectives**  
The key objectives for backup and recovery are:  
✅ Ensure no data loss for trade execution events (including partial trades).  
✅ Maintain tick data consistency after a system crash or restart.  
✅ Ensure quick recovery with minimal manual intervention.  
✅ Preserve user-defined configurations and state across instances.  

---

## **2. Scope of Backup and Recovery**  
### ✅ **2.1. Components to Back Up**
The following components will be included in the backup strategy:  

| **Component** | **Data Type** | **Backup Strategy** | **Retention Policy** |
|--------------|---------------|---------------------|---------------------|
| **Trade Execution State** | Order data, position state, open/closed trades | Redis → ClickHouse | 30 days |
| **Tick Data** | Tick stream data | No backup (real-time only) | No retention |
| **EA Configuration** | User settings, trade parameters | JSON file | No expiry |
| **Failed Trade Data** | Orders that failed to sync to Redis/ClickHouse | DuckDB | Until successful sync |
| **User Preferences** | Symbol settings, notification settings | Redis | 30 days |
| **Historical Data** | Historical order execution and performance data | ClickHouse | Permanent |
| **Broker Credentials** | Login, password, server | PostgreSQL | Permanent |
| **Session State** | Session tokens, active instances | Redis | Until session ends |

---

### ✅ **2.2. Components NOT Backed Up**
The following components will NOT be backed up:  
- **Active tick stream** – Tick data is transient and only relevant for real-time charting.  
- **UDP/TCP Tick Stream** – Tick stream data will be overwritten as new ticks arrive.  
- **Open trade buffer** – The buffer is memory-resident and will be reset on restart.  

---

## **3. Backup Strategy**  
### ✅ **3.1. Redis State Backup**  
Redis will be configured for persistent backups using a combination of:  
- **RDB (Snapshotting):** Periodic full dump of Redis state.  
- **AOF (Append-Only File):** Logs every write operation for replaying on recovery.  

**RDB Strategy:**  
- Snapshot every **60 seconds** or every **1000 changes** (whichever comes first).  
- Save state to disk using `save 60 1000` in `redis.conf`.  

**AOF Strategy:**  
- Use `appendfsync always` → Write changes to disk immediately after each operation.  
- Auto-rewrite AOF file when file size exceeds **64 MB**.  

**Tuning:**  
✅ Use AOF for trade state → Higher consistency  
✅ Use RDB for session state → Faster recovery  
✅ AOF size cap → **64 MB**  

---

### ✅ **3.2. DuckDB Backup**  
DuckDB will store trade execution data, failed syncs, and configuration settings.  
**Backup Strategy:**  
- Write changes to DuckDB every **500ms**.  
- Keep **5 most recent backups** on disk.  
- Store backups in `./backup/duckdb/` folder.  

**Tuning:**  
✅ Use WAL (Write-Ahead Logging) → Prevent data loss  
✅ File size cap → **256 MB**  
✅ Auto-pruning → Keep last **5 backups**  

---

### ✅ **3.3. ClickHouse Backup**  
ClickHouse will store trade execution history and performance data.  
**Backup Strategy:**  
- Enable **ZSTD compression** for efficient storage.  
- Keep full backup every **12 hours**.  
- Incremental backup every **1 hour**.  

**Tuning:**  
✅ Compression Level → **ZSTD 3**  
✅ Keep last **7 days of backups** → Retention policy  
✅ Store backups in `./backup/clickhouse/` folder  

---

### ✅ **3.4. JSON File Backup**  
The EA configuration (including user-defined settings) will be stored in JSON files.  
**Backup Strategy:**  
- Save JSON configuration every time a user modifies settings.  
- Keep last **3 backups**.  
- Store in `./config/` folder.  

**Tuning:**  
✅ Use atomic file writes → Avoid corruption  
✅ File size cap → **10 KB**  
✅ Max backups → **3**  

---

### ✅ **3.5. PostgreSQL Backup**  
Broker credentials and user authentication data will be stored in PostgreSQL.  
**Backup Strategy:**  
- Full dump every **6 hours**.  
- Keep last **14 days** of backups.  
- Use `pg_dump` for backups.  

**Tuning:**  
✅ Use GZIP for compression → Minimize storage size  
✅ Use transaction-safe backups → Prevent partial saves  
✅ Keep last **14 backups** → Rotation policy  

---

## **4. Recovery Strategy**  
### ✅ **4.1. Trade State Recovery**
Trade state recovery will prioritize consistency and accuracy.  
**Strategy:**  
1. Load last successful state from Redis (AOF)  
2. If Redis AOF is corrupted → Load last RDB snapshot  
3. Verify consistency against DuckDB  

**Tuning:**  
✅ AOF → Use for high consistency  
✅ RDB → Use for quick recovery  
✅ Auto-recovery time → **<10 seconds**  

---

### ✅ **4.2. Tick Data Recovery**
Tick data will NOT be recovered (since it's real-time).  
**Strategy:**  
1. Allow buffer to start clean.  
2. Allow TradingView to request historical data if needed.  
3. If UDP/TCP connection fails → Attempt reconnect after **2 seconds**.  

**Tuning:**  
✅ Reconnect interval → **2 seconds**  
✅ Buffer reset → On connection failure  

---

### ✅ **4.3. Configuration Recovery**
Configuration will be restored from JSON files.  
**Strategy:**  
1. Load latest valid JSON file.  
2. Verify integrity of JSON content.  
3. If corrupted → Load last backup.  

**Tuning:**  
✅ JSON validation → Yes  
✅ JSON backup recovery → Yes  
✅ Load time → **<1 second**  

---

### ✅ **4.4. Failed Trade Recovery**
Failed trades will be restored from DuckDB's `failed_syncs` table.  
**Strategy:**  
1. On system restart → Query `failed_syncs`.  
2. Attempt resync → First within **1 second**.  
3. If failure persists → Use exponential backoff.  

**Tuning:**  
✅ Retry interval → **1, 2, 4, 8, 16 seconds**  
✅ Max attempts → **5**  
✅ If max attempts fail → Notify user  

---

### ✅ **4.5. PostgreSQL Recovery**
Broker credentials and authentication data will be restored from PostgreSQL backups.  
**Strategy:**  
1. Use latest full backup.  
2. Use latest incremental backup if full backup is missing.  
3. Restart connection after recovery.  

**Tuning:**  
✅ Use transaction-safe recovery  
✅ Recovery time → **<5 seconds**  

---

### ✅ **4.6. Network Recovery**
If Redis, TCP, or UDP connection fails:  
- Attempt reconnect every **2 seconds** (exponential backoff).  
- If connection remains down for **30 seconds** → Restart EA.  
- If Redis instance is down → Attempt recovery from Redis Sentinel.  

**Tuning:**  
✅ Max reconnect attempts → **10**  
✅ Backoff → Exponential  
✅ Failover handling → Use Redis Sentinel  

---

## **5. Testing Strategy**  
| **Test** | **Goal** | **Pass Criteria** |
|---------|----------|------------------|
| Trade State Recovery Test | Ensure consistent trade state recovery | Trade state restored with no data loss |
| Tick Data Recovery Test | Ensure clean buffer after recovery | Buffer reset, no duplicate ticks |
| Failed Trade Recovery Test | Ensure failed trades are retried | Failed trades processed |
| Configuration Recovery Test | Ensure JSON settings are restored | Settings restored with no corruption |
| Redis RDB/AOF Test | Ensure Redis restores data correctly | Trade state recovered |

---

## **6. Summary of Backup and Recovery Goals**  
| **Component** | **Backup Type** | **Retention** | **Recovery Time** |
|--------------|------------------|---------------|------------------|
| Trade State | AOF + RDB | 30 days | <10 sec |
| Tick Data | None | None | N/A |
| Configuration | JSON | Last 3 files | <1 sec |
| Failed Trades | DuckDB | Until sync | <5 sec |
| Historical Data | ClickHouse | 7 days | <10 sec |
| Broker Credentials | PostgreSQL | 14 days | <5 sec |

---

✅ **Next: Shall we proceed to High Availability Design?**
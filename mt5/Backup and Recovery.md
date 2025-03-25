# ✅ **MT5 Backup and Recovery Design**  
*(Architectural Design for MT5 Backup and Recovery)*  

---

## **1. Overview**  
The MT5 Backup and Recovery strategy is designed to ensure that the Trade Engine and state data can recover from failures without data loss or trading disruptions.  
The backup and recovery strategy will handle the following scenarios:  
- Trade Engine crash or restart  
- Broker-side connection loss  
- Redis state failure  
- Rust trade signal processing failure  
- MT5 instance disconnection  

---

## **2. Backup Scope**  
The following components and data will be covered under the backup and recovery plan:  

| Component | Backup Type | Frequency | Location | Retention |
|-----------|-------------|-----------|----------|-----------|
| **Trade State (Redis RTS)** | Snapshot + AOF | Every 5 minutes | Local Redis instance | Last 5 backups |
| **Trade Signals (Redis RTSS)** | AOF (Append Only File) | Every 500ms | Local Redis instance | Last 5 backups |
| **Trade Results (Redis RTRS)** | AOF | Every 1 second | Local Redis instance | Last 5 backups |
| **Instance State (Redis RIS)** | Snapshot + AOF | Every 5 minutes | Local Redis instance | Last 5 backups |
| **MT5 Configurations** | JSON File | Every 5 minutes | Local file system | Last 5 backups |
| **User Preferences** | JSON File + Redis | Every 5 minutes | Local file system | Last 5 backups |
| **DuckDB Data** | Snapshot | Every 1 minute | Local file system | Last 10 backups |
| **Trade History** | Event-based + Batch-based | Real-time → Event-based | DuckDB | Permanent |
| **Trade Results Sync to ClickHouse** | Event-based | Every 500ms or 50 signals | ClickHouse | Permanent |
| **Broker Credentials** | PostgreSQL | Instant after user update | PostgreSQL | Permanent |

---

## **3. Backup Strategy**  
### ✅ **3.1. Redis Backup Strategy**  
Redis will handle three key state categories:  
1. **RTS (Trade State)**  
2. **RTSS (Trade Signals)**  
3. **RTRS (Trade Results)**  
4. **RIS (Instance State)**  

**Redis Backup Configuration:**  
- Use **Append-Only File (AOF)** for real-time backup.  
- Enable Redis snapshotting using `SAVE` every 5 minutes.  
- Store the last **5 backups** for rollback and recovery.  
- Enable `AOF` fsync every second to ensure write durability.  

**Redis Configuration:**  
```bash
save 300 1
appendonly yes
appendfsync everysec
dir /var/lib/redis/backup
```

---

### ✅ **3.2. MT5 Trade Engine Recovery Strategy**  
- Trade engine will store last known state in Redis RTS.  
- If trade engine crashes → On recovery:  
   - Read last known state from Redis RTS  
   - Resume from the last known state  
   - Resend pending trade signals from Redis RTSS  
   - Drop duplicate signals using UUID checks  

**Recovery State Logic:**  
- If the order was partially filled → Resume execution  
- If the order failed → Remove it from state  
- If the order was completed → Mark state as closed  

---

### ✅ **3.3. MT5 Connection Loss Recovery**  
- If MT5 instance loses connection →  
   - Retry connection every **3 seconds**  
   - After **5 failed attempts** → Notify user  
   - After **10 failed attempts** → Disable instance and trigger recovery  

**Recovery Priority:**  
1. Resume from last known position state  
2. Sync with Redis RTS  
3. Drop pending trades after a threshold of **10 failed attempts**  

---

### ✅ **3.4. Redis RTSS Recovery Strategy**  
- Redis RTSS will retain signals until they are processed or timeout.  
- On MT5 failure → Redis RTSS will retain pending signals.  
- After recovery →  
   - Resume signal processing  
   - Drop expired signals  

**RTSS Timeout:**  
- **30-second timeout** → Expire unprocessed signals  
- RTSS will automatically expire signals that exceed timeout  

---

### ✅ **3.5. Redis RTRS Recovery Strategy**  
- Redis RTRS will retain execution results until confirmed by backend sync.  
- On MT5 failure → Redis RTRS will retain last execution result.  
- After recovery →  
   - Sync Redis RTRS with DuckDB  
   - If execution already logged in DuckDB → Drop duplicate  

---

### ✅ **3.6. DuckDB Backup Strategy**  
- DuckDB will store trade history and trade state locally.  
- DuckDB will be snapshotted every **1 minute** using Python backend.  
- Failed sync attempts to ClickHouse will be logged in `failed_syncs` table.  

**Backup Location:**  
- `/var/lib/duckdb/backups/`  
- Store last **10 backups**  
- Compressed using **LZ4** for minimal space usage  

---

### ✅ **3.7. Trade Signal Sync to ClickHouse**  
- If backend sync to ClickHouse fails →  
   - Store signal in `failed_syncs` in DuckDB  
   - Retry every **5 seconds** until successful  
   - After **5 attempts** → Notify user and suspend sync  

**Sync Trigger:**  
- Immediate → After trade execution  
- Batch → After 50 trades or 500ms interval  

---

### ✅ **3.8. MT5 Configuration Backup Strategy**  
- MT5 config files (JSON) will be stored locally:  
   - `/etc/mt5/config/`  
   - `/var/lib/mt5/backup/`  
- Backup config every **5 minutes**  
- Retain last **5 backups**  
- On failure → Restore from last valid config  

---

### ✅ **3.9. Broker Credentials Backup Strategy**  
- Broker credentials stored securely in PostgreSQL.  
- Backup PostgreSQL daily using pg_dump:  
```bash
pg_dump -U postgres mt5db > mt5db_backup.sql
```
- Encrypt backup using AES-256.  
- Retain last **30 backups**.  

---

## **4. Recovery Strategy**  
### ✅ **4.1. Redis Recovery Strategy**  
- On Redis failure → Restart Redis instance  
- Restore from latest snapshot (`dump.rdb`)  
- Restore from latest append-only file (`appendonly.aof`)  

**Redis Recovery Commands:**  
```bash
redis-server --appendonly yes
```

---

### ✅ **4.2. MT5 Instance Recovery Strategy**  
- On MT5 failure → Restart MT5 instance  
- Restore from last known state in Redis RTS  
- Resume signal processing from Redis RTSS  

**MT5 Recovery Priority:**  
1. Restore open trade state  
2. Resume trade signals  
3. Drop duplicate trades based on UUID  

---

### ✅ **4.3. DuckDB Recovery Strategy**  
- On DuckDB failure → Restart DuckDB process  
- Restore from latest snapshot  
- Resume failed syncs to ClickHouse from `failed_syncs` table  

**DuckDB Recovery Commands:**  
```bash
duckdb mt5.db
```

---

### ✅ **4.4. Backend Recovery Strategy**  
- On backend failure → Restart backend process  
- Resume from Redis state and DuckDB  
- Drop duplicate signals using UUID  

**Backend Recovery Strategy:**  
- If Redis state available → Restore  
- If Redis state missing → Restore from DuckDB  
- If DuckDB state missing → Restore from ClickHouse  

---

### ✅ **4.5. PostgreSQL Recovery Strategy**  
- On PostgreSQL failure → Restart service  
- Restore from latest backup using `pg_restore`  
- Reset connection from backend → Resume normal operation  

**PostgreSQL Recovery Commands:**  
```bash
pg_restore -U postgres -d mt5db mt5db_backup.sql
```

---

## **5. Failure Handling**  
### ✅ **5.1. Broker Connection Loss**  
- Retry every **3 seconds**  
- After **5 failed attempts** → Notify user  
- After **10 failed attempts** → Disable instance  

---

### ✅ **5.2. Redis Memory Overflow**  
- Use `volatile-lru` for RTSS and RTRS streams  
- Use `allkeys-lru` for RTS state cache  

---

### ✅ **5.3. State Corruption**  
- If Redis state corrupted → Restore from snapshot  
- If DuckDB state corrupted → Restore from backup  
- If PostgreSQL state corrupted → Restore from backup  

---

## **6. Backup and Recovery Testing**  
1. Crash MT5 instance → Recover from Redis state.  
2. Crash Redis instance → Restore from AOF file.  
3. Crash backend → Resume from Redis and DuckDB state.  
4. Network outage → Resume once connectivity is restored.  
5. PostgreSQL failure → Restore from latest backup.  

# **DuckDB Configuration Design Document**  

This is the official **DuckDB Configuration Design Document** for the Trade Distribution System. It defines how DuckDB will be configured, accessed, and synced between Rust and Python components. It ensures consistent state handling, separation of concerns, and optimized performance.  

---

## **📂 Directory Structure**  
All configuration files are centralized under the `/config` directory:  

```plaintext
/config
├── duckdb_rust_config.toml
├── duckdb_python_config.toml
├── env.example
└── secrets.toml
```

---

## **🔹 1. Rust-Based DuckDB Configuration File (`duckdb_rust_config.toml`)**  
This file configures the DuckDB instance for the **Rust-based DB Manager**. Rust requires write access to DuckDB for direct state updates.  

```toml
[database]
path = "C:/TradeSystem/Data/trade_data.duckdb"   # Absolute path for DuckDB file
read_only = false                                 # Rust requires write capabilities

[performance]
max_threads = 4                                   # Controls parallelism (adjust for CPU cores)
cache_size_mb = 256                               # Cache size for faster queries
temp_directory = "C:/TradeSystem/Temp"            # Temp file location for large writes

[logging]
log_level = "INFO"                                # Available levels: TRACE, DEBUG, INFO, WARN, ERROR
log_path = "C:/TradeSystem/Logs/duckdb_rust.log"  # Dedicated log file for Rust-specific operations

[sync]
clickhouse_endpoint = "http://localhost:8123"     # CH endpoint for backup sync
backup_interval_sec = 600                         # Sync interval (10 mins)
max_retry_attempts = 5                            # Max retries before marking sync as failed

[error_handling]
failed_sync_table = "failed_syncs"                # Table for recording failed sync attempts
retry_backoff_sec = [5, 15, 30, 60]               # Adaptive retry strategy
```

### ✅ **Design Goals:**
✔️ Direct state updates for immediate consistency.  
✔️ Optimized parallel execution using `max_threads`.  
✔️ Independent logging for troubleshooting.  
✔️ Background sync to ClickHouse with adaptive retry.  

---

## **🔹 2. Python-Based DuckDB Configuration File (`duckdb_python_config.toml`)**  
This file configures the DuckDB instance for the **Python-based DB Manager**. Python will only read data from DuckDB — no direct state modification.  

```toml
[database]
path = "C:/TradeSystem/Data/trade_data.duckdb"    # Same DuckDB file as Rust
read_only = true                                  # Python will only read data

[performance]
cache_size_mb = 128                               # Cache size for faster query performance

[logging]
log_level = "INFO"
log_path = "C:/TradeSystem/Logs/duckdb_python.log"

[sync]
clickhouse_endpoint = "http://localhost:8123"     # CH endpoint for backup sync
batch_size = 50                                    # Number of records per sync batch
sync_interval_sec = 300                            # Sync interval (5 mins)

[error_handling]
failed_sync_table = "failed_syncs"
retry_backoff_sec = [10, 30, 60, 120]             # Adaptive retry strategy for failed syncs
```

### ✅ **Design Goals:**
✔️ Read-only mode prevents accidental state corruption.  
✔️ Smaller cache size optimizes memory usage for querying.  
✔️ Batch-based syncing reduces network load.  
✔️ Background sync ensures consistent state in ClickHouse.  

---

## **🔹 3. Environment Variables (`env.example`)**  
This file manages sensitive data and environment-specific configurations. It ensures that config files remain generic and secure.  

```bash
# Environment Variables for DuckDB Managers
DUCKDB_PATH="C:/TradeSystem/Data/trade_data.duckdb"
CLICKHOUSE_ENDPOINT="http://localhost:8123"

# Log file locations
DUCKDB_RUST_LOG="C:/TradeSystem/Logs/duckdb_rust.log"
DUCKDB_PYTHON_LOG="C:/TradeSystem/Logs/duckdb_python.log"
```

### ✅ **Design Goals:**
✔️ Separate sensitive data from config files.  
✔️ Environment-specific flexibility (local vs production).  
✔️ Supports multiple deployment environments.  

---

## **🔹 4. Secrets Configuration (`secrets.toml`)**  
This file is used to store secure credentials and encryption keys. It should **NOT** be version-controlled or shared publicly.  

```toml
[auth]
clickhouse_user = "admin"
clickhouse_password = "your-secure-password"

[encryption]
master_key = "YOUR-ENCRYPTION-KEY-HERE"           # For future encryption needs
```

### ✅ **Design Goals:**
✔️ Centralized credential management.  
✔️ Secure storage for sensitive keys.  
✔️ Future-proof for encryption and enhanced security.  

---

## **🔹 5. Example Usage in Rust DB Manager (`db_manager.rs`)**  
Example code for loading and using the DuckDB configuration in Rust:  

```rust
use config::{Config, File};
use duckdb::{Connection, Result};

pub struct DuckDBManager {
    pub conn: Connection,
}

impl DuckDBManager {
    pub fn new(config_path: &str) -> Result<Self> {
        let settings = Config::builder()
            .add_source(File::with_name(config_path))
            .build()
            .unwrap();

        let db_path = settings.get::<String>("database.path").unwrap();
        let conn = Connection::open(db_path)?;

        Ok(DuckDBManager { conn })
    }

    pub fn execute_query(&self, query: &str) -> Result<()> {
        self.conn.execute(query, [])
    }

    pub fn sync_to_clickhouse(&self) {
        // Sync logic using config values
    }
}
```

### ✅ **Design Goals:**
✔️ Clean separation between config and execution.  
✔️ Agnostic manager using centralized config.  
✔️ Centralized sync behavior using config values.  

---

## **🔹 6. Example Usage in Python DB Manager (`db_manager.py`)**  
Example code for loading and using the DuckDB configuration in Python:  

```python
import toml
import duckdb

class DuckDBManager:
    def __init__(self, config_path):
        self.config = toml.load(config_path)
        self.db_path = self.config['database']['path']
        self.conn = duckdb.connect(self.db_path, read_only=True)

    def execute_query(self, query):
        return self.conn.execute(query).fetchall()

    def sync_to_clickhouse(self):
        # Sync logic using config values
        pass
```

### ✅ **Design Goals:**
✔️ Clean separation between config and execution.  
✔️ Lightweight, read-only connection.  
✔️ Centralized sync behavior using config values.  

---

## **🔹 7. Key Design Considerations**
| Design Focus | Implementation Strategy |
|-------------|-------------------------|
| **Config-Driven Design** | Centralized configuration ensures flexible and consistent behavior. |
| **Security Best Practices** | Sensitive data stored in `.env` and `secrets.toml` only. |
| **Resilient Sync Strategy** | Adaptive retry and sync strategy for ClickHouse backup. |
| **Parallel Processing** | Max threads and batch sizes configured for peak performance. |
| **Centralized Logging** | Separate logs for Rust and Python for better traceability. |
| **Environment Agnostic** | Works consistently across development, staging, and production. |

---

## ✅ **8. Design Goals Summary**
✅ Config-Driven – Minimal hardcoding.  
✅ Secure – Environment and secrets separation.  
✅ Efficient – Threaded processing and batched sync.  
✅ Reliable – Resilient to network failures and sync issues.  
✅ Clean Execution – Consistent handling of sync between Redis ↔ DuckDB ↔ ClickHouse.  

---

## 🔥 **9. What’s Next**
1. Finalize DuckDB schema design.  
2. Implement sync logic in Rust and Python.  
3. Handle Redis ↔ DuckDB ↔ ClickHouse sync at service level.  
4. Test for concurrency, consistency, and performance.  

---

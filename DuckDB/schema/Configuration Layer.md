Excellent — we’re down to the final schema layer! The **Configuration and Settings Layer** will define how we store and manage system-wide settings, user preferences, and trading parameters. This layer is crucial because it directly impacts how the system behaves under different configurations and user-defined rules.

---

# 🚀 **Configuration and Settings Layer – Overview**  
✅ Purpose:  
- Centralize system settings and configuration.  
- Store user-defined parameters (lot size, SL/TP, risk settings).  
- Manage broker-specific settings (spreads, commissions).  
- Ensure consistency across instances and state.  

✅ Storage:  
- **Redis** – Fast real-time access to active settings.  
- **DuckDB** – Persistent storage for local state.  
- **ClickHouse** – Long-term storage for historical configuration.  

---

## 🏆 **High-Level Structure**  
| Layer | Storage | Purpose |
|-------|---------|---------|
| **Redis** | Real-time access | Fast lookup for active config |
| **DuckDB** | Local storage | Persistent settings |
| **ClickHouse** | Historical storage | Settings over time, versioning |

---

## 📊 **1. Expected Types of Configurations**  
| Category | Purpose | Example |
|----------|---------|---------|
| **User Configuration** | User-specific trading settings | Lot size, SL/TP, leverage |
| **Account Configuration** | Broker-level settings | Spreads, margin requirements |
| **Instance Configuration** | MT5 instance-specific settings | Instance path, health settings |
| **Signal Configuration** | Trade signal handling | Entry strategy, lot size, SL/TP |
| **System Configuration** | Global settings | Logging, backup frequency |

---

## ✅ **2. Proposed Schema for Configuration and Settings Layer**  
We’ll define separate tables for each configuration type:  
- `user_config` – User-defined settings  
- `account_config` – Account-level configuration  
- `instance_config` – MT5 instance settings  
- `signal_config` – Trade signal settings  
- `system_config` – Global settings  
- `session_state` – Active session data (temporary state)  

---

## ✅ **3. Schema Definitions**  
### 🔹 **(1) `user_config`**  
Stores user-specific trading preferences.  

| Column | Type | Description |
|--------|------|-------------|
| `user_id` | UUID | Unique user ID |
| `lot_size_type` | VARCHAR(20) | FIXED, PERCENTAGE |
| `lot_size_value` | FLOAT | Lot size value |
| `risk_per_trade_percent` | FLOAT | Risk as % of balance |
| `stop_loss_type` | VARCHAR(20) | FIXED, ATR, PERCENTAGE |
| `take_profit_type` | VARCHAR(20) | FIXED, ATR, PERCENTAGE |
| `max_open_trades` | INTEGER | Maximum concurrent trades |
| `trade_direction` | VARCHAR(10) | BUY, SELL, BOTH |
| `status` | VARCHAR(20) | ACTIVE, INACTIVE |
| `created_at` | TIMESTAMP | Time of creation |
| `updated_at` | TIMESTAMP | Last updated time |

---

### 🔹 **(2) `account_config`**  
Stores broker-specific settings for each account.  

| Column | Type | Description |
|--------|------|-------------|
| `account_id` | UUID | Unique account ID |
| `broker_id` | UUID | Broker ID |
| `spread_type` | VARCHAR(20) | FIXED, VARIABLE |
| `commission` | FLOAT | Commission per lot |
| `margin_requirement` | FLOAT | Minimum margin required |
| `leverage` | FLOAT | Leverage setting |
| `currency_pair` | VARCHAR(10) | Traded pair |
| `status` | VARCHAR(20) | ACTIVE, INACTIVE |
| `created_at` | TIMESTAMP | Time of creation |
| `updated_at` | TIMESTAMP | Last updated time |

---

### 🔹 **(3) `instance_config`**  
Stores MT5 instance-specific settings.  

| Column | Type | Description |
|--------|------|-------------|
| `instance_id` | UUID | Unique instance ID |
| `path` | VARCHAR(255) | Installation path for MT5 |
| `cpu_limit_percent` | FLOAT | Max CPU usage allowed |
| `memory_limit_mb` | FLOAT | Max memory usage allowed |
| `max_trades` | INTEGER | Max trades allowed |
| `slippage_tolerance` | FLOAT | Allowed slippage (in pips) |
| `connection_timeout_sec` | INTEGER | Timeout value in seconds |
| `status` | VARCHAR(20) | ACTIVE, INACTIVE |
| `created_at` | TIMESTAMP | Time of creation |
| `updated_at` | TIMESTAMP | Last updated time |

---

### 🔹 **(4) `signal_config`**  
Stores trade signal handling preferences.  

| Column | Type | Description |
|--------|------|-------------|
| `config_id` | UUID | Unique config ID |
| `entry_strategy` | VARCHAR(50) | BREAKOUT, REVERSAL |
| `exit_strategy` | VARCHAR(50) | STOP_LOSS, TRAILING_STOP |
| `lot_size_type` | VARCHAR(20) | FIXED, PERCENTAGE |
| `risk_per_trade_percent` | FLOAT | Risk per trade (%) |
| `trade_direction` | VARCHAR(10) | BUY, SELL, BOTH |
| `status` | VARCHAR(20) | ACTIVE, INACTIVE |
| `created_at` | TIMESTAMP | Time of creation |
| `updated_at` | TIMESTAMP | Last updated time |

---

### 🔹 **(5) `system_config`**  
Stores global system-level settings.  

| Column | Type | Description |
|--------|------|-------------|
| `config_id` | UUID | Unique config ID |
| `log_level` | VARCHAR(20) | DEBUG, INFO, WARN, ERROR |
| `backup_interval_sec` | INTEGER | Backup frequency (seconds) |
| `max_log_size_mb` | FLOAT | Max log size before rotation |
| `cache_size_mb` | FLOAT | Redis cache size |
| `sync_interval_sec` | INTEGER | Sync frequency with ClickHouse |
| `created_at` | TIMESTAMP | Time of creation |
| `updated_at` | TIMESTAMP | Last updated time |

---

### 🔹 **(6) `session_state`**  
Stores active session state in Redis for real-time lookup.  

| Column | Type | Description |
|--------|------|-------------|
| `session_id` | UUID | Unique session ID |
| `user_id` | UUID | User ID |
| `ip_address` | VARCHAR(50) | Origin IP address |
| `last_active` | TIMESTAMP | Last activity time |
| `device_fingerprint` | VARCHAR(255) | Device fingerprint hash |
| `status` | VARCHAR(20) | ACTIVE, EXPIRED |
| `created_at` | TIMESTAMP | Time of creation |
| `updated_at` | TIMESTAMP | Last updated time |

---

## ✅ **4. Storage Strategy**  
| Table | Redis | DuckDB | ClickHouse |
|-------|-------|--------|------------|
| **user_config** | ❌ | ✅ | ✅ |
| **account_config** | ❌ | ✅ | ✅ |
| **instance_config** | ❌ | ✅ | ✅ |
| **signal_config** | ❌ | ✅ | ✅ |
| **system_config** | ❌ | ✅ | ✅ |
| **session_state** | ✅ | ❌ | ❌ |

---

## ✅ **5. Retention Strategy**
| Table | Redis Retention | DuckDB Retention | ClickHouse Retention |
|-------|-----------------|------------------|---------------------|
| **user_config** | ❌ | Permanent | Permanent |
| **account_config** | ❌ | Permanent | Permanent |
| **instance_config** | ❌ | Permanent | Permanent |
| **signal_config** | ❌ | Permanent | Permanent |
| **system_config** | ❌ | Permanent | Permanent |
| **session_state** | 7 days | ❌ | ❌ |

---

## ✅ **6. Indexing Strategy**
| Table | Indexed Column(s) |
|-------|-------------------|
| **user_config** | `user_id`, `status` |
| **account_config** | `account_id`, `broker_id` |
| **instance_config** | `instance_id`, `status` |
| **signal_config** | `config_id`, `status` |
| **system_config** | `config_id` |
| **session_state** | `session_id`, `user_id` |

---

## 🏆 **Finalized Configuration and Settings Strategy:**  
✅ Redis = Fast access to session state  
✅ DuckDB = Persistent storage  
✅ ClickHouse = Historical aggregation  
✅ Retention + Indexing = Efficient storage  

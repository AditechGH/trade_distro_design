# 🏆 **MT5 Configuration Design Document**  
*Version: 1.0*  
*Author: Trade Distribution System Team*  
*Date: 2025-03-23*  

---

## 🔎 **Overview**  
This document outlines the MT5 configuration design for the Trade Distribution System. The MT5 configuration is critical because it defines how the MT5 instances will be initialized, monitored, and managed within the system. The MT5 configuration will consist of:  

✅ **Centralized Config Data** → Stored in **JSON files** for rapid MT5 bootstrapping.  
✅ **Instance-Specific Config Data** → Stored in **DuckDB** for structured reference and monitoring.  
✅ **Secure Access Credentials** → Stored in **PostgreSQL** for encryption and security.  

---

## 📂 **Storage Strategy**  

| **Data Type** | **Storage Location** | **Purpose** |
|--------------|----------------------|-------------|
| **Broker Login, Password, Server** | PostgreSQL | Secure storage and access control |
| **Instance-Specific Config (state, leverage, etc.)** | DuckDB | Analytical and long-term storage |
| **Trade Pairs and Limits** | JSON + DuckDB | Fast bootstrapping and analytical reference |
| **Trade State** | Redis | Real-time state management |
| **Execution Config** | JSON + DuckDB | MT5 runtime application + state tracking |
| **Connection Settings** | JSON + DuckDB | Fast loading and historical reference |
| **Risk Controls** | JSON + DuckDB | Fast runtime access + monitoring and logging |

---

## 🚀 **Configuration Objectives**  
✅ **Fast Bootstrapping** – JSON config allows rapid instance initialization.  
✅ **Consistency** – DuckDB ensures instance-specific configurations are consistent.  
✅ **Security** – PostgreSQL handles sensitive credentials securely.  
✅ **Real-Time State Handling** – Redis handles live execution state and events.  
✅ **Scalability** – MT5 config should scale linearly with the number of instances.  
✅ **Auto-Recovery** – MT5 should recover state and configuration in case of failure.  

---

## 📦 **MT5 Configuration Types**  
There are three main configuration types:  
1. **Broker-Level Config** – Defines broker-related settings shared across instances.  
2. **Instance-Level Config** – Defines instance-specific settings for each MT5 instance.  
3. **Execution-Level Config** – Defines settings related to trade execution and handling.  

---

## 🔥 **1. Broker-Level Configuration**  
Stored in:  
✅ JSON → For fast loading and initialization.  
✅ DuckDB → For analytical consistency and reference.  
✅ PostgreSQL → For login credentials and sensitive data.  

### **Example JSON Structure:**  
```json
{
  "broker_id": "ABC123",
  "broker_name": "BrokerX",
  "server": "brokerx.server.com",
  "login_number": "123456",
  "password": "********",
  "leverage": 100,
  "min_lot_size": 0.01,
  "max_lot_size": 100,
  "spread_type": "floating",
  "currency": "USD"
}
```

### **DuckDB Schema:**  
```sql
CREATE TABLE broker (
    broker_id UUID PRIMARY KEY,
    broker_name TEXT NOT NULL,
    server TEXT NOT NULL,
    leverage INT NOT NULL,
    min_lot_size FLOAT NOT NULL,
    max_lot_size FLOAT NOT NULL,
    spread_type TEXT NOT NULL,
    currency TEXT NOT NULL
);
```

### **PostgreSQL Schema:**  
```sql
CREATE TABLE broker_credentials (
    broker_id UUID PRIMARY KEY,
    login_number TEXT NOT NULL,
    password TEXT NOT NULL ENCRYPTED,
    server TEXT NOT NULL
);
```

---

## 🔥 **2. Instance-Level Configuration**  
Stored in:  
✅ JSON → For instance initialization.  
✅ DuckDB → For long-term reference and analysis.  
✅ Redis → For real-time instance state tracking.  

### **Example JSON Structure:**  
```json
{
  "instance_id": "ABC123-001",
  "broker_id": "ABC123",
  "trade_pairs": ["EURUSD", "GBPUSD"],
  "max_trades": 10,
  "slippage_limit": 2.0,
  "trade_routing": "direct",
  "health_check_interval_sec": 10
}
```

### **DuckDB Schema:**  
```sql
CREATE TABLE instance (
    instance_id UUID PRIMARY KEY,
    broker_id UUID NOT NULL,
    trade_pairs TEXT[],
    max_trades INT NOT NULL,
    slippage_limit FLOAT NOT NULL,
    trade_routing TEXT NOT NULL,
    health_check_interval_sec INT NOT NULL,
    FOREIGN KEY (broker_id) REFERENCES broker(broker_id)
);
```

### **Redis Schema (Real-Time State):**  
| **Key** | **Value** | **Expiry** |
|---------|-----------|------------|
| instance_state:<instance_id> | JSON state | No expiry |
| instance_health:<instance_id> | `UP`, `DOWN`, `DEGRADED` | No expiry |
| instance_failure:<instance_id> | `FAILURE_REASON` | No expiry |

---

## 🔥 **3. Execution-Level Configuration**  
Stored in:  
✅ JSON → For fast loading.  
✅ DuckDB → For analytical reference.  
✅ Redis → For live execution state.  

### **Example JSON Structure:**  
```json
{
  "execution_id": "EXEC-001",
  "instance_id": "ABC123-001",
  "stop_loss": 50.0,
  "take_profit": 100.0,
  "lot_size": 0.01,
  "risk_percent": 1.0,
  "trade_timeout_sec": 5
}
```

### **DuckDB Schema:**  
```sql
CREATE TABLE execution_config (
    execution_id UUID PRIMARY KEY,
    instance_id UUID NOT NULL,
    stop_loss FLOAT NOT NULL,
    take_profit FLOAT NOT NULL,
    lot_size FLOAT NOT NULL,
    risk_percent FLOAT NOT NULL,
    trade_timeout_sec INT NOT NULL,
    FOREIGN KEY (instance_id) REFERENCES instance(instance_id)
);
```

### **Redis Schema (Real-Time State):**  
| **Key** | **Value** | **Expiry** |
|---------|-----------|------------|
| execution_state:<execution_id> | JSON state | No expiry |
| execution_status:<execution_id> | `OPEN`, `CLOSED`, `FAILED` | No expiry |
| execution_result:<execution_id> | `PnL, Slippage` | No expiry |

---

## 🔒 **4. Risk Control Configuration**  
Stored in:  
✅ JSON → For fast loading.  
✅ DuckDB → For long-term reference.  

### **Example JSON Structure:**  
```json
{
  "max_loss_per_day": 1000.0,
  "max_loss_per_trade": 100.0,
  "max_trades_per_day": 50,
  "max_open_positions": 20
}
```

### **DuckDB Schema:**  
```sql
CREATE TABLE risk_control (
    risk_id UUID PRIMARY KEY,
    max_loss_per_day FLOAT NOT NULL,
    max_loss_per_trade FLOAT NOT NULL,
    max_trades_per_day INT NOT NULL,
    max_open_positions INT NOT NULL
);
```

---

## 🚦 **5. Health Check Configuration**  
Stored in:  
✅ JSON → For fast loading.  
✅ Redis → For real-time monitoring.  

### **Example JSON Structure:**  
```json
{
  "health_check_interval_sec": 5,
  "max_response_time_ms": 300,
  "socket_timeout_sec": 2
}
```

### **Redis Schema:**  
| **Key** | **Value** | **Expiry** |
|---------|-----------|------------|
| health_check:<instance_id> | `PASS`, `FAIL` | No expiry |
| health_check_latency:<instance_id> | `100ms` | No expiry |
| health_check_error:<instance_id> | `SOCKET_TIMEOUT` | No expiry |

---

## 🚨 **6. Trade Pair Configuration**  
Stored in:  
✅ JSON → For fast loading.  
✅ DuckDB → For reference.  

### **Example JSON Structure:**  
```json
{
  "trade_pair": "EURUSD",
  "spread_type": "floating",
  "lot_size": 0.01
}
```

### **DuckDB Schema:**  
```sql
CREATE TABLE trade_pair (
    pair_id UUID PRIMARY KEY,
    pair_name TEXT NOT NULL,
    spread_type TEXT NOT NULL,
    lot_size FLOAT NOT NULL
);
```

---

## 🔥 **Next Step:**  
✅ Finalize the JSON config structure.  
✅ Finalize the DuckDB schema changes.  
✅ Implement Redis key management strategy.  
✅ Proceed to **MT5 Monitoring Design**.  

# ✅ **MT5 Bootstrap Strategy**  
*(Setup of Minimal Executable MT5 Binaries)*  

---

## **1. Overview**  
The goal of the MT5 bootstrap strategy is to create lightweight, pre-configured MT5 instances that can be dynamically spawned and managed through the Trade Engine (Rust) and backend (Python). Each instance will operate independently but will share a common base configuration and connection logic.  

The idea is to create **minimal MT5 executable binaries** that are:  
✅ Pre-configured for fast startup  
✅ Minimal in size to reduce memory overhead  
✅ Easily deployable across different brokers  
✅ Configurable for different trading strategies  
✅ Secure and isolated from the core system  

---

## **2. Strategy Overview**  
We will create a **base MT5 configuration** and use that to bootstrap individual MT5 instances. The Trade Engine (Rust) will manage the lifecycle of these instances, while the backend (Python) will handle authentication and data sync.

### ✅ **Approach:**  
1. Create a base MT5 installation  
2. Strip down to minimal required files and libraries  
3. Pre-configure common broker settings  
4. Package as a lightweight executable  
5. Store minimal binary configuration in JSON files  
6. Rust Trade Engine will dynamically launch and manage instances  

---

## **3. MT5 Installation Strategy**  
### ✅ **3.1. Base Installation**  
1. Download MT5 installer from MetaQuotes official source:  
   - Link → [https://www.metatrader5.com](https://www.metatrader5.com)  
2. Install MT5 on a clean Windows environment.  
3. Disable unnecessary components:  
   - Remove default charts  
   - Disable auto news updates  
   - Remove default indicators  

### ✅ **3.2. Strip Down to Minimal Binaries**  
1. **Keep Only Critical Binaries:**  
   - `terminal.exe` → Main MT5 terminal  
   - `mql5.dll` → MT5 execution library  
   - `config.ini` → Configuration file  
   - `symbols.raw` → Market symbols  
   - `orders.dat` → Order management  
   - `history.dat` → Historical data  
   - `templates/` → Chart templates  
   - `experts/` → Expert Advisors directory  

2. **Remove Unnecessary Files:**  
   - Remove all default market news files  
   - Remove sample scripts and unused indicators  
   - Remove built-in documentation and example templates  

3. **Final Size:**  
- Goal → < **50 MB**  
- Trade Engine will dynamically download symbol data instead of bundling it  
- Historical data can be downloaded on demand  

---

### ✅ **3.3. Configuration Strategy**  
1. **Pre-configure Broker Settings**  
- Store common broker settings in JSON file:  
```json
{
  "broker": {
    "name": "MyBroker",
    "login": "123456",
    "password": "securepassword",
    "server": "MyBroker-Demo"
  },
  "trade_settings": {
    "max_volume": 10.0,
    "min_volume": 0.01,
    "leverage": 100
  },
  "connection_settings": {
    "ping_interval_ms": 1000,
    "reconnect_attempts": 5,
    "connection_timeout_sec": 3
  }
}
```

2. **Create Config.ini File**  
- Pre-fill the `config.ini` file with basic settings:  
```
[Common]
Language=en
Path=C:\Program Files\MetaTrader 5
[Server]
Server=MyBroker-Demo
Login=123456
Password=securepassword
[Terminal]
AllowLiveTrading=true
AutoTrading=false
```

3. **Store Base Configuration in JSON**  
- Create a JSON-based template for broker configurations:  
```json
{
  "broker_id": "mybroker",
  "instance_id": "instance-1234",
  "mt5_path": "C:\\Program Files\\MetaTrader 5",
  "history_path": "C:\\Program Files\\MetaTrader 5\\history",
  "templates_path": "C:\\Program Files\\MetaTrader 5\\templates"
}
```

4. **User-Specific Data Stored in DB**  
- Credentials stored securely in PostgreSQL (AES-256 encrypted).  
- Instance-specific state stored in Redis Instance State (RIS).  
- Performance metrics stored in Redis and scraped by Prometheus.  

---

## **4. MT5 Instance Management**  
### ✅ **4.1. Rust Trade Engine Responsibility**  
1. **Instance Lifecycle:**  
- Trade Engine will:  
   - Spawn new instances from base configuration  
   - Terminate or restart instances based on error codes  
   - Monitor instance health via heartbeat signals  

2. **Dynamic Configuration:**  
- Broker-specific configuration will be passed at runtime.  
- Trade Engine will update config files dynamically before launching instance.  

### ✅ **4.2. Instance Spawning Flow**  
1. User authenticates via frontend (UI).  
2. Backend verifies authentication with PostgreSQL.  
3. Backend sends request to Trade Engine to spawn instance.  
4. Trade Engine:  
   - Clones base MT5 binary  
   - Injects broker settings into config.ini  
   - Launches MT5 instance via `terminal.exe`  
5. Trade Engine registers the new instance in Redis Instance State (RIS).  
6. MT5 instance establishes broker connection.  
7. Trade Engine sends test ping to verify connection.  
8. Successful connection → State logged in Redis.  

---

### ✅ **4.3. Instance Termination Flow**  
1. Backend requests instance termination.  
2. Trade Engine verifies instance state from Redis.  
3. Trade Engine sends termination signal to MT5 instance.  
4. Waits for clean shutdown.  
5. If clean shutdown fails → Force kill the process.  
6. Cleanup files and memory.  

---

## **5. MT5 Binary Handling**  
### ✅ **5.1. Binary Handling Strategy**  
| File | Purpose | Location |
|-------|---------|----------|
| `terminal.exe` | Main MT5 binary | `C:\Program Files\MetaTrader 5` |
| `mql5.dll` | MT5 core library | `C:\Program Files\MetaTrader 5` |
| `config.ini` | Broker connection info | `instance-specific` |
| `symbols.raw` | Market symbols | `instance-specific` |
| `history.dat` | Historical trade data | `instance-specific` |
| `orders.dat` | Order history | `instance-specific` |

---

### ✅ **5.2. Binary Isolation**  
- Each MT5 instance will run in its own directory  
- Trade Engine will generate a unique directory for each instance:  
```
C:\MT5\instances\instance-1234\
C:\MT5\instances\instance-5678\
```
- Binary files will be copied and isolated for each instance  
- No shared file access between instances  

---

### ✅ **5.3. Cleanup and Maintenance**  
- Failed instances will have their directories deleted after cleanup.  
- Historical data and logs will be retained for 30 days.  
- Trade Engine will monitor and automatically delete stale logs.  

---

## **6. Security Handling**  
| Component | Strategy | Notes |
|-----------|----------|-------|
| **Broker Credentials** | AES-256 encryption | Stored in PostgreSQL |
| **Trade Signals** | HMAC-SHA256 signed | Verified by backend |
| **Instance Communication** | Secure Socket Layer (SSL) | Trade Engine ↔ Backend |
| **Instance Isolation** | Per-instance binary separation | No shared memory |

---

## **7. Edge Cases**  
| Scenario | Handling Strategy |
|----------|-------------------|
| Failed MT5 Connection | Retry with exponential backoff (5 attempts) |
| Corrupt Config File | Regenerate from base template |
| Instance Crash | Force kill → Restart → Update Redis |
| Broker Login Failure | Notify frontend + Reconnect |
| Market Close | Suspend trade execution |

---

## **8. Expected Performance**  
| Metric | Goal | Strategy |
|--------|------|----------|
| **Instance Startup Time** | < 1 second | Minimal binary size + pre-configured |
| **Max Concurrent Instances** | 20–30 | Rust handles threading |
| **Memory Consumption** | < 100 MB per instance | Strip down unnecessary binaries |
| **CPU Usage** | < 20% | Efficient event-based model |

---

## ✅ **Phase Outcome**  
1. Base MT5 executable prepared  
2. Dynamic config template generated  
3. Trade Engine configured for dynamic instance spawning  
4. Redis keyspaces defined for state tracking  
5. Backend ready for sync and control  

---

## ✅ **Next Step:**  
- **MT5 Monitoring and Performance Tuning**  
- **Begin integration testing**  
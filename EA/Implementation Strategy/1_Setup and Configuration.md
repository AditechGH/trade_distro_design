## ✅ **EA Architectural Design Document: Implementation Strategy – Phase 1 (Setup and Configuration)**  
**Purpose:**  
This document defines the step-by-step implementation strategy for setting up the Expert Advisor (EA). The goal is to create a standardized and automated setup process that ensures consistency, lightweight configuration, and minimal memory footprint across all instances. The EA will be configured to handle both **trade execution** and **tick data streaming** with high reliability and fault tolerance.  

---

## **1. Objectives**  
The key objectives for the setup and configuration phase are:  
✅ Create a lightweight MT5 executable with minimal footprint.  
✅ Configure the EA with the correct broker connection and account credentials.  
✅ Configure network settings (UDP/TCP) for efficient tick data streaming.  
✅ Set up the trade execution parameters.  
✅ Set up failover and auto-recovery mechanisms.  
✅ Automate deployment to reduce manual intervention.  

---

## **2. Setup Components**  
The following components need to be set up during the initial configuration:  

| **Component** | **Purpose** | **Setup Method** | **Location** |
|--------------|-------------|------------------|-------------|
| **MT5 Executable** | Core trading platform | Pre-built bootstrapped binary | Local |
| **EA Executable** | Handles trading and tick data | Installed as part of MT5 | Local |
| **Network Configuration** | UDP/TCP for data streaming | Configured via EA input | Local |
| **Broker Connection** | Trading credentials and server info | JSON + PG | Local/Cloud |
| **Trade Parameters** | Lot size, SL, TP, spread, etc. | JSON + Redis + PG | Local |
| **Redis Connection** | For real-time state sync | Configured via EA input | Local |
| **DuckDB Connection** | For trade history and state | Configured via EA input | Local |
| **ClickHouse Sync** | For trade state backup | Configured in backend | Cloud |
| **Session State** | For handling login and state recovery | Configured in Redis | Local |

---

## **3. Setup Flow**  
### ✅ **3.1. Create a Minimal MT5 Instance**  
1. Download MT5 installer from the broker's website.  
2. Strip down unnecessary components:  
   - Remove default chart templates.  
   - Remove unused indicators and scripts.  
   - Remove unnecessary UI components (e.g., toolbars).  
3. Copy the minimal MT5 directory as a template.  
4. Store the template in a **compressed archive** for fast deployment.  

✅ **Goal:** Lightweight executable with minimal disk and memory footprint.  

---

### ✅ **3.2. Deploy EA to MT5**  
1. Copy the compiled EA `.ex5` file to the `Experts` folder:  
   - `/MT5/Experts/TradeEngine.ex5`  
2. Create a pre-configured `symbols.set` file to pre-load symbols.  
3. Create a `tick_history.set` file to configure historical data loading.  
4. Create a `trade_params.set` file to store trading parameters.  
5. Set up automatic loading of the EA on MT5 startup.  

✅ **Goal:** Pre-configured EA ready for execution.  

---

### ✅ **3.3. Set Up Broker Connection**  
1. Create a JSON configuration file for broker login:  
   ```json
   {
       "broker_id": "12345",
       "login_number": "12345678",
       "server": "ICMarkets-Demo",
       "password": "encrypted_password"
   }
   ```  
2. Store encrypted credentials in PostgreSQL:  
   - `broker_id` → Primary key  
   - `login_number` → Encrypted  
   - `password` → AES-256 encrypted  
3. Create a connection object in the EA to read from JSON file:  
   - `login` → Use `login_number`  
   - `password` → Decrypt from PG  
   - `server` → Use `broker_id` to connect  

✅ **Goal:** Secure and consistent broker connection.  

---

### ✅ **3.4. Configure Trade Parameters**  
1. Create a `trade_params.json` file:  
   ```json
   {
       "lot_size": 0.1,
       "max_slippage": 2,
       "take_profit": 50,
       "stop_loss": 25,
       "max_spread": 3
   }
   ```  
2. Store trade parameters in Redis for real-time updates:  
   - `lot_size` → `RTS:lot_size`  
   - `max_slippage` → `RTS:max_slippage`  
   - `take_profit` → `RTS:take_profit`  
3. Set Redis TTL to ensure state remains consistent.  

✅ **Goal:** Flexible trade parameter configuration.  

---

### ✅ **3.5. Set Up Network Configuration**  
1. Configure UDP for low-latency tick data streaming:  
   - UDP port → `8888`  
   - Packet size → `512 bytes`  
2. Configure TCP for reliable failover and historical data:  
   - TCP port → `9000`  
   - Packet size → `1024 bytes`  
3. Set up fallback logic:  
   - UDP → First  
   - TCP → After 100ms  

✅ **Goal:** Low-latency and reliable network communication.  

---

### ✅ **3.6. Set Up State Management**  
1. Store state in Redis:  
   - Account state → `RIS:account_state`  
   - Trade state → `RTS:trade_state`  
2. Define Redis expiration policy:  
   - Open trades → No expiration  
   - Closed trades → Expire after **30 days**  
3. Create automatic recovery mechanism on Redis restart.  

✅ **Goal:** Real-time state consistency and recovery.  

---

### ✅ **3.7. Set Up Historical Data**  
1. Read historical data from MT5 using EA:  
   - `SymbolInfoTick()` for tick data  
   - `CopyTicksRange()` for historical ticks  
2. Send data to TradingView over TCP:  
   - Historical → TCP only  
   - Real-time → UDP first, TCP fallback  
3. Enable JSON formatting for easy parsing on TradingView.  

✅ **Goal:** Seamless real-time and historical data handling.  

---

### ✅ **3.8. Set Up Auto-Restart and Recovery**  
1. Register EA as a **Windows Service**:  
   - Start type → **Automatic**  
   - Restart on failure → **Yes**  
2. Create watchdog script:  
   - Monitors memory usage and connection state  
   - Restarts EA if memory threshold exceeded or connection lost  
3. Write state to Redis on restart.  

✅ **Goal:** Auto-restart and recovery from failure.  

---

### ✅ **3.9. Set Up Trade Execution and Failover**  
1. Define failover conditions:  
   - `TRADE_RETCODE_NO_MONEY` → Stop and notify  
   - `TRADE_RETCODE_REQUOTE` → Retry 3 times  
   - `TRADE_RETCODE_TIMEOUT` → Retry with exponential backoff  
2. Set retry strategy:  
   - First retry → 1 second  
   - Second retry → 2 seconds  
   - Third retry → 4 seconds  
   - Max retry → 5 attempts  

✅ **Goal:** Reliable trade execution with failover.  

---

### ✅ **3.10. Set Up Logging**  
1. Configure logging to `clickhouse-server.log`:  
   - `tick_latency.log` → Tick processing time  
   - `trade_execution.log` → Trade state and outcome  
   - `session.log` → Connection and account state  
2. Use `INFO` level for standard events.  
3. Use `ERROR` level for trade failures.  

✅ **Goal:** Centralized logging for easy monitoring and debugging.  

---

## **4. Deployment Strategy**  
| **Component** | **Deployment Method** | **Location** |
|--------------|-----------------------|-------------|
| **MT5 Executable** | Pre-built archive | Local |
| **EA Executable** | Pre-built `.ex5` | Local |
| **Configuration Files** | JSON templates | Local |
| **Trade State** | Redis | Local |
| **Session State** | Redis | Local |
| **Historical Data** | TCP to TradingView | Local/Network |
| **Trade Execution State** | Redis → ClickHouse | Local + Cloud |
| **Broker Config** | JSON + PG | Local + Cloud |

---

## **5. Success Criteria**  
✅ EA starts successfully upon boot.  
✅ Broker connection established without failure.  
✅ Trade execution successful within 100ms (no timeout).  
✅ Tick data sent to TradingView via UDP and TCP.  
✅ Auto-restart and failover tested successfully.  

---

✅ **Next: Trade Execution Handling?**
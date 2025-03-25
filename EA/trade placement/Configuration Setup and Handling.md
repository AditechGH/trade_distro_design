### ✅ **EA Trade Execution Phase – Configuration Setup and Handling**  

This section will cover how the EA trade execution phase will be configured and how the configurations will be handled at runtime.

---

## **1. Overview**  
The EA (Expert Advisor) will be configured to handle **trade execution** based on the user-defined settings provided at the instance level. The EA will accept configuration data upon initialization and will allow for dynamic updates during runtime without needing to restart the EA.

### ✅ **Trade Execution Goals:**  
✔️ Handle trade placement and management (open, modify, close).  
✔️ Follow the user-defined input settings (lot size, stop loss, take profit, risk %, etc.).  
✔️ Ensure the execution adheres to broker and market rules.  
✔️ Maintain minimal footprint and avoid memory overhead.  
✔️ Ensure fast execution, low slippage, and minimal latency.  
✔️ Support both market and pending orders.  
✔️ Automatically adjust for floating spreads and leverage changes.  

---

## **2. Configuration Types**  
The EA will handle **three levels** of configuration:

| Level | Source | Purpose | Storage |
|-------|--------|---------|---------|
| **Instance-Level Config** | JSON file (stored locally) | Persistent settings for the instance (broker, pair, lot size, TP/SL, etc.) | JSON |
| **Session-Level Config** | Redis (RTS/RIS) | Real-time session state and trade state updates | Redis |
| **Dynamic Config** | User Input via UI | Adjust settings without restarting EA (lot size, TP/SL, etc.) | JSON + Redis |  

---

## **3. Configuration Fields**  
### ✅ **Instance-Level Configuration (JSON)**  
The JSON file will hold static data that persists across sessions. It will be loaded into memory when the EA starts and retained until the EA is terminated.  

Example JSON Structure:  
```json
{
  "instance_id": "a7b3c92d-f3c4-4b6d-98c7-12a4bb3411cd",
  "broker": "OANDA",
  "symbol": "EURUSD",
  "lot_size": 0.1,
  "stop_loss": 30,
  "take_profit": 50,
  "risk_percent": 1.0,
  "max_spread": 2.5,
  "slippage_tolerance": 0.2,
  "magic_number": 12345678
}
```

### ✅ **Session-Level Configuration (Redis)**  
Session-level settings will cover real-time state and trade data. The EA will subscribe to updates via Redis streams (`RTS`) and store temporary session state information in memory.

Example Redis Structure:
- `instance_state:{instance_id}` → Instance state data.
- `trade_state:{trade_id}` → Open trade data.
- `broker_state:{broker_id}` → Broker status information.

| Key | Type | Description | TTL |
|------|------|-------------|-----|
| `instance_state:a7b3c92d` | Hash | Instance state (connected, active, etc.) | No expiry |
| `trade_state:1324` | Hash | Active trade data | Expire after close |
| `broker_state:oanda` | Hash | Broker state (market open/closed) | No expiry |

Example:
```python
HSET instance_state:a7b3c92d status connected
HSET trade_state:1324 profit 23.5
HSET broker_state:oanda market_status open
```

### ✅ **Dynamic Configuration (Real-Time Updates)**  
The EA will allow the user to update the configuration **without restarting the EA**.  
- The EA will listen for configuration changes from Redis (`RTS`) or via direct user input from the frontend.  
- Upon receiving the update → Apply the new config immediately.  
- For critical updates (like broker switching) → The EA will trigger a soft restart.  

---

## **4. Configuration Handling at Startup**  
### ✅ **Step 1: Load Instance-Level Config from JSON**  
- Upon startup, the EA will read the JSON configuration and load it into memory.  
- If the JSON file is invalid or missing → EA will fail to load and log the error.  

### ✅ **Step 2: Load Session State from Redis**  
- EA will subscribe to `instance_state:{instance_id}` and `trade_state:{trade_id}` in Redis.  
- If Redis connection fails → EA will retry with exponential backoff.  

### ✅ **Step 3: Apply Dynamic Config**  
- The EA will check Redis for any existing updates to session state or user inputs.  
- It will overwrite the static config values with dynamic config values (if any).  
- Example:  
    - JSON config = `lot_size = 0.1`  
    - Dynamic config update = `lot_size = 0.2`  
    - EA will use `0.2` for execution.  

---

## **5. Configuration Handling at Runtime**  
### ✅ **Dynamic Config Handling**
| Event | Action | Example |
|-------|--------|---------|
| **New Config Received** | Apply update immediately | Lot size change from `0.1` to `0.2` |
| **Invalid Config** | Ignore and log error | Lot size set to negative value |
| **Broker Change** | Restart EA | Change from OANDA to IC Markets |
| **Symbol Change** | Restart EA | Change from EURUSD to GBPUSD |
| **Trade Settings Update** | Apply live | Change SL/TP/Risk% |  

---

## **6. Synchronization Between Redis and JSON**  
1. Redis values will **override JSON** config if they differ.  
2. Redis updates → Persisted back to JSON on soft shutdown.  
3. JSON config → Used as a fallback if Redis connection is lost.  

Example:
- `Lot size = 0.1` in JSON  
- User updates via UI to `0.2` → Redis updated → Redis value is used.  
- On soft shutdown → Redis value saved to JSON.  

---

## **7. Error Handling**  
| Error | Cause | Action |
|-------|-------|--------|
| **Invalid Config** | Bad JSON format | Log error and skip loading |
| **Missing Config** | JSON file not found | Use default config |
| **Redis Connection Failed** | Network failure | Retry with backoff |
| **Invalid Redis Update** | Value out of range | Ignore and log error |
| **Config Conflict** | Redis value conflicts with JSON | Use Redis value |

---

## **8. Configuration Update Handling**  
| Source | Action | Example |  
|--------|--------|---------|  
| **User Input (UI)** | Update config and send to Redis | Lot size → 0.2 → Updated immediately |  
| **Direct Redis Update** | Update internal config | Risk % → 2.0 → Updated immediately |  
| **Invalid Redis Update** | Ignore and log | Negative lot size → Ignored |  

---

## **9. Performance Tuning for Config Handling**  
1. **Memory Footprint** – Config data will remain small (< 5 KB).  
2. **Latency** – Redis latency for config sync expected to be <1ms.  
3. **Concurrency** – EA will use a single-threaded event loop for config updates.  

---

## **10. Security**  
1. Config files will be stored locally with read-only permissions.  
2. Redis will require authentication for config updates.  
3. Session data will not be exposed to external access.  

---

## **11. Backup and Recovery**  
1. JSON config → Saved on disk.  
2. Redis config → Replicated and backed up.  
3. On failure → EA will reload from JSON or Redis.  

---

## **12. Final Flow**  
✅ Load JSON → ✅ Load Redis → ✅ Apply Config → ✅ Watch for Updates → ✅ Execute Trades  

---

## ✅ **Summary**  
- Configuration is stored in JSON + Redis.  
- Redis config will take priority over JSON.  
- Dynamic config will allow real-time updates without EA restart.  
- On failure → EA will fall back to JSON config.  
- Secure, low-latency, and lightweight config handling.  

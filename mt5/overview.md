## **MT5 Architectural Design – Outline**  
MetaTrader 5 (MT5) will serve as the core trade execution engine for the Trade Distribution System. MT5 is where actual trades will be placed and executed, and it will feed live market data back into the system. Given that MT5 is the heart of the project, the architectural design must be airtight, efficient, and fully aligned with the overall system architecture.  

This outline will follow the same **12-step process** we've applied to previous services, but with an MT5-specific focus.  

---

## **1. Purpose and Scope**  
### **Purpose:**  
- MT5 will act as the central trading hub where trade signals are received and executed.  
- MT5 will provide real-time market data (tick data) and account state updates.  
- MT5 will handle trade execution and risk management at the broker level.  
- MT5 will manage trade synchronization across multiple broker instances.  

### **Scope:**  
| Component | Purpose | Status |  
|-----------|---------|--------|  
| **Trade Execution** | Place and manage trades | Core |  
| **Live Market Data** | Feed real-time market data to Redis | Core |  
| **Account State** | Track balance, equity, and margin | Core |  
| **Trade Copier** | Sync and distribute trades to multiple instances | Core |  
| **Broker State** | Track broker-level state and execution results | Core |  

---

## **2. Configuration Design**  
We will set up a **pre-installed MT5 program** on each instance.  
- Each instance will have configurable parameters for:  
   - Lot size (fixed/percentage-based)  
   - Stop-loss (SL) and take-profit (TP)  
   - Risk settings  
   - Trade session timing  
- All configuration data will be centrally managed and passed to MT5 via Python.  

### **Directory Structure:**  
```plaintext
/config
  ├── mt5_config.json
  ├── credentials.json
/logs
  ├── mt5_engine.log
/data
  ├── tick_data
  └── order_data
```

### **Configuration Example:**  
```json
{
  "account_id": "MT5-12345",
  "lot_size": 0.1,
  "risk_mode": "percentage",
  "max_drawdown": 5.0,
  "slippage": 1.5,
  "broker": "IC Markets",
  "session": {
    "start_time": "08:00",
    "end_time": "16:00"
  }
}
```

---

## 🛠️ **3. Schema Design**  
MT5 will not have its own schema — data will flow into Redis and DuckDB.  
| Component | Storage | Purpose |  
|-----------|---------|---------|  
| **Trade State** | Redis (RTS) | Live trade state  
| **Account State** | Redis (RTS) | Live account state  
| **Execution Results** | Redis → DuckDB → ClickHouse | Trade history  
| **Broker State** | Redis (RTS) | Broker-specific data  

---

## 🎯 **4. Monitoring Design**  
MT5 performance and execution state will be monitored via Prometheus and Grafana.  

### ✅ **Trade Execution Metrics:**  
- Number of trades executed per second  
- Order rejection rate  
- Partial fill rate  
- Order modification/cancel rate  
- Slippage  

### ✅ **Account Metrics:**  
- Account balance  
- Margin level  
- Equity  
- Leverage  

### ✅ **Broker Metrics:**  
- Spread  
- Commission  
- Order execution latency  

---

## 🚨 **5. Edge Case Handling**  
| Edge Case | Impact | Handling Strategy |  
|-----------|--------|-------------------|  
| **Partial Fill** | Loss of execution control | Adjust order size and retry |  
| **Slippage** | Execution at worse price | Adjust order price tolerance |  
| **Order Rejection** | Failure to open/close position | Retry + adjust parameters |  
| **Network Loss** | Lost connection to broker | Trigger auto-recovery |  
| **Platform Crash** | MT5 failure | Restart MT5 instance |  
| **Incorrect State** | Discrepancy in state | Resync state from Redis → MT5 |  
| **Broker Outage** | Loss of trading capability | Pause trade execution |  

---

## 🛡️ **6. Permissions and Security**  
### ✅ **MT5 Authentication:**  
- OAuth-based login for broker-level access  
- Session-based token handling  
- Secure storage of credentials using AES-256 encryption  

### ✅ **Trade Authorization:**  
- Allow only whitelisted instruments  
- Allow only pre-approved lot sizes and risk levels  

### ✅ **Role-Based Access Control:**  
- Admin: Configure instances, access logs  
- User: View trades, configure risk settings  
- Read-Only: View-only access  

---

## 🚀 **7. Performance Tuning**  
| Parameter | Value | Purpose |  
|-----------|-------|---------|  
| **Max Trades Per Second** | 100 TPS | Ensure broker-side limits are respected  
| **Max Simultaneous Orders** | 10 orders/sec | Prevent execution bottlenecks  
| **Max Connection Retries** | 3 | Prevent overload on network failure  
| **Tick Data Retention** | 30 days | Minimize Redis memory usage  

---

## 🔄 **8. Backup and Recovery**  
| Component | Backup Type | Frequency | Location |  
|-----------|-------------|-----------|----------|  
| **Trade Logs** | Rolling backup | Hourly | Local + Cloud |  
| **Account State** | Redis snapshot | 5 minutes | Local |  
| **Execution Results** | DuckDB → ClickHouse | Real-time sync | Cloud |  

---

## 🏆 **9. High Availability Design**  
1. **Multiple MT5 Instances**  
   - Each broker has a dedicated MT5 instance.  
   - Each instance runs in parallel with real-time sync.  

2. **Instance Recovery**  
   - If an instance crashes → Auto-restart after 10 seconds  
   - If 3 restart failures → Shut down and raise alert  

3. **Failover Strategy**  
   - Redis holds trade state → On recovery → Resync from Redis  

---

## 💪 **10. Fault Tolerance Design**  
| Failure Type | Impact | Recovery Strategy |  
|--------------|--------|-------------------|  
| **MT5 Crash** | Loss of trade execution | Auto-restart |  
| **Trade State Mismatch** | Incorrect state | Redis resync |  
| **Order Failure** | Rejection/failure | Auto-retry |  
| **Slippage > Threshold** | Execution loss | Adjust price tolerance |  
| **Broker Failure** | No execution | Auto-recover after 10 seconds |  

---

## 🏗️ **11. Implementation Strategy**  
### ✅ **Phase 1: Setup and Configuration**  
1. Pre-install MT5 on all instances  
2. Setup OAuth-based broker authentication  

### ✅ **Phase 2: Execution Layer**  
1. Integrate MT5 with Redis for real-time execution  
2. Configure risk controls  

### ✅ **Phase 3: Monitoring and State Handling**  
1. Monitor execution state using Prometheus  
2. Capture execution logs  

### ✅ **Phase 4: Sync Layer**  
1. Save execution state into Redis  
2. Sync state into DuckDB  

---

## 🚀 **12. Deployment Strategy**  
| Component | Action | Status |  
|-----------|--------|--------|  
| MT5 | Pre-installed | ✅ Configured |  
| Redis Connection | Read + Write | ✅ Configured |  
| DuckDB Sync | Enabled | ✅ Configured |  
| Broker Access | OAuth | ✅ Configured |  
| Risk Controls | Enabled | ✅ Configured |  
| Alerting | Via Prometheus | ✅ Configured |  

---

## ✅ **Deployment Checklist**  
✔️ MT5 Pre-installed  
✔️ Risk Controls Configured  
✔️ Trade State Mapped to Redis  
✔️ Sync to DuckDB Configured  
✔️ Permissions and Role-Based Access Configured  
✔️ Recovery Strategy in Place  

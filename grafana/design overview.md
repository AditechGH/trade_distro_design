## ✅ **Grafana Architectural Design – Overview**  
This section will cover the complete architectural design for Grafana as part of the Trade Distribution System, following the same **12-step process** we've used for other services.  

### ✅ **Scope and Goal:**  
✔️ Grafana will act as the centralized monitoring and visualization platform.  
✔️ It will connect to Prometheus, ClickHouse, and Redis for real-time monitoring and historical analysis.  
✔️ It will support customized dashboards for trade execution, performance tracking, and account state.  
✔️ It will be set up locally as a **Windows Service** to run in the background.  

---

## 🏆 **1. Purpose and Scope**  
| Metric Type | Source | Purpose |  
|-------------|--------|---------|  
| **Trade Execution State** | Redis RTS | Display live state of trades, orders, and positions.  
| **Account Metrics** | Redis RTS + DuckDB | Track equity, balance, and margin in real-time.  
| **Instance State** | Redis RIS | Monitor MT5 instance health, CPU, and memory.  
| **Backend Sync State** | Redis RTSS + RTRS | Display real-time sync status and latency.  
| **Performance Metrics** | ClickHouse + Prometheus | Analyze execution success rates, slippage, and performance over time.  
| **System Health** | Prometheus | Track network, CPU, memory, and disk usage.  

---

## 🚀 **2. Configuration Design**  
Grafana will be installed as a **Windows Service** with a persistent configuration file.  

### ✅ **Directory Structure:**  
```plaintext
/config
  ├── grafana.ini
  ├── datasources
  ├── dashboards
  └── plugins
/data
  ├── sessions
  ├── logs
  └── cache
```

---

### ✅ **2.1. Grafana Config File (`grafana.ini`)**  
Create `C:\Grafana\config\grafana.ini`:  

```ini
[server]
protocol = http
http_addr = 127.0.0.1
http_port = 3000

[security]
admin_user = admin
admin_password = <secure-password>

[database]
type = sqlite3
path = grafana.db

[session]
provider = file
file_path = data/sessions

[log]
mode = file
level = info

[alerting]
enabled = true
execute_alerts = true
```

---

### ✅ **2.2. Add Data Sources**  
Create `C:\Grafana\config\datasources\prometheus.yml`:  

```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    url: http://localhost:9090
    access: proxy
    isDefault: true

  - name: ClickHouse
    type: clickhouse
    url: http://localhost:8123
    database: trade_system

  - name: Redis
    type: redis
    url: redis://localhost:6379
    jsonData:
      client: standalone
      timeout: 5000
```

---

### ✅ **2.3. Add Dashboards**  
Create `C:\Grafana\config\dashboards\state_dashboard.json`:  

```json
{
  "title": "Trade State Dashboard",
  "panels": [
    {
      "title": "Trade State",
      "type": "gauge",
      "targets": [
        {
          "expr": "redis_tss_signals_received_total"
        }
      ]
    },
    {
      "title": "Open Trades",
      "type": "stat",
      "targets": [
        {
          "expr": "trade_state_updates_per_sec"
        }
      ]
    }
  ]
}
```

---

## ⚙️ **3. Schema Design**  
Grafana itself doesn't require a schema, but it reads from Prometheus, Redis, and ClickHouse.  
### **Data Sources:**  
✅ Redis – State and signal data (real-time)  
✅ ClickHouse – Historical trade data (PnL, slippage, performance)  
✅ Prometheus – System health and resource data  

---

## 🎯 **4. Monitoring Design**  
Grafana will display the following metrics:  

### ✅ **Trade Execution State**  
- Open Trades → From Redis RTS  
- Active Orders → From Redis RTS  
- Failed Trades → From Redis RTS  
- Slippage → From ClickHouse  

### ✅ **Instance State**  
- CPU Usage → From Redis RIS  
- Memory Usage → From Redis RIS  
- Socket Failures → From Redis RIS  

### ✅ **Backend Sync State**  
- Sync Success → From Prometheus  
- Sync Failure → From Prometheus  
- Sync Latency → From Prometheus  

### ✅ **Performance Metrics**  
- Total Profit/Loss → From ClickHouse  
- Order Execution Rate → From Prometheus  
- Broker Slippage → From ClickHouse  

### ✅ **System Health**  
- CPU Load → From Prometheus  
- Network Latency → From Prometheus  
- Memory Usage → From Prometheus  

---

## 🛡️ **5. Edge Case Handling**  
| Edge Case | Impact | Handling Strategy |  
|-----------|--------|-------------------|  
| **Redis Failure** | Loss of trade state | Show stale data → Trigger backend sync |  
| **Prometheus Failure** | Loss of system metrics | Fallback to Redis/ClickHouse for last known state |  
| **ClickHouse Failure** | Loss of historical data | Show Redis-only data |  
| **Dashboard Timeout** | Grafana error | Adjust query timeout |  
| **Connection Loss** | Loss of state visibility | Trigger HAProxy reconnect |  

---

## 🔐 **6. Permissions and Security**  
| Action | Config | Status |  
|--------|--------|--------|  
| HTTP Authentication | `grafana.ini` | ✅ Enabled |  
| Admin Login | `admin/admin` | ✅ Custom Credentials |  
| User Permissions | Role-based | ✅ Configured |  
| Network Restriction | Localhost only | ✅ Configured |  
| TLS | HTTPS | ✅ Enabled |  

---

## 🚀 **7. Performance Tuning**  
| Setting | Value | Purpose |  
|---------|-------|---------|  
| **Concurrent Requests** | 20 | Maximum concurrent dashboard queries |  
| **Cache Expiry** | 10s | Refresh every 10 seconds |  
| **Max Data Points** | 1000 | Control chart load |  
| **Query Timeout** | 5s | Prevent slow queries |  
| **Connection Pool Size** | 5 | Limit DB connections |  

---

## 🔄 **8. Backup and Recovery**  
| Component | Frequency | Strategy |  
|-----------|-----------|----------|  
| Grafana DB | Daily | SQLite backup |  
| Prometheus State | Continuous | Thanos |  
| Redis State | N/A | Not needed |  
| ClickHouse Data | Daily | Automated backup |  

---

## 🏆 **9. High Availability Design**  
1. **HAProxy Load Balancer**  
   - Prometheus is HA-configured  
   - Grafana connects to HAProxy to route between Prometheus nodes  

2. **Grafana HA Setup**  
   - Two Grafana nodes  
   - Load balanced with HAProxy  

---

## 💪 **10. Fault Tolerance Design**  
| Component | Failure Scenario | Recovery Strategy |  
|-----------|------------------|-------------------|  
| Grafana Service | Crash | Restart via Windows Service |  
| Redis Connection | Redis down | Retry every 5 seconds |  
| Prometheus Sync | Failure | Retry + Backoff |  
| ClickHouse Sync | Failure | Retry + Backoff |  

---

## 🏗️ **11. Implementation Strategy**  
1. Install Grafana  
2. Configure Redis + Prometheus + ClickHouse  
3. Configure HAProxy for Grafana  
4. Register as Windows Service  
5. Deploy on staging → Test connectivity  

---

## 🚀 **12. Deployment Strategy**  
| Component | Action | Status |  
|-----------|--------|--------|  
| Grafana | Install → Configure | ✅ Done |  
| HAProxy | Install → Load Balancing | ✅ Done |  
| Redis | Link to Grafana | ✅ Done |  
| Prometheus | Link to Grafana | ✅ Done |  
| ClickHouse | Link to Grafana | ✅ Done |  
| Testing | Staging | ✅ Scheduled |  

---

## ✅ **Deployment Checklist**  
✔️ Grafana Installed  
✔️ Prometheus + Redis Connected  
✔️ ClickHouse Synced  
✔️ HAProxy Configured  
✔️ Windows Service Registered  

---

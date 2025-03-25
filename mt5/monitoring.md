# 🏆 **MT5 Monitoring Design**  
*Version: 1.0*  
*Author: Trade Distribution System Team*  
*Date: 2025-03-23*  

---

## 🔎 **Overview**  
This document outlines the **MT5 Monitoring Design** for the Trade Distribution System. Since MT5 is the **core execution engine** in the system, monitoring it for performance, health, state consistency, and order execution success is critical. 

### ✅ **What We Want to Achieve:**  
- ✅ Ensure MT5 instances are online and responsive.  
- ✅ Detect execution errors and trade failures quickly.  
- ✅ Monitor order flow and trade execution efficiency.  
- ✅ Automatically detect high resource utilization (CPU, memory, network).  
- ✅ Auto-recover from failures when possible.  
- ✅ Send alerts when recovery fails or state consistency is lost.  

---

## 🚀 **Monitoring Strategy**  
1. **Real-Time State Monitoring** → Redis (RIS) will store real-time state data.  
2. **Performance Metrics** → Prometheus will scrape Redis and system metrics.  
3. **Health Check Monitoring** → MT5 instances will send heartbeat signals to Redis.  
4. **Execution Monitoring** → Successful and failed trades will be logged and monitored.  
5. **Auto-Recovery** → MT5 will attempt self-recovery upon failure.  
6. **Threshold-Based Alerts** → Prometheus will alert based on performance thresholds.  

---

## 📂 **Storage Strategy**  
| **Metric Type** | **Storage Location** | **Purpose** |
|-----------------|----------------------|-------------|
| **Instance State** | Redis (RIS) | Real-time instance state |
| **Execution State** | Redis (RTS) | Real-time execution state |
| **Performance Metrics** | Prometheus | For alerting and analysis |
| **Historical Performance** | DuckDB | For long-term analysis |
| **Recovery State** | Redis + DuckDB | Auto-recovery upon failure |

---

## 🔥 **1. Monitoring Scope**  
We will monitor the following MT5 components:  
✅ **MT5 Instance State** – Health, uptime, connection state, CPU/Memory usage.  
✅ **Trade Execution** – Order fill rate, slippage, execution latency.  
✅ **Order Book** – Depth of market (DOM) and order fill delays.  
✅ **Risk State** – Margin level, equity, drawdown, leverage.  
✅ **Connectivity** – MT5 connection, broker connection, and network issues.  
✅ **Performance** – CPU, memory, load, and response time.  

---

## 📡 **2. Prometheus Monitoring Endpoints**  
We will create Prometheus scraping endpoints to collect the following:  

### ✅ **Instance Metrics**  
| **Metric** | **Type** | **Description** |
|------------|----------|-----------------|
| `mt5_instance_cpu_usage` | Gauge | CPU usage per MT5 instance |
| `mt5_instance_memory_usage` | Gauge | Memory usage per MT5 instance |
| `mt5_instance_network_latency` | Gauge | Network latency per instance |
| `mt5_instance_uptime` | Counter | Total uptime in seconds |
| `mt5_instance_restart_count` | Counter | Number of auto-restarts |
| `mt5_instance_health_state` | Gauge | 1 = UP, 0 = DOWN |
| `mt5_instance_load_average` | Gauge | System load per instance |
| `mt5_instance_health_check_latency` | Histogram | Health check response time |
| `mt5_instance_connection_errors_total` | Counter | Total MT5 connection failures |

---

### ✅ **Execution Metrics**  
| **Metric** | **Type** | **Description** |
|------------|----------|-----------------|
| `mt5_trade_execution_latency` | Histogram | Trade execution time (ms) |
| `mt5_trade_fill_rate` | Gauge | Percentage of successful fills |
| `mt5_trade_slippage` | Histogram | Execution slippage in pips |
| `mt5_trade_failure_count` | Counter | Number of failed trades |
| `mt5_trade_rejected_count` | Counter | Number of rejected trades |
| `mt5_trade_open_count` | Gauge | Number of open trades |
| `mt5_trade_closed_count` | Gauge | Number of closed trades |
| `mt5_trade_partially_filled_count` | Counter | Number of partially filled trades |
| `mt5_trade_auto_recovery_count` | Counter | Number of auto-recovered trades |
| `mt5_trade_cancellation_rate` | Gauge | Percentage of trades cancelled |

---

### ✅ **Order Book Metrics**  
| **Metric** | **Type** | **Description** |
|------------|----------|-----------------|
| `mt5_order_book_depth` | Gauge | Number of orders in the book |
| `mt5_order_book_spread` | Gauge | Bid-ask spread in pips |
| `mt5_order_book_latency` | Histogram | Time to refresh order book |
| `mt5_order_book_rejection_rate` | Gauge | Percentage of rejected orders |
| `mt5_order_book_cancellation_rate` | Gauge | Percentage of cancelled orders |

---

### ✅ **Risk Metrics**  
| **Metric** | **Type** | **Description** |
|------------|----------|-----------------|
| `mt5_margin_level` | Gauge | Account margin level |
| `mt5_account_equity` | Gauge | Current account equity |
| `mt5_account_balance` | Gauge | Current account balance |
| `mt5_drawdown` | Gauge | Max drawdown in percentage |
| `mt5_leverage` | Gauge | Current leverage level |

---

### ✅ **Connectivity Metrics**  
| **Metric** | **Type** | **Description** |
|------------|----------|-----------------|
| `mt5_connection_state` | Gauge | 1 = Connected, 0 = Disconnected |
| `mt5_connection_latency` | Histogram | Connection latency (ms) |
| `mt5_connection_loss_count` | Counter | Number of connection failures |
| `mt5_broker_connection_latency` | Histogram | Time to connect to broker |
| `mt5_network_throughput` | Gauge | Data rate over network (Mbps) |

---

### ✅ **Performance Metrics**  
| **Metric** | **Type** | **Description** |
|------------|----------|-----------------|
| `mt5_cpu_usage` | Gauge | CPU usage by MT5 process |
| `mt5_memory_usage` | Gauge | Memory usage by MT5 process |
| `mt5_load_average` | Gauge | Load average of MT5 instance |
| `mt5_thread_count` | Gauge | Number of active MT5 threads |

---

## 🚦 **3. Health Check Design**  
Each MT5 instance will send a health check signal to Redis every **10 seconds**.  
- Health check state stored in: `instance_health:<instance_id>`  
- Health check latency stored in: `health_check_latency:<instance_id>`  

### ✅ **Health Check Response**  
| **State** | **Response** |
|----------|--------------|
| `PASS` | Instance is healthy |
| `FAIL` | Instance is unreachable |
| `DEGRADED` | Instance is alive but under load |

---

## 🚀 **4. Auto-Recovery Design**  
✅ If MT5 instance is in `FAIL` state:  
1. Attempt to restart instance.  
2. If restart fails → Attempt recovery.  
3. If recovery fails → Notify admin + trigger fallback.  
4. If multiple recovery attempts fail → Isolate instance.  

✅ If MT5 instance is in `DEGRADED` state:  
1. Monitor for load issues (CPU/Memory).  
2. Attempt to reduce instance load.  
3. Throttle trade processing if needed.  

✅ If MT5 instance is `DOWN`:  
1. Attempt auto-restart.  
2. If down for 5 minutes → Trigger admin notification.  

---

## 🚨 **5. Prometheus Alerts**  
| **Alert Type** | **Trigger** | **Action** |
|---------------|-------------|------------|
| **High CPU Usage** | CPU > 80% for 10s | Notify admin + throttle execution |
| **High Memory Usage** | Memory > 80% for 10s | Notify admin + throttle execution |
| **Instance Failure** | Transition to `FAIL` | Attempt auto-recovery |
| **Execution Latency High** | Latency > 200ms | Log and investigate |
| **Network Failure** | Latency > 500ms | Attempt reconnect + notify |
| **Slippage Out of Range** | Slippage > 3 pips | Log and notify |
| **Drawdown High** | Drawdown > 20% | Notify admin + throttle trading |

---

## 🏆 **Next Step:**  
✅ Finalize Redis key structure for health check data.  
✅ Finalize Prometheus scraping logic.  
✅ Finalize Redis-DuckDB state sync strategy.  
✅ Proceed to **MT5 Edge Case Handling**.  

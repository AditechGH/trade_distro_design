# **Trade Distribution System – Complete Schema Plan**  

This is the comprehensive schema plan for the Trade Distribution System. The schema is structured into **eight functional layers**, each handling a specific aspect of state, execution, and reporting. This document outlines the purpose, table definitions, and sync strategy for each layer.  

---

## 🚀 **1. Trade State Layer (Live State from Redis)**
### ✅ **Purpose:**
- To store real-time state information of trades, including open positions, pending orders, and execution state.  
- State information is temporary — Redis will discard it after execution or completion.  

### ✅ **Tables:**
| Table Name | Purpose | Primary Key | Notes |
|------------|---------|-------------|-------|
| **trade_state** | Stores real-time trade state (open trades, pending orders) | `trade_id (UUID)` | Lives in Redis |
| **account_state** | Stores real-time account state (balance, margin, leverage) | `account_id (UUID)` | Lives in Redis |
| **execution_state** | Stores execution state (partial fills, rejections) | `trade_id (UUID)` | Lives in Redis |
| **broker_state** | Stores broker-level state (spreads, fees) | `broker_id (UUID)` | Lives in Redis |  

---

## 🎯 **2. Instance State Layer (Instance Management)**
### ✅ **Purpose:**
- To store the state of the MetaTrader 5 instances, including CPU/memory usage, uptime, and health status.  
- State information is temporary but can be retained for monitoring and troubleshooting.  

### ✅ **Tables:**
| Table Name | Purpose | Primary Key | Notes |
|------------|---------|-------------|-------|
| **instance_state** | Stores the current state of each MT5 instance | `instance_id (UUID)` | Lives in Redis |
| **instance_metrics** | Stores resource usage (CPU, memory) per instance | `instance_id (UUID)` | Lives in Redis |
| **instance_health** | Stores health status and uptime per instance | `instance_id (UUID)` | Lives in Redis |  

---

## 📡 **3. Signal Stream Layer (Live Signals from TradingView)**
### ✅ **Purpose:**
- To store incoming trading signals from TradingView for processing and execution.  
- Signals are temporary — they will be processed and removed from Redis after execution.  

### ✅ **Tables:**
| Table Name | Purpose | Primary Key | Notes |
|------------|---------|-------------|-------|
| **trade_signal** | Stores trade signals (buy/sell, lot size) | `signal_id (UUID)` | Lives in Redis |
| **signal_execution_status** | Stores execution status of each signal | `signal_id (UUID)` | Lives in Redis |
| **signal_metrics** | Stores metrics (response time, latency) for signal processing | `signal_id (UUID)` | Lives in Redis |
| **signal_rejection** | Stores rejected or failed signals for troubleshooting | `signal_id (UUID)` | Lives in Redis |  

---

## 📈 **4. Result Stream Layer (Execution Results)**
### ✅ **Purpose:**
- To store the execution results of trades.  
- Lives temporarily in Redis but is permanently stored in DuckDB and ClickHouse for analysis.  

### ✅ **Tables:**
| Table Name | Purpose | Primary Key | Notes |
|------------|---------|-------------|-------|
| **trade_result** | Stores execution results (PnL, slippage) | `result_id (UUID)` | Synced to DuckDB and ClickHouse |
| **result_metrics** | Stores performance data (execution latency, broker slippage) | `result_id (UUID)` | Synced to DuckDB and ClickHouse |
| **trade_reconciliation** | Stores reconciliation status (success/fail) | `trade_id (UUID)` | Synced to DuckDB and ClickHouse |  

---

## 🗃️ **5. Historical Data Layer (DuckDB + ClickHouse)**
### ✅ **Purpose:**
- To store historical trade data and performance metrics.  
- Data is written to DuckDB by Rust and synced to ClickHouse by Python.  

### ✅ **Tables:**
| Table Name | Purpose | Primary Key | Notes |
|------------|---------|-------------|-------|
| **trades** | Stores full trade history (complete executions) | `trade_id (UUID)` | Stored in DuckDB + ClickHouse |
| **trade_performance** | Stores historical performance metrics (PnL, slippage) | `trade_id (UUID)` | Stored in DuckDB + ClickHouse |
| **broker_performance** | Stores broker performance over time (fees, slippage) | `broker_id (UUID)` | Stored in DuckDB + ClickHouse |
| **account_performance** | Stores account-level performance (profit/loss) | `account_id (UUID)` | Stored in DuckDB + ClickHouse |
| **instance_performance** | Stores instance-level performance over time | `instance_id (UUID)` | Stored in DuckDB + ClickHouse |
| **trade_reconciliation_log** | Stores trade reconciliation results (success/fail) | `trade_id (UUID)` | Stored in DuckDB + ClickHouse |
| **trade_failure_log** | Stores details of failed trades | `trade_id (UUID)` | Stored in DuckDB + ClickHouse |
| **commission_log** | Stores commission and fee data | `trade_id (UUID)` | Stored in DuckDB + ClickHouse |  

---

## 🔎 **6. Configuration and Settings Layer (DuckDB + Redis)**
### ✅ **Purpose:**
- To store configuration and user settings.  
- Configuration data is critical for trade execution and user preferences.  

### ✅ **Tables:**
| Table Name | Purpose | Primary Key | Notes |
|------------|---------|-------------|-------|
| **account_config** | Stores account-level settings (lot size, SL/TP) | `account_id (UUID)` | Stored in DuckDB |
| **instance_config** | Stores instance-level settings (MT5 config) | `instance_id (UUID)` | Stored in DuckDB |
| **broker_config** | Stores broker-level settings (spread settings) | `broker_id (UUID)` | Stored in DuckDB |
| **signal_config** | Stores signal handling settings | `config_id (UUID)` | Stored in DuckDB |  

---

## 🛡️ **7. Security and Access Layer (DuckDB + Redis)**
### ✅ **Purpose:**
- To store user authentication, role-based access, and permission data.  
- Data is retained for audit and security logging.  

### ✅ **Tables:**
| Table Name | Purpose | Primary Key | Notes |
|------------|---------|-------------|-------|
| **user** | Stores user login info | `user_id (UUID)` | Stored in DuckDB |
| **user_role** | Stores user roles (admin, user) | `role_id (UUID)` | Stored in DuckDB |
| **user_permission** | Stores specific permissions for each user | `permission_id (UUID)` | Stored in DuckDB |
| **login_audit** | Logs login attempts and access | `login_id (UUID)` | Stored in DuckDB |
| **token_store** | Stores OAuth tokens and session state | `session_id (UUID)` | Stored in Redis |  

---

## 🚀 **8. Monitoring and Logging Layer (DuckDB + ClickHouse)**
### ✅ **Purpose:**
- To store system-level monitoring data.  
- Performance metrics, system state, and execution health are logged.  

### ✅ **Tables:**
| Table Name | Purpose | Primary Key | Notes |
|------------|---------|-------------|-------|
| **system_event_log** | Stores system-level events (start, stop, error) | `event_id (UUID)` | Stored in DuckDB + ClickHouse |
| **performance_log** | Stores system performance over time | `instance_id (UUID)` | Stored in DuckDB + ClickHouse |
| **alert_log** | Stores system alerts (failures, latency issues) | `alert_id (UUID)` | Stored in DuckDB + ClickHouse |  

---

## 🏆 **Summary**
| Layer | Tables | Storage Location |
|-------|--------|------------------|
| **Trade State Layer** | 4 tables | Redis |
| **Instance State Layer** | 3 tables | Redis |
| **Signal Stream Layer** | 4 tables | Redis |
| **Result Stream Layer** | 3 tables | Redis, DuckDB, ClickHouse |
| **Historical Data Layer** | 7 tables | DuckDB, ClickHouse |
| **Configuration Layer** | 4 tables | DuckDB |
| **Security Layer** | 5 tables | DuckDB, Redis |
| **Monitoring Layer** | 3 tables | DuckDB, ClickHouse |  

---

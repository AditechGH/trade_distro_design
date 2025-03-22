# **Historical Data Layer – Definition for Aggregated Performance Metrics**   
✅ Real-time rolling aggregation for broker and account performance.  
✅ Periodic snapshots for historical benchmarking and backtesting.  
✅ Clean separation between transactional data and aggregated data.  
✅ Efficient update mechanisms without recalculating from scratch.  

---

## ✅ **Architectural Strategy for Historical Data Layer**  
### 🔹 **1. Real-time Rolling Aggregation**  
- Use incremental updates → Update aggregate values directly based on new trades.  
- Avoid recalculating aggregates from scratch → Use weighted formulas.  

### 🔹 **2. Periodic Snapshot Strategy**  
- Take periodic snapshots of rolling aggregates (daily, weekly).  
- Store snapshots in both **DuckDB** (for local queries) and **ClickHouse** (for cloud analytics).  

### 🔹 **3. Backfilling Strategy**  
- If data loss or inconsistency occurs → Backfill from `trades` table.  
- Backfill = Recalculate aggregates based on raw trade history.  

---

## ✅ **Updated Tables for Historical Data Layer**
| Table Name | Purpose | State Type | Storage |
|------------|---------|------------|---------|
| **trades** | Full trade history (executed trades) | Persistent | DuckDB + ClickHouse |
| **trade_performance** | Real-time + historical performance metrics | Persistent + Incremental | DuckDB + ClickHouse |
| **broker_performance** | Broker-level aggregated data | Incremental + Snapshot | DuckDB + ClickHouse |
| **account_performance** | Account-level aggregated data | Incremental + Snapshot | DuckDB + ClickHouse |
| **instance_performance** | MT5 instance performance | Incremental + Snapshot | DuckDB + ClickHouse |
| **trade_reconciliation_log** | Reconciliation status and discrepancies | Persistent | DuckDB + ClickHouse |
| **trade_failure_log** | Failed trades and error handling | Persistent | DuckDB + ClickHouse |
| **commission_log** | Commission and fee data | Persistent | DuckDB + ClickHouse |  

---

## 🔎 **1. `trades` Table**
### ✅ **Purpose:**
- Store the full trade history (including execution details).  
- Serve as the **source of truth** for rebuilding aggregate data.  

### ✅ **Schema Design (SQL):**
```sql
CREATE TABLE trades (
    trade_id UUID PRIMARY KEY,
    account_id UUID NOT NULL,
    broker_id UUID NOT NULL,
    symbol STRING NOT NULL,
    direction STRING NOT NULL,                  -- 'buy' or 'sell'
    lot_size FLOAT NOT NULL,
    execution_price FLOAT NOT NULL,
    requested_price FLOAT NOT NULL,
    stop_loss FLOAT,
    take_profit FLOAT,
    pnl FLOAT,
    state STRING NOT NULL,                      -- 'executed', 'partially_filled', 'failed'
    slippage FLOAT,
    commission FLOAT,
    fees FLOAT,
    execution_latency_ms INT,
    executed_at TIMESTAMP NOT NULL,
    closed_at TIMESTAMP
);
```

---

### ✅ **Rolling Aggregation Update Example:**
👉 When a new trade is recorded in `trades` → The following happens:  
1. **Account-level performance** is updated (P&L, win rate).  
2. **Broker-level performance** is updated (slippage, commission).  
3. **Instance-level performance** is updated (latency, spread).  

---

### ✅ **Example Rolling Update:**
```sql
UPDATE broker_performance
SET 
    avg_slippage = ((avg_slippage * total_trades) + NEW.slippage) / (total_trades + 1),
    total_trades = total_trades + 1
WHERE broker_id = NEW.broker_id;
```

---

## 🔎 **2. `trade_performance` Table**
### ✅ **Purpose:**
- Store detailed performance data for each trade.  
- Record execution speed, latency, and slippage for benchmarking.  
- Serve as a source for broker and account benchmarking.  

---

### ✅ **Schema Design (SQL):**
```sql
CREATE TABLE trade_performance (
    trade_id UUID PRIMARY KEY,
    account_id UUID NOT NULL,
    broker_id UUID NOT NULL,
    execution_latency_ms INT,
    broker_latency_ms INT,
    network_latency_ms INT,
    slippage FLOAT,
    pnl FLOAT,
    commission FLOAT,
    fees FLOAT,
    executed_at TIMESTAMP NOT NULL
);
```

---

### ✅ **Rolling Aggregation Update Example:**
1. Incrementally update broker and account performance from this table.  
2. For each new row in `trade_performance` → Update aggregate stats using weighted average formulas.  

---

## 🔎 **3. `broker_performance` Table**
### ✅ **Purpose:**
- Track broker-wide performance based on execution quality.  
- Aggregate across all trades executed via the broker.  
- Support rolling updates + periodic snapshots.  

---

### ✅ **Schema Design (SQL):**
```sql
CREATE TABLE broker_performance (
    broker_id UUID PRIMARY KEY,
    avg_slippage FLOAT,                         -- Weighted slippage
    avg_commission FLOAT,                       -- Weighted commission
    avg_execution_latency_ms INT,               -- Weighted latency
    total_trades INT,
    win_rate FLOAT,
    last_updated_at TIMESTAMP NOT NULL
);
```

---

### ✅ **Rolling Update Formula:**  
For slippage (using weighted average):  
```sql
UPDATE broker_performance
SET 
    avg_slippage = ((avg_slippage * total_trades) + NEW.slippage) / (total_trades + 1),
    total_trades = total_trades + 1
WHERE broker_id = NEW.broker_id;
```

---

## 🔎 **4. `account_performance` Table**
### ✅ **Purpose:**
- Track profitability and performance of individual accounts.  
- Monitor win rate and PnL over time.  

---

### ✅ **Schema Design (SQL):**
```sql
CREATE TABLE account_performance (
    account_id UUID PRIMARY KEY,
    total_pnl FLOAT,
    total_commission FLOAT,
    total_fees FLOAT,
    total_trades INT,
    win_rate FLOAT,
    last_updated_at TIMESTAMP NOT NULL
);
```

---

### ✅ **Rolling Update Formula:**  
For win rate (using weighted average):  
```sql
UPDATE account_performance
SET 
    total_pnl = total_pnl + NEW.pnl,
    total_trades = total_trades + 1,
    win_rate = ((win_rate * (total_trades - 1)) + 
               CASE WHEN NEW.pnl > 0 THEN 1 ELSE 0 END) 
               / total_trades
WHERE account_id = NEW.account_id;
```

---

## 🔎 **5. `instance_performance` Table**
### ✅ **Purpose:**
- Track instance-level resource usage and health.  
- Aggregated at the instance level over time.  

---

### ✅ **Schema Design (SQL):**
```sql
CREATE TABLE instance_performance (
    instance_id UUID PRIMARY KEY,
    avg_cpu_usage FLOAT,
    avg_memory_usage FLOAT,
    avg_execution_latency_ms INT,
    total_trades INT,
    last_updated_at TIMESTAMP NOT NULL
);
```

---

## 🔎 **6. `trade_reconciliation_log` Table**
### ✅ **Purpose:**
- Track reconciliation between expected and actual trade outcomes.  
- Record discrepancy and reconciliation status.  

---

### ✅ **Schema Design (SQL):**
```sql
CREATE TABLE trade_reconciliation_log (
    trade_id UUID PRIMARY KEY,
    reconciliation_status STRING NOT NULL,        -- 'success', 'failed'
    discrepancy FLOAT,
    rejection_reason STRING,
    created_at TIMESTAMP NOT NULL
);
```

---

## 🔎 **7. `trade_failure_log` Table**
### ✅ **Purpose:**
- Record failed trades for auditing and debugging.  

---

### ✅ **Schema Design (SQL):**
```sql
CREATE TABLE trade_failure_log (
    trade_id UUID PRIMARY KEY,
    failure_reason STRING NOT NULL,
    created_at TIMESTAMP NOT NULL
);
```

---

## 🔎 **8. `commission_log` Table**
### ✅ **Purpose:**
- Record broker commissions at the trade level.  
- Track how much was paid for each trade.  

---

### ✅ **Schema Design (SQL):**
```sql
CREATE TABLE commission_log (
    trade_id UUID PRIMARY KEY,
    commission FLOAT NOT NULL,
    created_at TIMESTAMP NOT NULL
);
```

---

## ✅ **Real-time Aggregation + Snapshot Strategy:**
| Type | Approach | Frequency |
|-------|----------|------------|
| **Rolling Aggregation** | Weighted average updates | Real-time |
| **Snapshot** | Periodic table snapshot | Daily/Weekly |
| **Backfilling** | On-demand | As needed |

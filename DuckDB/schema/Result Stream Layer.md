# **Result Stream Layer – Schema Definition**  
The **Result Stream Layer** will store the execution results of trades. This layer represents the **final state** of trades — after they are executed or closed — and will reside in both **Redis** (short-term) and **DuckDB + ClickHouse** (permanent storage).  

This layer will handle:  
✅ Real-time execution results and partial fills.  
✅ Slippage, profit/loss (PnL), and execution latency.  
✅ Reconciliation between intended and actual trade outcomes.  

---

## ✅ **Purpose of Result Stream Layer:**
1. Track execution results of trades (PnL, slippage, execution price).  
2. Provide reconciliation data between expected and actual outcomes.  
3. Monitor execution latency and broker-side performance.  
4. Sync real-time execution data from Redis → DuckDB → ClickHouse.  

---

## 🚀 **Tables in the Result Stream Layer**  
| Table Name | Purpose | State Type | Storage |
|------------|---------|------------|---------|
| **trade_result** | Stores execution results (PnL, slippage) | Temporary (Redis) + Persistent | Redis → DuckDB → ClickHouse |
| **result_metrics** | Tracks performance data (execution latency, broker slippage) | Temporary (Redis) + Persistent | Redis → DuckDB → ClickHouse |
| **trade_reconciliation** | Stores reconciliation status and discrepancies | Persistent | DuckDB + ClickHouse |  

---

## 🔎 **1. `trade_result` Table**
### ✅ **Purpose:**
- Store the execution outcome of a trade.  
- Track slippage, commission, and fees.  
- Record the final state of the trade (executed, partially filled, or rejected).  
- Sync to DuckDB and ClickHouse for long-term storage.  

---

### ✅ **Schema Design (SQL):**
```sql
CREATE TABLE trade_result (
    result_id UUID PRIMARY KEY,                  -- Unique ID for the result
    trade_id UUID NOT NULL,                      -- Reference to trade_state
    symbol STRING NOT NULL,                      -- Trading symbol
    direction STRING NOT NULL,                   -- 'buy', 'sell'
    lot_size FLOAT NOT NULL,                     -- Size of the order
    execution_price FLOAT NOT NULL,              -- Actual execution price
    requested_price FLOAT,                       -- Price requested at order time
    slippage FLOAT,                              -- Execution slippage
    pnl FLOAT,                                   -- Profit/loss (in account currency)
    commission FLOAT,                            -- Broker commission for execution
    fees FLOAT,                                  -- Additional execution fees
    state STRING NOT NULL,                       -- 'executed', 'partially_filled', 'failed'
    executed_at TIMESTAMP NOT NULL               -- Timestamp of execution
);
```

---

### ✅ **Redis Adaptation:**
1. **Key:** `result:{result_id}`  
2. **Value Type:** Redis Hash  

### ✅ Example Redis Command:
```bash
HSET result:123e4567-e89b-12d3-a456-426614174000 \
    trade_id "456e1234-e89b-12d3-a456-426614174123" \
    symbol "EURUSD" \
    direction "buy" \
    lot_size 1.5 \
    execution_price 1.12340 \
    requested_price 1.12345 \
    slippage -0.00005 \
    pnl 75.0 \
    commission 1.5 \
    fees 0.5 \
    state "executed" \
    executed_at "2025-03-22T10:16:00Z"
```

---

### ✅ **Example Retrieval:**
```bash
HGETALL result:123e4567-e89b-12d3-a456-426614174000
```
**Output:**
```bash
1) "trade_id"
2) "456e1234-e89b-12d3-a456-426614174123"
3) "symbol"
4) "EURUSD"
5) "direction"
6) "buy"
7) "lot_size"
8) "1.5"
9) "execution_price"
10) "1.12340"
11) "requested_price"
12) "1.12345"
13) "slippage"
14) "-0.00005"
15) "pnl"
16) "75.0"
17) "commission"
18) "1.5"
19) "fees"
20) "0.5"
21) "state"
22) "executed"
```

---

### ✅ **Example State Transitions:**
| Initial State | Transition | Final State |
|---------------|------------|-------------|
| `pending` → `executed` | Trade filled at requested price | `executed` |
| `executed` → `partially_filled` | Partial fill due to liquidity | `partially_filled` |
| `executed` → `failed` | Trade rejected due to price gap | `failed` |

---

### ✅ **Example State Update:**
```bash
HSET result:123e4567-e89b-12d3-a456-426614174000 state "partially_filled"
```

---

## 🔎 **2. `result_metrics` Table**
### ✅ **Purpose:**
- Track execution latency and performance.  
- Measure broker-side slippage and execution quality.  
- Provide data for broker performance benchmarking.  

---

### ✅ **Schema Design (SQL):**
```sql
CREATE TABLE result_metrics (
    result_id UUID PRIMARY KEY,                  -- Reference to result_id
    execution_latency_ms INT,                    -- Time from signal to execution
    broker_latency_ms INT,                       -- Time taken by broker to fill order
    network_latency_ms INT,                      -- Time taken by network to transmit order
    slippage FLOAT,                              -- Slippage compared to requested price
    created_at TIMESTAMP NOT NULL                -- Timestamp of metrics recording
);
```

---

### ✅ **Redis Adaptation:**
1. **Key:** `result_metrics:{result_id}`  
2. **Value Type:** Redis Hash  

### ✅ Example Redis Command:
```bash
HSET result_metrics:123e4567-e89b-12d3-a456-426614174000 \
    execution_latency_ms 120 \
    broker_latency_ms 50 \
    network_latency_ms 20 \
    slippage -0.00005 \
    created_at "2025-03-22T10:16:05Z"
```

---

### ✅ **Example Retrieval:**
```bash
HGETALL result_metrics:123e4567-e89b-12d3-a456-426614174000
```

---

## 🔎 **3. `trade_reconciliation` Table**
### ✅ **Purpose:**
- Store reconciliation status of trades.  
- Handle discrepancies between requested and executed price.  
- Track reconciliation failures for audit purposes.  

---

### ✅ **Schema Design (SQL):**
```sql
CREATE TABLE trade_reconciliation (
    trade_id UUID PRIMARY KEY,                  -- Reference to trade_id
    reconciliation_status STRING NOT NULL,       -- 'success', 'failed'
    discrepancy FLOAT,                           -- Difference between requested and executed price
    rejection_reason STRING,                     -- Reason for rejection (if applicable)
    reconciled_at TIMESTAMP NOT NULL             -- Timestamp of reconciliation
);
```

---

### ✅ **Redis Adaptation:**
1. **Key:** `reconciliation:{trade_id}`  
2. **Value Type:** Redis Hash  

### ✅ Example Redis Command:
```bash
HSET reconciliation:123e4567-e89b-12d3-a456-426614174000 \
    reconciliation_status "failed" \
    discrepancy -0.00010 \
    rejection_reason "Slippage too high" \
    reconciled_at "2025-03-22T10:17:00Z"
```

---

### ✅ **Example Retrieval:**
```bash
HGETALL reconciliation:123e4567-e89b-12d3-a456-426614174000
```

---

## 🏆 **Summary of Result Stream Layer Schema**
| Table | Purpose | State Type | Storage Location |
|-------|---------|------------|------------------|
| **trade_result** | Execution results | Temporary → Persistent | Redis → DuckDB → ClickHouse |
| **result_metrics** | Performance data | Temporary → Persistent | Redis → DuckDB → ClickHouse |
| **trade_reconciliation** | Reconciliation data | Persistent | DuckDB → ClickHouse |  

---

## ✅ **Design Strengths:**
✅ Fast real-time tracking of execution results.  
✅ Clean separation between execution, reconciliation, and metrics.  
✅ Direct sync from Redis → DuckDB → ClickHouse.  
✅ Supports high-frequency order processing without data loss.  

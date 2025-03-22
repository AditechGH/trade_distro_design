# **Signal Stream Layer – Schema Definition**  
The **Signal Stream Layer** will store real-time trading signals from TradingView and other input sources. Since trading signals are inherently real-time and require fast processing, they will reside entirely in **Redis**.  

This layer will handle both:  
✅ **Signal generation** → Incoming buy/sell signals.  
✅ **Signal execution** → Processing status, execution outcome, and error handling.  

---

## ✅ **Purpose of Signal Stream Layer:**
1. Store incoming trading signals for immediate processing.  
2. Track execution status and response latency.  
3. Handle rejected signals for retry or diagnostic purposes.  
4. Support parallel signal processing for low-latency execution.  

---

## 🚀 **Tables in the Signal Stream Layer**  
| Table Name | Purpose | State Type | Storage |
|------------|---------|------------|---------|
| **trade_signal** | Stores incoming trade signals | Temporary | Redis |
| **signal_execution_status** | Stores execution status of each signal | Temporary | Redis |
| **signal_metrics** | Stores response time and latency for signal processing | Temporary | Redis |
| **signal_rejection** | Stores rejected or failed signals | Temporary | Redis |  

---

## 🔎 **1. `trade_signal` Table**
### ✅ **Purpose:**
- Store raw incoming trade signals.  
- Track symbol, order type, direction, and lot size.  
- Handle both market and pending orders.  

### ✅ **Schema Design (SQL):**
```sql
CREATE TABLE trade_signal (
    signal_id UUID PRIMARY KEY,                  -- Unique ID for the signal
    symbol STRING NOT NULL,                      -- Trading symbol (EURUSD, XAUUSD)
    order_type STRING NOT NULL,                  -- 'market', 'limit', 'stop'
    direction STRING NOT NULL,                   -- 'buy', 'sell'
    lot_size FLOAT NOT NULL,                     -- Lot size of the order
    price FLOAT,                                 -- Expected price (NULL for market orders)
    stop_loss FLOAT,                             -- Stop loss level (optional)
    take_profit FLOAT,                           -- Take profit level (optional)
    expiration_time TIMESTAMP,                   -- Order expiration time (NULL if market order)
    status STRING NOT NULL,                      -- 'pending', 'processing', 'completed', 'failed'
    created_at TIMESTAMP NOT NULL                -- Timestamp of signal creation
);
```

---

### ✅ **Redis Adaptation:**
1. **Key:** `signal:{signal_id}`  
2. **Value Type:** Redis Hash  

### ✅ Example Redis Command:
```bash
HSET signal:123e4567-e89b-12d3-a456-426614174000 \
    symbol "EURUSD" \
    order_type "market" \
    direction "buy" \
    lot_size 1.5 \
    price 1.12345 \
    stop_loss 1.12000 \
    take_profit 1.13000 \
    expiration_time "" \
    status "pending" \
    created_at "2025-03-22T10:15:30Z"
```

---

### ✅ **Example Retrieval:**
```bash
HGETALL signal:123e4567-e89b-12d3-a456-426614174000
```
**Output:**
```bash
1) "symbol"
2) "EURUSD"
3) "order_type"
4) "market"
5) "direction"
6) "buy"
7) "lot_size"
8) "1.5"
9) "price"
10) "1.12345"
11) "status"
12) "pending"
```

---

### ✅ **Example State Update:**
```bash
HSET signal:123e4567-e89b-12d3-a456-426614174000 status "processing"
```

---

### ✅ **Example State Transitions:**
| Initial State | Transition | Final State |
|---------------|------------|-------------|
| `pending` → `processing` | Signal received and being processed | `processing` |
| `processing` → `completed` | Signal executed successfully | `completed` |
| `processing` → `failed` | Rejection due to price gap | `failed` |
| `pending` → `failed` | Expired signal or invalid order | `failed` |

---

### ✅ **Indexes for Fast Lookups:**
Use **Redis Sets** to track signal state:  
1. Add to set on creation:  
```bash
SADD signal:state:pending "123e4567-e89b-12d3-a456-426614174000"
```

2. Remove from set on completion:  
```bash
SREM signal:state:pending "123e4567-e89b-12d3-a456-426614174000"
```

3. Get all pending signals:  
```bash
SMEMBERS signal:state:pending
```

---

## 🔎 **2. `signal_execution_status` Table**
### ✅ **Purpose:**
- Track execution status of a signal.  
- Handle partial fills and price slippage.  
- Store final execution price and order fill status.  

### ✅ **Schema Design (SQL):**
```sql
CREATE TABLE signal_execution_status (
    signal_id UUID PRIMARY KEY,                  -- Signal reference
    status STRING NOT NULL,                      -- 'processing', 'completed', 'failed'
    execution_price FLOAT,                       -- Actual execution price
    fill_percentage FLOAT DEFAULT 0,             -- Percentage of order filled
    slippage FLOAT,                              -- Difference from expected price
    rejection_reason STRING,                     -- Reason for rejection (if applicable)
    executed_at TIMESTAMP                        -- Timestamp of execution
);
```

---

### ✅ **Redis Adaptation:**
1. **Key:** `signal_execution_status:{signal_id}`  
2. **Value Type:** Redis Hash  

### ✅ Example Redis Command:
```bash
HSET signal_execution_status:123e4567-e89b-12d3-a456-426614174000 \
    status "completed" \
    execution_price 1.12340 \
    fill_percentage 100.0 \
    slippage -0.00005 \
    rejection_reason "" \
    executed_at "2025-03-22T10:16:00Z"
```

---

## 🔎 **3. `signal_metrics` Table**
### ✅ **Purpose:**
- Track signal processing performance.  
- Measure response time and execution latency.  

### ✅ **Schema Design (SQL):**
```sql
CREATE TABLE signal_metrics (
    signal_id UUID PRIMARY KEY,                  -- Signal reference
    processing_time_ms INT,                      -- Time to process signal
    execution_latency_ms INT,                    -- Time from signal to execution
    created_at TIMESTAMP NOT NULL                -- Timestamp of metrics recording
);
```

---

### ✅ **Redis Adaptation:**
1. **Key:** `signal_metrics:{signal_id}`  
2. **Value Type:** Redis Hash  

### ✅ Example Redis Command:
```bash
HSET signal_metrics:123e4567-e89b-12d3-a456-426614174000 \
    processing_time_ms 12 \
    execution_latency_ms 100 \
    created_at "2025-03-22T10:16:05Z"
```

---

## 🔎 **4. `signal_rejection` Table**
### ✅ **Purpose:**
- Track rejected signals for troubleshooting.  
- Store rejection reason and failed execution state.  

### ✅ **Schema Design (SQL):**
```sql
CREATE TABLE signal_rejection (
    signal_id UUID PRIMARY KEY,                  -- Signal reference
    rejection_reason STRING NOT NULL,             -- Reason for rejection
    rejected_at TIMESTAMP NOT NULL                -- Timestamp of rejection
);
```

---

### ✅ **Redis Adaptation:**
1. **Key:** `signal_rejection:{signal_id}`  
2. **Value Type:** Redis Hash  

### ✅ Example Redis Command:
```bash
HSET signal_rejection:123e4567-e89b-12d3-a456-426614174000 \
    rejection_reason "Invalid stop-loss" \
    rejected_at "2025-03-22T10:16:10Z"
```

---

## 🏆 **Summary of Signal Stream Layer Schema**
| Table | Purpose | State Type | Storage Location |
|-------|---------|------------|------------------|
| **trade_signal** | Incoming signals | Temporary | Redis |
| **signal_execution_status** | Execution status | Temporary | Redis |
| **signal_metrics** | Performance tracking | Temporary | Redis |
| **signal_rejection** | Rejected signals | Temporary | Redis |  

---

## ✅ **Design Strengths:**
✅ Parallel signal processing.  
✅ Fast real-time state handling.  
✅ Efficient rejection and error tracking.  
✅ Clean separation between processing and execution.  

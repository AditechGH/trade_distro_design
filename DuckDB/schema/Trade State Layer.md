# **Trade State Layer – Schema Definition**  
The **Trade State Layer** will store real-time state information for trades, accounts, execution state, and broker state. Since this layer handles **live state**, it will reside entirely in **Redis** for low-latency performance and fast access.  

---

## ✅ **Purpose of Trade State Layer:**
1. Handle open trade state and execution state in real-time.  
2. Handle partial fills and rejections.  
3. Handle real-time account and broker state changes.  
4. Serve as the foundation for the execution module and trade processing.  

---

## 🚀 **Tables in the Trade State Layer**  
| Table Name | Purpose | State Type | Storage |
|------------|---------|------------|---------|
| **trade_state** | Stores real-time trade state (open, closed, failed) | Temporary | Redis |
| **account_state** | Stores real-time account state (balance, margin) | Temporary | Redis |
| **execution_state** | Stores execution state (partial fills, rejections) | Temporary | Redis |
| **broker_state** | Stores broker-level state (spreads, fees) | Temporary | Redis |  

---

## 🔎 **1. `trade_state` Table**
### ✅ **Purpose:**
- Store real-time state of open trades and pending orders.  
- Track state changes → `pending` → `executing` → `executed` → `closed`.  
- Handle partial fills and stop-loss/take-profit changes.  

### ✅ **Schema:**
```sql
CREATE TABLE trade_state (
    trade_id UUID PRIMARY KEY,                  -- Unique ID for the trade
    account_id UUID NOT NULL,                   -- Associated account ID
    broker_id UUID NOT NULL,                    -- Broker ID
    symbol STRING NOT NULL,                     -- Trading symbol (EURUSD, XAUUSD)
    order_type STRING NOT NULL,                 -- 'market', 'limit', 'stop'
    direction STRING NOT NULL,                  -- 'buy', 'sell'
    lot_size FLOAT NOT NULL,                    -- Size of the trade
    requested_price FLOAT NOT NULL,             -- Requested price for order
    execution_price FLOAT,                      -- Actual execution price (null if not yet executed)
    stop_loss FLOAT,                            -- Stop loss level (optional)
    take_profit FLOAT,                          -- Take profit level (optional)
    partial_fill_percentage FLOAT DEFAULT 0,    -- % of order filled (used for partial fills)
    state STRING NOT NULL,                      -- 'pending', 'executing', 'executed', 'failed', 'closed'
    execution_latency_ms INT,                   -- Time from order to execution (ms)
    created_at TIMESTAMP NOT NULL,              -- Time order was created
    updated_at TIMESTAMP NOT NULL               -- Last updated timestamp
);

-- Indexes for faster query performance:
CREATE INDEX idx_trade_state_account ON trade_state(account_id);
CREATE INDEX idx_trade_state_symbol ON trade_state(symbol);
CREATE INDEX idx_trade_state_state ON trade_state(state);
```

---

### ✅ **Example State Transitions:**
| Initial State | Transition | Final State |
|---------------|------------|-------------|
| `pending` → `executing` | Order opened | `executing` |
| `executing` → `executed` | Order fully filled | `executed` |
| `executing` → `failed` | Rejection due to slippage | `failed` |
| `executed` → `closed` | Order manually closed | `closed` |
| `executed` → `closed` | Stop-loss or take-profit hit | `closed` |

---

### ✅ **Example Record:**
```sql
INSERT INTO trade_state VALUES (
    '123e4567-e89b-12d3-a456-426614174000',
    '550e8400-e29b-41d4-a716-446655440000',
    '776e4400-e29b-41d4-a716-446655440001',
    'EURUSD',
    'market',
    'buy',
    1.5,
    1.12345,
    NULL,
    1.12000,
    1.13000,
    0,
    'pending',
    NULL,
    '2025-03-22T10:15:30Z',
    '2025-03-22T10:15:30Z'
);
```

---

### ✅ **Design Notes:**
✔️ Trade ID is the unique key → Used for state transitions.  
✔️ `state` field → Drives execution and state change.  
✔️ `execution_price` = `NULL` → Trade not yet executed.  
✔️ Partial fills handled via `partial_fill_percentage` → Real-time tracking.  
✔️ No `FOREIGN KEY` constraints → Redis doesn’t support relational links.  

---

## 🔎 **2. `account_state` Table**
### ✅ **Purpose:**
- Store real-time account information during execution.  
- Update in real-time based on open/closed trades.  
- Sync to DuckDB after state changes.  

### ✅ **Schema:**
```sql
CREATE TABLE account_state (
    account_id UUID PRIMARY KEY,                -- Unique ID for account
    broker_id UUID NOT NULL,                    -- Associated broker
    balance FLOAT NOT NULL,                     -- Account balance
    equity FLOAT NOT NULL,                      -- Account equity
    margin FLOAT NOT NULL,                      -- Used margin
    free_margin FLOAT NOT NULL,                 -- Free margin
    leverage FLOAT NOT NULL,                    -- Leverage
    currency STRING NOT NULL,                   -- Base currency
    last_updated_at TIMESTAMP NOT NULL          -- Last update time
);
```

---

### ✅ **Example Record:**
```sql
INSERT INTO account_state VALUES (
    '550e8400-e29b-41d4-a716-446655440000',
    '776e4400-e29b-41d4-a716-446655440001',
    10000.0,
    10500.0,
    500.0,
    9500.0,
    30,
    'USD',
    '2025-03-22T10:15:30Z'
);
```

---

## 🔎 **3. `execution_state` Table**
### ✅ **Purpose:**
- Store execution-level state.  
- Handle partial fills and rejections.  
- Store execution latency.  

### ✅ **Schema:**
```sql
CREATE TABLE execution_state (
    trade_id UUID PRIMARY KEY,                  -- Trade reference
    state STRING NOT NULL,                      -- 'pending', 'executing', 'failed'
    fill_percentage FLOAT DEFAULT 0,            -- Percentage filled
    slippage FLOAT,                             -- Execution slippage
    execution_latency_ms INT,                   -- Execution latency in ms
    created_at TIMESTAMP NOT NULL               -- Time of execution attempt
);
```

---

### ✅ **Example Record:**
```sql
INSERT INTO execution_state VALUES (
    '123e4567-e89b-12d3-a456-426614174000',
    'executing',
    50.0,
    -0.0002,
    120,
    '2025-03-22T10:15:30Z'
);
```

---

## 🔎 **4. `broker_state` Table**
### ✅ **Purpose:**
- Store real-time broker-level state.  
- Handle spread, commissions, and fees.  
- Provide real-time pricing to execution modules.  

### ✅ **Schema:**
```sql
CREATE TABLE broker_state (
    broker_id UUID PRIMARY KEY,                 -- Broker reference
    symbol STRING NOT NULL,                     -- Trading symbol
    spread FLOAT NOT NULL,                      -- Current spread
    commission FLOAT NOT NULL,                  -- Commission per lot
    fees FLOAT NOT NULL,                        -- Execution fees
    last_updated_at TIMESTAMP NOT NULL          -- Last update time
);
```

---

### ✅ **Example Record:**
```sql
INSERT INTO broker_state VALUES (
    '776e4400-e29b-41d4-a716-446655440001',
    'EURUSD',
    0.0002,
    1.5,
    0.5,
    '2025-03-22T10:15:30Z'
);
```

---

## 🏆 **Summary of Trade State Layer Schema**
| Table | Purpose | State Type | Storage Location |
|-------|---------|------------|------------------|
| **trade_state** | Trade state | Temporary | Redis |
| **account_state** | Account state | Temporary | Redis |
| **execution_state** | Execution state | Temporary | Redis |
| **broker_state** | Broker state | Temporary | Redis |  

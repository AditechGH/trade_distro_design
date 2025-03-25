## ✅ **EA Trade Execution Handling – Design and Flow**  

This section defines how the Expert Advisor (EA) will handle trade execution in real-time, including order placement, order modification, order closing, error handling, and reconciliation. 

---

## **1. Overview**  
The EA is responsible for handling all trade-related operations, including:  
✔️ Placing market and pending orders.  
✔️ Modifying and closing existing orders.  
✔️ Handling broker-specific trade limitations.  
✔️ Ensuring fast execution with minimal slippage.  
✔️ Maintaining a persistent state in Redis (RTS) and local state in DuckDB.  
✔️ Handling partial fills, rejections, requotes, and other edge cases.  
✔️ Ensuring consistent state between MT5 and the backend.  

---

## **2. Trade Types**  
The EA will support the following trade types:

| Trade Type | Description | Mode |
|------------|-------------|------|
| **Market Order** | Directly executes at the current price | Instant |
| **Limit Order** | Executes when the price hits a specified level | Pending |
| **Stop Order** | Executes when the price breaks through a specified level | Pending |
| **Trailing Stop** | Adjusts stop loss dynamically based on price movement | Real-time |
| **Partial Fill** | Allows order to fill partially if full fill is not possible | Instant |
| **Hedge Order** | Supports opening both long and short positions on the same symbol | Real-time |

---

## **3. Trade Execution Flow**  
The EA will follow the flow below for handling trades:

### ✅ **Step 1: Receive Trade Signal**  
- Trade signal is received from **RTSS** (Redis Trade Signal Stream).  
- The EA extracts the following trade details:  
    - Order type (Market/Limit/Stop)  
    - Symbol  
    - Lot size  
    - Stop Loss (SL)  
    - Take Profit (TP)  
    - Execution Type (Instant/Partial Fill)  
    - Order Expiry  

Example:
```json
{
  "order_type": "market",
  "symbol": "EURUSD",
  "lot_size": 0.1,
  "stop_loss": 30,
  "take_profit": 50,
  "execution_type": "instant"
}
```

---

### ✅ **Step 2: Validate Trade Parameters**  
The EA will validate the trade request to prevent execution failures:

| Parameter | Validation Criteria | Example |
|-----------|---------------------|---------|
| **Lot Size** | Within broker's minimum/maximum limits | Min: 0.01, Max: 100 |
| **Stop Loss** | SL > 0 and within broker range | 5 pips – 500 pips |
| **Take Profit** | TP > 0 and within broker range | 5 pips – 1000 pips |
| **Leverage** | Within allowed leverage | 1:100 – 1:500 |
| **Spread** | Less than `max_spread` defined in config | 2.5 pips |

---

### ✅ **Step 3: Execute Trade**  
1. If trade parameters are valid → Send order to MT5 via `OrderSend()` API.  
2. The EA will handle the following order execution modes:
   - **INSTANT** → Direct execution at market price.  
   - **PENDING** → Place limit/stop order with expiration time.  
   - **HEDGE** → Open both long and short orders simultaneously.  

Example:
```mql5
MqlTradeRequest request;
request.action = TRADE_ACTION_DEAL;
request.symbol = "EURUSD";
request.volume = 0.1;
request.type = ORDER_BUY;
request.price = SymbolInfoDouble(request.symbol, SYMBOL_BID);
request.stoploss = request.price - 0.0020;
request.takeprofit = request.price + 0.0030;
request.magic = 12345678;
OrderSend(request);
```

---

### ✅ **Step 4: Receive Execution Response**  
- Upon execution, the EA will receive a response from MT5:  
    - Success → `TRADE_RETCODE_PLACED`  
    - Partial Fill → `TRADE_RETCODE_DONE_PARTIAL`  
    - Failure → Error code returned  

| Code | Description | Action |
|------|-------------|--------|
| **10008** | TRADE_RETCODE_PLACED | Order successfully placed |
| **10010** | TRADE_RETCODE_DONE_PARTIAL | Partial fill → Adjust order |
| **10006** | TRADE_RETCODE_REJECT | Reject order → Notify backend |
| **10019** | TRADE_RETCODE_NO_MONEY | Insufficient balance → Log error |
| **10020** | TRADE_RETCODE_PRICE_CHANGED | Price change → Retry |
| **10031** | TRADE_RETCODE_CONNECTION | No connection → Retry with backoff |
| **10024** | TRADE_RETCODE_TOO_MANY_REQUESTS | Too many requests → Throttle |
| **10040** | TRADE_RETCODE_LIMIT_POSITIONS | Position limit reached → Reject order |

---

### ✅ **Step 5: Record State in Redis**  
- Upon successful execution → Update Redis `RTS` (Redis Trade State) and `RTRS` (Redis Trade Result Stream).  
- Redis updates will trigger Prometheus monitoring and backend state sync.  

Example:
```python
HSET trade_state:1324 status open
HSET trade_state:1324 profit 23.5
HSET trade_state:1324 entry_price 1.24567
```

---

### ✅ **Step 6: Handle Failures**  
1. If the order fails → Retry with exponential backoff (3 attempts).  
2. If order is partially filled → Attempt to fill the remaining volume.  
3. If the order remains unfilled → Move to `failed_syncs` table in DuckDB for recovery.  

Example:
- First failure → Retry after **1 second**  
- Second failure → Retry after **2 seconds**  
- Third failure → Retry after **4 seconds**  

---

### ✅ **Step 7: Modify Order (If Applicable)**  
- If trade state changes (e.g., trailing stop), modify order dynamically.  
- Adjust TP/SL using `OrderModify()` function.  

Example:
```mql5
OrderModify(ticket, new_price, new_stoploss, new_takeprofit, ORDER_TIME_GTC);
```

---

### ✅ **Step 8: Close Order**  
- Orders are closed under the following conditions:
    - Profit/loss target hit  
    - Stop-loss hit  
    - Manual close from UI  
- Use `OrderClose()` function.  

Example:
```mql5
OrderClose(ticket, volume, bid, slippage);
```

---

### ✅ **Step 9: Synchronize to Backend (Python)**  
- Successful trades → Sent to backend over TCP.  
- Failed trades → Stored in `failed_syncs` table in DuckDB.  
- State reconciliation handled by backend.  

---

## **4. Trade State Tracking**  
The EA will track the following trade state data in Redis:

| Key | Type | Description | TTL |
|------|------|-------------|-----|
| `trade_state:{trade_id}` | Hash | Active trade state | Until close |
| `trade_result:{trade_id}` | Stream | Trade execution result | Until processed |
| `instance_state:{instance_id}` | Hash | Instance-level status | Persistent |

---

## **5. Performance Optimization**  
1. Use **asynchronous execution** to reduce latency.  
2. Configure Redis to handle high-frequency signals.  
3. Use **in-memory execution** to minimize I/O bottlenecks.  
4. Process trade signals in parallel using Rust threading.  

---

## **6. Error Handling**  
| Error | Cause | Resolution |
|-------|-------|------------|
| **Trade Rejection** | Broker rejection | Log and retry |
| **Connection Error** | Network failure | Retry with backoff |
| **Partial Fill** | Market volume issue | Modify order |
| **Order Slippage** | Fast market movement | Retry at market price |
| **Invalid Volume** | Broker rules | Reject order |

---

## **7. Risk Management**  
1. Limit simultaneous trades per symbol.  
2. Set `max_order_size` to avoid oversizing.  
3. Cap open positions to broker limit.  

---

## ✅ **Summary**  
- Direct trade execution using `OrderSend()`  
- Async processing with backoff retries  
- Partial fill handling  
- Dynamic order modification  
- State reconciliation with Redis + backend  

---

### ✅ **Next:** Tick Data Handling 👇
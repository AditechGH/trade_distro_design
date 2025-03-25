### ✅ **EA Architectural Design Document: Tick Data Handling**  
**Purpose:**  
This document defines the detailed design for the **Tick Data Handling** aspect of the Expert Advisor (EA) for the Trade Distribution System. The goal is to ensure low-latency, accurate, and high-frequency transmission of tick data from MT5 to TradingView while maintaining consistent state and handling potential failures.  

---

## **1. Overview**  
Tick data forms the backbone of real-time trading analysis and execution. The EA will be responsible for:  
1. Reading tick data directly from the MT5 trading terminal.  
2. Streaming tick data to TradingView using both **UDP** and **TCP** for reliability and performance.  
3. Handling historical data backfill upon user request.  
4. Managing memory buffer overflow, network congestion, and failure scenarios.  

---

## **2. Data Stream Design**  
### ✅ **Hybrid Transmission Model:**  
The EA will use a hybrid communication model for tick transmission:  
- **UDP** – Used for real-time tick streaming due to low latency.  
- **TCP** – Used for backup transmission and historical data backfill due to higher reliability.  

### ✅ **Data Structure:**  
The EA will stream tick data in the following structure:  

| Field | Type | Description |
|-------|------|-------------|
| `timestamp` | int64 | UNIX epoch timestamp in milliseconds |
| `symbol` | string | Symbol name (e.g., "EURUSD") |
| `bid` | float | Bid price |
| `ask` | float | Ask price |
| `spread` | float | Spread in pips |
| `volume` | float | Volume of trade |
| `tick_direction` | int | Tick direction: 1 = up, -1 = down |
| `tick_type` | string | Type of tick: "new", "modified", "deleted" |

---

## **3. Tick Data Handling Flow**  
### ✅ **Real-Time Tick Stream:**  
1. EA reads tick data from the MT5 terminal using the `OnTick()` event.  
2. Tick data is immediately stored in the **Tick Buffer** (memory-mapped).  
3. EA sends tick data using **UDP** for low-latency transmission.  
4. EA sends the same tick data using **TCP** as a backup.  
5. If TCP value conflicts with UDP → Replace UDP data with TCP.  
6. Tick buffer size is dynamically adjusted based on available memory.  

---

### ✅ **Historical Backfill Flow:**  
1. On user request → EA queries historical data from MT5 using `CopyTicksRange()` function.  
2. Data is formatted into 1-minute OHLC bars.  
3. EA sends historical data via TCP.  
4. Historical data is cached in Redis (in case of repeated user requests).  
5. Historical backfill limited to **24 hours** of data to prevent memory overload.  

---

### ✅ **Tick Buffer Management:**  
1. Tick buffer size dynamically adjusted based on available memory.  
2. FIFO (First In, First Out) model for buffer overflow protection.  
3. Oldest ticks are dropped when buffer exceeds maximum size.  

---

### ✅ **Data Transmission Strategy:**  
| Protocol | Purpose | Frequency | Handling Strategy |
|----------|---------|-----------|------------------|
| **UDP** | Real-time streaming | Every tick | Sent immediately; discard on failure |
| **TCP** | Backup + historical backfill | Every tick (or on request) | Retransmit on failure |
| **WebSocket** | Confirmation of delivery | Every tick | Persistent connection for status update |

---

### ✅ **Tick Stream Handling by TradingView:**  
1. TradingView receives UDP stream first.  
2. TCP stream replaces UDP stream if discrepancy occurs.  
3. TradingView handles timeframe aggregation automatically.  
4. User-requested historical data is merged with real-time data.  

---

## **4. Buffer Design**  
### ✅ **Tick Buffer:**  
| Parameter | Value | Description |
|-----------|-------|-------------|
| **Buffer Type** | FIFO | Oldest ticks are overwritten when full |
| **Buffer Size** | 1000 ticks (adjustable) | Dynamic based on memory availability |
| **Tick Expiry** | 1 second | Expire ticks if not processed within 1 second |
| **Data Retention** | None | Tick data discarded once sent to TradingView |

### ✅ **Order State Buffer:**  
| Parameter | Value | Description |
|-----------|-------|-------------|
| **Buffer Type** | FIFO | Oldest state data overwritten when full |
| **Buffer Size** | 2000 orders | Track order state changes |
| **State Expiry** | 30 minutes | Expire state data after 30 minutes |

### ✅ **Historical Data Cache:**  
| Parameter | Value | Description |
|-----------|-------|-------------|
| **Buffer Type** | Redis | Cache for repeated requests |
| **Cache Expiry** | 24 hours | Historical data retained for 24 hours |
| **Data Format** | OHLC (1-minute) | Data compressed for fast delivery |

---

## **5. Concurrency Model**  
✅ **Separate Thread for Tick Handling:**  
- Dedicated thread for reading tick data.  
- Memory-mapped buffer for direct tick handling.  
- Low-latency UDP/TCP socket for real-time transmission.  

✅ **Separate Thread for Historical Backfill:**  
- Historical data fetched asynchronously from MT5.  
- Data formatted and sent via TCP.  
- Cache stored in Redis for repeated requests.  

✅ **Concurrent Processing:**  
- Multiple tick streams processed concurrently using Rust-based threading.  
- Tick buffer size adjusted dynamically based on concurrent load.  

---

## **6. Handling Network Congestion**  
✅ **UDP:**  
- If UDP transmission fails → Ignore tick (no retransmit).  
- If network congestion is detected → Reduce tick frequency by 50%.  

✅ **TCP:**  
- If TCP transmission fails → Retransmit using exponential backoff.  
- If TCP backlog exceeds 10 ticks → Drop oldest ticks.  

✅ **WebSocket:**  
- Persistent connection with TradingView for error handling.  
- Notify TradingView of UDP failure → Switch to TCP-only mode if needed.  

---

## **7. Handling Overflow & Memory Pressure**  
| Scenario | Impact | Handling Strategy |
|----------|--------|------------------|
| **Tick Buffer Overflow** | Oldest ticks lost | Reduce tick frequency, increase buffer size |
| **Memory Pressure** | High CPU/Memory usage | Reduce tick frequency |
| **High Tick Rate (> 1000/sec)** | CPU overload | Throttle tick rate, drop oldest ticks |
| **Redis Overflow** | Historical data loss | Increase Redis memory allocation |

---

## **8. Edge Cases**  
| Case | Impact | Handling Strategy |
|-------|--------|------------------|
| **MT5 Stops Sending Ticks** | No data | Reconnect after 5 seconds |
| **UDP/TCP Transmission Failure** | Data loss | Retransmit with exponential backoff |
| **TradingView Drops Connection** | Data loss | Reconnect after 2 seconds |
| **Tick Data Corruption** | Mismatched values | Drop and resend |
| **Data Desync Between UDP and TCP** | Inconsistent state | TCP replaces UDP data |
| **High Network Latency** | Increased tick delivery time | Reduce tick frequency by 50% |
| **Redis Cache Overflow** | Data loss | Purge oldest data |

---

## **9. Performance Tuning**  
✅ **Tick buffer size** → Dynamic adjustment based on available memory  
✅ **Tick expiry** → Short expiry to reduce memory footprint  
✅ **Redis eviction policy** → `allkeys-lru` for tick cache  
✅ **Concurrency model** → Rust-based threading for low-latency processing  
✅ **UDP vs TCP** → UDP prioritized; TCP used as fallback  

---

## **10. Permissions and Security**  
✅ UDP secured with **DTLS**  
✅ TCP secured with **TLS 1.2**  
✅ Redis secured with **password-based authentication**  
✅ TradingView endpoint whitelisted at network level  
✅ MT5 permissions restricted to read-only mode for tick data  

---

## **11. High Availability**  
✅ EA restarts automatically on failure.  
✅ Redis-backed state for reconnection consistency.  
✅ Persistent TCP connection ensures fallback upon UDP loss.  

---

## **12. Fault Tolerance**  
✅ Tick buffer overflow handled via FIFO model.  
✅ UDP drop → No action.  
✅ TCP drop → Retransmit using exponential backoff.  
✅ Redis cache overflow → Eviction using `allkeys-lru`.  

---

## 🚀 **✅ We now have a complete, high-performance, and fault-tolerant Tick Data Handling Design!**  
👉 Next: Should we focus on trade execution design or permissions?
That’s a very good question — we need to account for **both login attempts** and **account closure** in the MT5 context because they directly impact instance state and trade execution flow. Let’s break it down:

---

## ✅ **1. Handling Login Attempts**
Login attempts are critical because MT5 instances are tied to broker accounts, which require secure authentication to execute trades. 

### 🚀 **Login Flow:**
1. **Trigger:**  
   - When an MT5 instance starts, it will attempt to log in to the broker using the stored credentials.  
   - The login credentials are stored securely in **PostgreSQL** (PG) and fetched during instance startup.  

2. **State Tracking:**  
   - State will be tracked in Redis (Instance State Layer).  
   - Example Redis keys:  
     - `instance_state:<instance_id>:status` → `CONNECTING`  
     - `instance_state:<instance_id>:login_attempt_count` → **Tracks login attempts**  
     - `instance_state:<instance_id>:connection_state` → `DISCONNECTED`, `CONNECTED`, `FAILED`  

3. **Limit Login Attempts:**  
   - **Allow up to 5 login attempts** in a row.  
   - After 5 failed attempts → Mark instance as `FAILED`.  
   - Send a notification to the backend → Save failure details in DuckDB for investigation.  

4. **Backoff Strategy:**  
   - Apply **exponential backoff** after each failed login attempt.  
     - 1st Attempt → Retry after **5 seconds**  
     - 2nd Attempt → Retry after **10 seconds**  
     - 3rd Attempt → Retry after **30 seconds**  
     - 4th Attempt → Retry after **1 minute**  
     - 5th Attempt → Retry after **2 minutes**  

5. **Authentication Failure Handling:**  
   If the final login attempt fails:  
   - **Log the failure** to Redis and DuckDB.  
   - **Trigger alert** via Prometheus.  
   - **Stop the instance** and mark it as `FAILED`.  
   - **Auto-recovery** will not be triggered after repeated login failures.  

6. **Manual Reset:**  
   - User can manually reset the state via the UI or API.  
   - This will reset the login attempt counter and attempt to re-establish a connection.  

---

### 🔥 **Prometheus Alerts for Login Handling**  
| **Alert Type** | **Trigger** | **Action** |
|---------------|-------------|------------|
| **Login Failure** | Failed to authenticate after 5 attempts | Stop instance + notify |
| **Repeated Login Failure** | 3 consecutive failed login attempts | Notify + throttle connection |
| **Successful Login After Failure** | Successful login after failure | Clear login failure state |
| **Account Banned** | Broker account marked as suspended/banned | Disable instance + notify |

---

### ✅ **New Redis Keys for Login Handling**
| **Redis Key** | **Value** | **Purpose** |
|---------------|-----------|-------------|
| `instance_state:<instance_id>:status` | `CONNECTING`, `CONNECTED`, `FAILED` | Current login state |
| `instance_state:<instance_id>:login_attempt_count` | Integer | Tracks login attempts |
| `instance_state:<instance_id>:last_login_attempt` | Timestamp | Last login attempt time |
| `instance_state:<instance_id>:login_failure_reason` | String | Reason for login failure |

---

## ✅ **2. Handling Account Closure**
Account closure is more delicate because it impacts both trade execution and state retention.  

### 🚀 **Closure Flow:**
1. **Trigger:**  
   - If a broker account is closed → The broker will **terminate all active sessions**.  
   - MT5 will detect that the connection is lost.  
   - This will trigger the instance to attempt reconnection → But it will fail because the account no longer exists.  

2. **State Tracking:**  
   - MT5 will detect a `DISCONNECTED` state → Mark the state as `FAILED`.  
   - Store in Redis under:  
     - `instance_state:<instance_id>:status` → `DISCONNECTED`  
     - `instance_state:<instance_id>:login_failure_reason` → `ACCOUNT_CLOSED`  

3. **Auto-Recovery Handling:**  
   - If the failure reason is `ACCOUNT_CLOSED` →  
     - DO NOT attempt auto-recovery.  
     - Mark the instance as **permanently closed**.  

4. **Remove State and Credentials:**  
   - Remove the credentials from PostgreSQL (after confirmation).  
   - Remove the instance state from Redis.  
   - Mark the instance as **INACTIVE** in DuckDB.  

5. **Notify User:**  
   - Send an email or notification via UI to inform the user that the account is closed.  
   - Offer the option to add a new account.  

---

### 🔥 **Prometheus Alerts for Account Closure**
| **Alert Type** | **Trigger** | **Action** |
|---------------|-------------|------------|
| **Account Closed** | Broker connection lost + `ACCOUNT_CLOSED` state detected | Notify + disable instance |
| **Auto-Recovery Failure** | Account closure prevents login recovery | Notify + stop instance |
| **Credentials Deleted** | Account deletion confirmed | Clear instance state |

---

### ✅ **New Redis Keys for Account Closure Handling**
| **Redis Key** | **Value** | **Purpose** |
|---------------|-----------|-------------|
| `instance_state:<instance_id>:status` | `INACTIVE`, `FAILED` | Current state after closure |
| `instance_state:<instance_id>:last_failure_reason` | `ACCOUNT_CLOSED` | Store closure reason |
| `instance_state:<instance_id>:connection_state` | `DISCONNECTED` | Instance connection state |
| `instance_state:<instance_id>:login_attempt_count` | Integer | Set to 0 after closure |

---

## ✅ **3. Credentials Cleanup**  
After confirming account closure:  
✅ Remove login credentials from PostgreSQL.  
✅ Remove active sessions from Redis.  
✅ Archive account state to DuckDB (for historical tracking).  
✅ Clear Redis keys for the instance.  

---

## 🚨 **Security Risks & Mitigations**  
| **Risk** | **Mitigation** |
|----------|---------------|
| **Brute Force Login Attempts** | Limit login attempts to 5 with exponential backoff |
| **Session Hijacking** | OAuth tokens tied to device fingerprint |
| **Race Condition During Recovery** | Lock Redis key for `login_attempt` during recovery |
| **Failed Auto-Recovery** | Attempt only after valid session confirmation |

---

## ✅ **4. Auto-Recovery Handling After Login Failure**  
| **State** | **Action** | **Recovery Attempt** |
|-----------|------------|---------------------|
| `CONNECTING` | Attempt login | 5 attempts with backoff |
| `FAILED` | Attempt restart | 3 attempts |
| `INACTIVE` | Stop instance | No recovery |
| `ACCOUNT_CLOSED` | Stop instance | No recovery |

---

## 🚀 **5. Edge Cases for Login and Closure**
### ✅ **Edge Case 1:** Network Loss During Login  
- MT5 loses network connection mid-login → Wait 10 seconds → Retry login if no reconnect.  

### ✅ **Edge Case 2:** Partial Fill During Account Closure  
- If trades are still open → Attempt to close them gracefully.  
- If broker rejects them → Mark trades as `FAILED`.  
- If order book is locked → Attempt to cancel orders.  

### ✅ **Edge Case 3:** Auto-Recovery Failure  
- If auto-recovery fails → Mark instance as `FAILED`.  
- Notify user via UI and logs.  

---

## 🏆 **Next Step:**  
✅ Finalize Redis key structure for login and state tracking.  
✅ Finalize Prometheus scraping logic.  
✅ Finalize instance failure recovery logic.  
✅ Proceed to **MT5 Edge Case Handling**.  

### ✅ **MT5 Permission and Security Handling**  
*(Architectural Design for Permission and Security Control in MT5)*  

---

## **1. Overview**  
Permissions and security handling for MT5 instances ensures that the system:  
- Restricts unauthorized access to MT5 instances.  
- Validates broker-specific permissions and trading rights.  
- Controls access to trade execution based on user roles and instance-level settings.  
- Implements secure storage and transmission of MT5 credentials.  
- Handles secure login and session management.  
- Prevents unauthorized execution of trades or configuration changes.  

---

## **2. Permission Model Overview**  
The permission model will operate at three levels:  
| Level | Description | Scope |
|-------|-------------|-------|
| **Instance-Level Permissions** | Control trade execution rights and settings at the instance level. | Single instance |
| **User-Level Permissions** | Control user access to MT5 instance configuration and execution. | Single user |
| **System-Level Permissions** | Control overall system access and security settings. | Entire system |  

---

## **3. MT5 Permission Handling**  
### **3.1. Instance-Level Permissions**
Permissions specific to individual instances. These are set and managed by the user during instance creation and configuration.  

| Permission | Description | Impact |  
|------------|-------------|--------|  
| **Trade Execution** | Allow/disallow trade execution on the instance. | Blocks order placement at the broker level. |  
| **Order Modification** | Allow/disallow order modification after execution. | Disables trade modification at the instance level. |  
| **Order Closing** | Allow/disallow order closing. | Prevents closing of open orders manually. |  
| **Max Order Size** | Restrict the maximum order size allowed. | Prevents oversized trades from being placed. |  
| **Stop-Loss/Take-Profit Modification** | Allow/disallow modification of stop-loss and take-profit orders. | Protects trades from forced closure. |  
| **Hedging** | Allow/disallow hedging on the instance. | Prevents opening opposite trades. |  
| **Trailing Stop** | Allow/disallow use of trailing stop-loss orders. | Prevents automatic trade adjustments. |  
| **Auto-Close on Margin Call** | Allow/disallow automatic closure of trades upon margin call. | Protects capital during volatility. |  

---

### **3.2. User-Level Permissions**  
Permissions specific to the user logged into the system. These are set and managed at the backend using PostgreSQL.  

| Permission | Description | Scope |  
|------------|-------------|--------|  
| **Trade Execution** | Allow/disallow execution of trades by the user. | User-level |  
| **Modify Configuration** | Allow/disallow modification of instance configuration. | User-level |  
| **Close Trades** | Allow/disallow manual closing of trades. | User-level |  
| **Access Historical Data** | Allow/disallow user to view historical trade data. | User-level |  
| **Create New Instance** | Allow/disallow creation of new MT5 instances. | User-level |  
| **Terminate Instance** | Allow/disallow termination of running MT5 instances. | User-level |  

---

### **3.3. System-Level Permissions**  
Permissions at the system level, defined in the backend (Python) and managed in PostgreSQL.  

| Permission | Description | Scope |  
|------------|-------------|--------|  
| **Create User** | Allow/disallow creation of new user accounts. | System-wide |  
| **Modify User Role** | Allow/disallow changes to user roles. | System-wide |  
| **Modify System Settings** | Allow/disallow modification of global system settings. | System-wide |  
| **View Logs** | Allow/disallow viewing of system-level logs. | System-wide |  
| **Instance Allocation** | Allow/disallow manual allocation of system resources to MT5 instances. | System-wide |  

---

## **4. MT5 Security Handling**  
Security handling covers the following key areas:  

### **4.1. Secure Storage of MT5 Credentials**
- MT5 login details (login, password, server) are stored securely in **PostgreSQL**.  
- **AES-256 encryption** will be used for password storage.  
- Encryption keys will be stored securely in a dedicated key management service (KMS).  
- Key rotation every **90 days** for enhanced security.  

---

### **4.2. Session Management**  
- Upon successful MT5 login → Session ID is generated and stored in Redis (RIS).  
- Session expiration → After **30 minutes** of inactivity.  
- Refresh token rotation → Every **14 minutes**.  
- Backend Python module will handle:  
  - Session validation  
  - Session renewal  
  - Logout  

---

### **4.3. Fingerprint Binding**  
- On login, a device fingerprint is generated using:  
  - Device ID  
  - Hardware ID  
  - Operating System  
  - Browser Type (if applicable)  
- Fingerprint is stored in Redis.  
- All subsequent session renewals will be validated against the saved fingerprint.  
- If fingerprint mismatch occurs:  
  - Block login attempt  
  - Trigger MFA verification  

---

### **4.4. Secure Transmission**
- All data between the frontend (Rust) and backend (Python) is transmitted over **TLS 1.3**.  
- MT5 server connection → Encrypted using **TLS 1.3**.  
- ClickHouse and PostgreSQL connections → Encrypted using TLS.  

---

### **4.5. Rate Limiting**  
- **Login attempts** → Maximum of **3 attempts** within a 5-minute window.  
- **Trade execution attempts** → Maximum of **10 attempts** per minute.  
- **IP Banning** → After 3 consecutive failed login attempts → Temporary ban for **15 minutes**.  

---

### **4.6. Multi-Factor Authentication (MFA)**
- MFA will be enforced if:  
  - Device fingerprint mismatch  
  - IP address change  
  - Suspicious login attempts  
- MFA Types Supported:  
  - Google Authenticator  
  - Email-based verification  
  - SMS-based verification  

---

### **4.7. Broker-Specific Permissions**
- Certain permissions are broker-specific and are fetched directly from the broker’s MT5 API:  
   - **Minimum order size**  
   - **Maximum order size**  
   - **Stop-loss/take-profit settings**  
   - **Margin requirements**  
   - **Market availability**  

---

## **5. Security Edge Cases and Handling**  
| Edge Case | Description | Resolution |  
|-----------|-------------|------------|  
| **Network Loss** | Connection to MT5 server is lost during trade execution. | Auto-reconnect and resume operation. |  
| **Invalid Login** | Incorrect MT5 login details. | Return error and notify user. |  
| **Expired Session** | Session expires due to inactivity. | Require re-login or refresh token. |  
| **Fingerprint Mismatch** | Attempted login from different device. | Require MFA confirmation. |  
| **Broker Disconnection** | Broker disconnects from MT5 instance. | Attempt automatic reconnect after 5 seconds. |  
| **Insufficient Balance** | Not enough funds to open trade. | Return error and notify user. |  
| **Invalid Stop-Loss/Take-Profit** | SL/TP value violates broker’s limits. | Return error and notify user. |  
| **Trade Execution Error** | Order rejected by broker due to market state. | Return error and notify user. |  

---

## **6. Architectural Design Flow**  

### **✅ Happy Path**  
1. User logs into MT5.  
2. Session is established and bound to device fingerprint.  
3. Trade operation request → Verified by instance and user permissions.  
4. Trade request is sent to broker → Successful execution.  

---

### **❌ Edge Case 1: Invalid Credentials**  
1. User logs into MT5 with incorrect login details.  
2. Broker returns `TRADE_RETCODE_REJECT`.  
3. Return error to frontend.  
4. Increment login attempt counter.  

---

### **❌ Edge Case 2: Session Expiration**  
1. User is inactive for 30 minutes.  
2. Session expires.  
3. Require re-login or refresh token.  

---

### **❌ Edge Case 3: Fingerprint Mismatch**  
1. Device fingerprint mismatch detected.  
2. Reject login attempt.  
3. Trigger MFA.  

---

## **7. Next Step**  
1. Finalize MT5 permission and security schema in Redis, DuckDB, and PostgreSQL.  
2. Define JSON config file format for storing instance-level permissions.  
3. Handle error handling and broker-specific permission discrepancies.  

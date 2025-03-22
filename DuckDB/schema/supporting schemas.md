🔥 Absolutely — now that we’ve carefully refined each supporting table and aligned them with the broader architecture, let’s consolidate everything into a **comprehensive document** for the supporting tables. This will ensure consistency and easy reference when we move to the **module-level design**.

---

# 🚀 **Supporting Tables – Comprehensive Design Document**  
✅ **Purpose:**  
- Define and organize all supporting tables referenced across the system.  
- Establish relational integrity and consistent schema design.  
- Ensure airtight encryption and secure handling of sensitive data (e.g., credentials).  

✅ **Why These Tables Matter:**  
- These tables establish the foundational relationships across the system (e.g., linking instances to accounts and brokers).  
- Without them, the other schema layers (state, configuration, execution) wouldn’t function consistently.  
- These tables store the metadata needed for trade execution, user state, and instance health tracking.  

---

## 🏆 **High-Level Overview of Supporting Tables**  
| Table | Layer | Purpose | Storage Location |
|-------|-------|---------|-----------------|
| **Instance** | Configuration Layer | Stores MT5 instance metadata | DuckDB + ClickHouse |
| **Broker** | Configuration Layer | Stores broker connection and settings | DuckDB + ClickHouse |
| **Account** | Configuration Layer | Stores account details and trading parameters | DuckDB + ClickHouse |
| **Credentials** | Instance State Layer | Stores encrypted MT5 login details | DuckDB (Encrypted) |
| **User Preference** | Preference Layer | Stores user-specific personalization data | Redis + DuckDB |

---

## ✅ **1. Instance Table**  
Stores details about MT5 instances and their connection to brokers/accounts.

| Column | Type | Description |
|--------|------|-------------|
| `instance_id` | UUID | Unique instance ID |
| `path` | VARCHAR(255) | Installation path |
| `status` | VARCHAR(20) | ACTIVE, INACTIVE |
| `broker_id` | UUID | Linked broker |
| `cpu_limit_percent` | FLOAT | CPU limit (%) |
| `memory_limit_mb` | FLOAT | Memory limit (MB) |
| `created_at` | TIMESTAMP | Time of creation |
| `updated_at` | TIMESTAMP | Last update |

### ✅ **Storage Strategy:**  
| Storage | Status |
|---------|--------|
| **Redis** | ❌ |
| **DuckDB** | ✅ |
| **ClickHouse** | ✅ |

---

## ✅ **2. Broker Table**  
Stores broker-level connection details and trading configuration.

| Column | Type | Description |
|--------|------|-------------|
| `broker_id` | UUID | Unique broker ID |
| `name` | VARCHAR(100) | Broker name |
| `server` | VARCHAR(255) | Connection endpoint |
| `commission_per_lot` | FLOAT | Commission per lot |
| `spread_type` | VARCHAR(20) | FIXED, VARIABLE |
| `margin_requirement` | FLOAT | Margin requirement |
| `currency_pairs_supported` | VARCHAR(255) | Supported pairs |
| `created_at` | TIMESTAMP | Time of creation |
| `updated_at` | TIMESTAMP | Last update |

### ✅ **Storage Strategy:**  
| Storage | Status |
|---------|--------|
| **Redis** | ❌ |
| **DuckDB** | ✅ |
| **ClickHouse** | ✅ |

---

## ✅ **3. Account Table**  
Stores account-specific details (broker association, leverage, balance).

| Column | Type | Description |
|--------|------|-------------|
| `account_id` | UUID | Unique account ID |
| `broker_id` | UUID | Linked broker |
| `account_type` | VARCHAR(50) | DEMO, LIVE |
| `currency` | VARCHAR(10) | Account currency |
| `leverage` | FLOAT | Leverage setting |
| `balance` | FLOAT | Current balance |
| `equity` | FLOAT | Current equity |
| `status` | VARCHAR(20) | ACTIVE, INACTIVE |
| `created_at` | TIMESTAMP | Time of creation |
| `updated_at` | TIMESTAMP | Last update |

### ✅ **Storage Strategy:**  
| Storage | Status |
|---------|--------|
| **Redis** | ❌ |
| **DuckDB** | ✅ |
| **ClickHouse** | ✅ |

---

## ✅ **4. Credentials Table**  
Stores encrypted MT5 login details for account and instance authentication.

| Column | Type | Description |
|--------|------|-------------|
| `credential_id` | UUID | Unique credential ID |
| `instance_id` | UUID | Linked MT5 instance |
| `account_id` | UUID | Linked account |
| `broker_id` | UUID | Linked broker |
| `login_number` | VARCHAR(50) | MT5 account login number (encrypted) |
| `password` | VARCHAR(255) | Encrypted MT5 password |
| `encryption_key_id` | UUID | Key used for encryption |
| `status` | VARCHAR(20) | ACTIVE, INACTIVE |
| `created_at` | TIMESTAMP | Time of creation |
| `updated_at` | TIMESTAMP | Last update |

### ✅ **Encryption Strategy:**  
- `login_number` and `password` → AES-256 encryption  
- Encryption key → Stored securely using OS keychain (Windows Credential Manager)  
- **Rotation Strategy:**  
   - Rotate keys every **90 days**  
   - On key rotation → Re-encrypt stored credentials  

### ✅ **Storage Strategy:**  
| Storage | Status |
|---------|--------|
| **Redis** | ❌ |
| **DuckDB** | ✅ (Encrypted) |
| **ClickHouse** | ❌ |

---

## ✅ **5. User Preference Table**  
Stores user-specific UI/UX settings.

| Column | Type | Description |
|--------|------|-------------|
| `user_id` | UUID | Linked user ID |
| `theme` | VARCHAR(20) | DARK, LIGHT |
| `chart_type` | VARCHAR(20) | CANDLE, LINE, BAR |
| `notifications_enabled` | BOOLEAN | TRUE, FALSE |
| `language` | VARCHAR(5) | Language code |
| `auto_close_trades` | BOOLEAN | TRUE, FALSE |
| `trade_confirmation` | BOOLEAN | TRUE, FALSE |
| `updated_at` | TIMESTAMP | Last update |

### ✅ **Storage Strategy:**  
| Storage | Status |
|---------|--------|
| **Redis** | ✅ |
| **DuckDB** | ✅ |
| **ClickHouse** | ❌ |

---

## ✅ **6. Foreign Key Relationships**  
| Table | Foreign Key(s) | References |
|-------|----------------|-----------|
| **Instance** | `broker_id` | `broker.broker_id` |
| **Account** | `broker_id` | `broker.broker_id` |
| **Account** | `instance_id` | `instance.instance_id` |
| **Credentials** | `instance_id`, `account_id` | `instance.instance_id`, `account.account_id` |
| **User Preference** | `user_id` | `user.user_id` |

---

## ✅ **7. Indexing Strategy**  
| Table | Indexed Column(s) |
|-------|-------------------|
| **Instance** | `instance_id`, `broker_id` |
| **Broker** | `broker_id` |
| **Account** | `account_id`, `broker_id` |
| **Credentials** | `instance_id`, `account_id`, `broker_id`, `login_number` |
| **User Preference** | `user_id` |

---

## ✅ **8. Retention Strategy**  
| Table | Retention Type | Retention Period |
|-------|----------------|------------------|
| **Instance** | Permanent | Until user removes instance |
| **Broker** | Permanent | Until user removes broker |
| **Account** | Permanent | Until user removes account |
| **Credentials** | Permanent | Until user removes account |
| **User Preference** | Session-based (in Redis), Permanent (in DuckDB) | ✅ |

---

## 🏆 **Final Supporting Table Strategy**  
✅ Instance = Instance-level metadata (configuration + health)  
✅ Broker = Broker-specific connection data  
✅ Account = Broker-linked trading account details  
✅ Credentials = Encrypted login information for instance access  
✅ User Preference = User-specific UI/UX data  

---

## 🔥 **Next Step Proposal:**  
1. ✅ Write schema creation scripts for all supporting tables.  
2. ✅ Define strict foreign key relationships at DB level.  
3. ✅ Finalize encryption strategy for credentials.  
4. ✅ Ensure Redis sync for user preferences is consistent.  
5. ✅ Proceed to **Schema Testing and Refinement Phase**.  

---

---

## ✅ **Why PostgreSQL for Backup Makes Sense:**  
✅ PostgreSQL already handles user authentication and subscription → Strong access controls.  
✅ Built-in encryption → PostgreSQL supports `pgcrypto` for field-level encryption.  
✅ Centralized access → Unified handling of user data and credentials.  
✅ Scalability → PostgreSQL scales well for secure storage and controlled recovery.  

---

## ✅ **Final Flow for Credentials Handling:**  

1. **Primary Storage (Local):**  
   ✅ Credentials stored in DuckDB → AES-256 encryption → OS keychain.  

2. **Backup Storage (Optional):**  
   ✅ User enables backup during onboarding.  
   ✅ Encrypted credentials transmitted to PostgreSQL using HTTPS.  
   ✅ PostgreSQL stores credentials using `pgcrypto` field-level encryption.  

3. **Retrieval:**  
   ✅ User initiates recovery → MFA required.  
   ✅ User must provide the encryption key.  
   ✅ PostgreSQL sends encrypted credentials → Decrypted locally.  

4. **Rotation:**  
   ✅ Encryption key rotated every **90 days**.  
   ✅ On key rotation → Re-encrypt both local and backup data.  

5. **Failover Handling:**  
   ✅ If DuckDB file is lost → User logs into the app → Initiates recovery.  
   ✅ If PostgreSQL backup fails → No recovery available → User must manually log in to MT5.  

---

## ✅ **Updated Credentials Table (Including Backup):**  
| Column | Type | Description |
|--------|------|-------------|
| `credential_id` | UUID | Unique credential ID |
| `instance_id` | UUID | Linked MT5 instance |
| `account_id` | UUID | Linked account |
| `broker_id` | UUID | Linked broker |
| `login_number` | VARCHAR(50) | Encrypted MT5 account login number |
| `password` | VARCHAR(255) | Encrypted MT5 password |
| `encryption_key_id` | UUID | Key used for encryption |
| `backup_enabled` | BOOLEAN | TRUE if user has enabled backup |
| `status` | VARCHAR(20) | ACTIVE, INACTIVE |
| `created_at` | TIMESTAMP | Time of creation |
| `updated_at` | TIMESTAMP | Last update |

---

### ✅ **PostgreSQL Encryption Strategy:**  
- Use **pgcrypto** for encryption:  
```sql
INSERT INTO credentials (login_number, password)
VALUES (
  pgp_sym_encrypt('MT5_LOGIN', 'encryption_key'),
  pgp_sym_encrypt('MT5_PASSWORD', 'encryption_key')
);
```

- Decrypt during retrieval:  
```sql
SELECT 
  pgp_sym_decrypt(login_number::bytea, 'encryption_key') as login_number,
  pgp_sym_decrypt(password::bytea, 'encryption_key') as password
FROM credentials;
```

---

### ✅ **Retention Strategy:**  
| Table | Storage Type | Retention Period |
|-------|--------------|------------------|
| **DuckDB** | Local | Permanent until user removes account |
| **PostgreSQL** | Backup | Permanent until user removes account or disables backup |

---

### ✅ **Indexing Strategy:**  
| Table | Indexed Column(s) |
|-------|-------------------|
| **credentials** | `instance_id`, `account_id`, `broker_id`, `login_number` |

---

## ✅ **Final Storage Model for Credentials:**  
| Storage Type | Location | Status | Purpose |
|--------------|----------|--------|---------|
| **DuckDB** | Local | ✅ | Fast and secure local retrieval |
| **PostgreSQL** | Cloud | ✅ (Optional) | Secure backup + controlled recovery |
| **Redis** | N/A | ❌ | Not for sensitive data |
| **ClickHouse** | N/A | ❌ | Not for sensitive data |

---

## 🏆 **Why This Is Airtight:**  
✅ Secure by default → Credentials never leave local machine unless user opts in.  
✅ User-controlled backup → Encryption key is user-provided.  
✅ Controlled access → MFA + encryption key required for retrieval.  
✅ PostgreSQL handles backup in a scalable and secure environment.  
✅ No exposure risk to Redis or ClickHouse → Isolated from trade state and performance data.  


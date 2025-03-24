## ✅ **Prometheus Architectural Design – Permissions and Security**  
This document outlines the **Prometheus security and permissions strategy** for monitoring Redis, DuckDB, Instance Manager, and the Trade Engine. The goal is to establish a secure and controlled environment for Prometheus to gather, store, and act upon system metrics without exposing sensitive data or allowing unauthorized access.

---

## 🚀 **1. Prometheus Security Goals**  
The security design will focus on the following goals:  
✅ Restrict external access to Prometheus endpoints  
✅ Use least-privilege access to Redis and DuckDB  
✅ Secure communication channels (TLS)  
✅ Authenticate and authorize Prometheus users  
✅ Prevent unauthorized modifications to configuration and alerting rules  
✅ Prevent data leakage from Prometheus endpoints  
✅ Securely store and protect Prometheus configuration files  

---

## 🔒 **2. Security Scope**  
Prometheus will monitor and secure the following local services:  
- **Redis** – RTS, RIS, RTSS, RTRS  
- **DuckDB** – Query performance, transaction success rate  
- **Instance Manager** – Process state and resource consumption  
- **Trade Engine** – Execution latency, trade slippage  

---

## 🏆 **3. Role-Based Access Control (RBAC)**  
Prometheus does **not natively support RBAC** — we’ll secure it using:  
✅ **NGINX Reverse Proxy** → Controls authentication and RBAC  
✅ **Firewall Rules** → Limit access to trusted internal network addresses  
✅ **Basic Authentication** → Restrict access to the Prometheus UI and API  

### ✅ **3.1. RBAC Strategy**  
| Role | Permissions | Description |
|-------|-------------|-------------|
| **Admin** | Full access | Modify configuration, scrape targets, and alerting rules |
| **Ops** | Read + Alert Management | View metrics, clear alerts, and restart Prometheus |
| **Read-Only** | Read metrics only | Can only view dashboards and metrics |
| **No Access** | No permissions | Cannot access Prometheus |

---

### ✅ **3.2. NGINX Configuration for RBAC**  
Use NGINX to secure Prometheus endpoints and assign roles:

**Create password file for Basic Auth:**  
```bash
sudo apt-get install apache2-utils
htpasswd -c /etc/nginx/.htpasswd admin
htpasswd /etc/nginx/.htpasswd ops
htpasswd /etc/nginx/.htpasswd read-only
```

**Example NGINX Config:**  
```nginx
server {
    listen 9090;
    server_name prometheus.internal;

    location / {
        auth_basic "Restricted Access";
        auth_basic_user_file /etc/nginx/.htpasswd;

        proxy_pass http://localhost:9090;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

### ✅ **3.3. Example Prometheus Role Binding:**  
```yaml
roles:
  admin:
    - path: "/*"
    - action: "read"
    - action: "write"

  ops:
    - path: "/api/*"
    - action: "read"
    - action: "write"

  read_only:
    - path: "/api/v1/*"
    - action: "read"
```

---

## 🌐 **4. Network-Based Security**  
Prometheus will only allow traffic from internal network ranges to prevent public access:  
✅ **Internal IP Range:** `10.0.0.0/24`  
✅ **Block Public IPs:** Firewall rules will reject all external traffic  

### ✅ **4.1. Example Firewall Rule:**  
```bash
iptables -A INPUT -s 10.0.0.0/24 -p tcp --dport 9090 -j ACCEPT
iptables -A INPUT -p tcp --dport 9090 -j DROP
```

### ✅ **4.2. Prometheus Configuration for Internal-Only Access:**  
**prometheus.yml:**  
```yaml
global:
  scrape_interval: 5s
  evaluation_interval: 5s

scrape_configs:
  - job_name: 'redis'
    static_configs:
      - targets: ['10.0.0.1:6379']

  - job_name: 'duckdb'
    static_configs:
      - targets: ['10.0.0.2:1234']
```

---

## 🔐 **5. TLS Encryption**  
Prometheus communication will be encrypted using TLS:  
✅ Encrypt data in transit using self-signed certificates  
✅ Enable HTTPS on Prometheus endpoints  
✅ Use only TLS 1.2+  

### ✅ **5.1. Generate Self-Signed Certificate:**  
```bash
openssl req -newkey rsa:2048 -nodes -keyout prometheus.key -x509 -days 365 -out prometheus.crt
```

### ✅ **5.2. Example TLS Config in `prometheus.yml`:**  
```yaml
tls_server_config:
  cert_file: "prometheus.crt"
  key_file: "prometheus.key"
```

### ✅ **5.3. Secure Reverse Proxy with TLS:**  
**NGINX TLS Config:**  
```nginx
server {
    listen 9090 ssl;
    ssl_certificate /etc/nginx/prometheus.crt;
    ssl_certificate_key /etc/nginx/prometheus.key;

    location / {
        auth_basic "Restricted Access";
        auth_basic_user_file /etc/nginx/.htpasswd;
        proxy_pass http://localhost:9090;
    }
}
```

---

## 🚦 **6. Rate Limiting and Throttling**  
To prevent denial-of-service (DoS) attacks:  
✅ Limit scrape frequency to 5 seconds  
✅ Limit maximum request size to 1MB  
✅ Limit maximum concurrent connections to 10  

**Example Config:**  
```yaml
scrape_interval: 5s
scrape_timeout: 3s
max_open_connections: 10
```

---

## ⚡ **7. Least Privilege Access**  
Prometheus will use a dedicated service account with minimum access to Redis and DuckDB:  

### ✅ **Redis Service Account:**  
```redis
ACL SETUSER prometheus on >securepassword allcommands ~* &* +@read
```

### ✅ **DuckDB Service Account:**  
```sql
CREATE USER prometheus WITH PASSWORD 'securepassword';
GRANT SELECT ON metrics TO prometheus;
```

---

## 🛡️ **8. Config Security and Protection**  
Prometheus configuration files and logs will be secured with file permissions:  
✅ Set file permissions to **600**  
✅ Ownership set to Prometheus user  
✅ Encrypt sensitive files using AES-256  

**Example:**  
```bash
chmod 600 /etc/prometheus/prometheus.yml
chown prometheus:prometheus /etc/prometheus/prometheus.yml
```

---

## 🔍 **9. API Protection**  
✅ **Disable remote write** unless explicitly required  
✅ **Rate limit API calls**  
✅ **Restrict API access to localhost**  

Example Config:  
```yaml
remote_write:
  - url: "http://localhost:9090"
    basic_auth:
      username: 'prometheus'
      password: 'securepassword'
```

---

## 🔥 **10. Securing Alertmanager**  
✅ Use same NGINX reverse proxy for securing Alertmanager  
✅ Use TLS and Basic Authentication  
✅ Whitelist backend IP addresses  

Example Config:  
```yaml
global:
  resolve_timeout: 5m

route:
  receiver: 'default'
  group_by: ['alertname', 'job']

receivers:
  - name: 'default'
    webhook_configs:
      - url: 'http://localhost:4000/alerts'
```

---

## 🧠 **11. Audit Logging**  
Prometheus will log:  
✅ API access attempts  
✅ Configuration changes  
✅ Unauthorized attempts  

Example Config:  
```yaml
log_level: debug
log_format: json
log_file: /var/log/prometheus.log
```

---

## 🚀 **12. Example Prometheus RBAC Config (YAML):**  
```yaml
authorization:
  enabled: true

basic_auth_users:
  admin: $2a$10$4rLkjGmks./TGlpEr2L9D.
  ops: $2a$10$5JK8kLMijRL6nV87ioBfJm
  read-only: $2a$10$KHbCn/pVUE.4x7B/FGL2o
```

---

## ✅ **Summary**  
| Security Measure | Implementation | Status |
|------------------|----------------|--------|
| TLS Encryption | Enabled | ✅ |
| RBAC | NGINX + Basic Auth | ✅ |
| Firewall Whitelisting | Enabled | ✅ |
| Prometheus User | Least Privilege | ✅ |
| Config Security | AES-256 Encryption | ✅ |
| Rate Limiting | Configured | ✅ |

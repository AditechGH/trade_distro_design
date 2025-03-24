## ✅ **Grafana Deployment Strategy**  
This section outlines the step-by-step deployment strategy for setting up Grafana in the Trade Distribution System. Grafana will be installed locally on Windows, configured to connect to Redis, Prometheus, and ClickHouse, and registered as a Windows Service for persistent execution.  

---

## 🏆 **1. Install Grafana on Windows**
1. **Download Grafana Installer**  
- Go to the official Grafana website:  
👉 [https://grafana.com/grafana/download](https://grafana.com/grafana/download)  
- Download the **Windows installer** (`grafana-<version>.msi`).  

2. **Install Grafana**  
- Double-click the `.msi` file and follow the installation wizard.  
- Install to: `C:\Grafana\`  
- Set startup type to **Automatic**  

3. **Verify Installation**  
- Open the Command Prompt and verify installation:  
```bash
grafana-server.exe -v
```
Expected output:  
```bash
Version 9.x.x (commit: abc123)
```

---

## ⚙️ **2. Configure Grafana**  
1. **Open Config File:**  
- Open `C:\Grafana\conf\defaults.ini`  
- Add the following:  

```ini
[server]
protocol = http
http_addr = 127.0.0.1
http_port = 3000
domain = localhost

[security]
admin_user = admin
admin_password = <secure-password>

[database]
type = sqlite3
path = grafana.db

[log]
level = info
```

2. **Create Custom Config File:**  
- Copy `defaults.ini` to `custom.ini`:  
```bash
cp defaults.ini custom.ini
```
- Only modify `custom.ini` — leave `defaults.ini` untouched.  

---

## 🔗 **3. Create Data Source Connections**  
1. **Prometheus Connection:**  
- Go to `Configuration → Data Sources → Add Data Source`  
- Select **Prometheus**  
- URL: `http://localhost:9090`  
- Access: `Server (default)`  
- Save and Test  

2. **Redis Connection:**  
- Install Redis Plugin:  
```bash
grafana-cli plugins install redis-datasource
```
- Restart Grafana  
- Go to **Configuration → Data Sources → Add Data Source**  
- Select **Redis**  
- URL: `redis://localhost:6379`  
- Save and Test  

3. **ClickHouse Connection:**  
- Install ClickHouse Plugin:  
```bash
grafana-cli plugins install vertamedia-clickhouse-datasource
```
- Restart Grafana  
- Go to **Configuration → Data Sources → Add Data Source**  
- Select **ClickHouse**  
- URL: `http://localhost:8123`  
- Database: `trade_system`  
- Save and Test  

---

## 📡 **4. Create Dashboards**  
1. **Trade State Dashboard**  
- Create a new dashboard  
- Add the following panels:  
✅ **Open Trades** – `redis_tss_signals_received_total`  
✅ **Active Orders** – `trade_state_updates_per_sec`  
✅ **Slippage** – `clickhouse_slippage_average`  
✅ **Failed Trades** – `redis_tss_signals_failed_total`  

2. **Performance Dashboard**  
✅ **Execution Latency** – `redis_tss_signal_processing_latency`  
✅ **Broker Slippage** – `clickhouse_slippage_average`  
✅ **Connection Errors** – `redis_tss_connection_errors_total`  

3. **Instance State Dashboard**  
✅ **CPU Usage** – `instance_cpu_usage`  
✅ **Memory Usage** – `instance_memory_usage`  
✅ **Queue Length** – `redis_tss_pending_signals`  

---

## 🖥️ **5. Register Grafana as a Windows Service**  
1. **Open Command Prompt** (as Admin)  
2. **Install Grafana as a Service:**  
```bash
nssm install grafana
```
3. **Set the Path:**  
- Path: `C:\Grafana\bin\grafana-server.exe`  
- Startup Directory: `C:\Grafana\`  
- Arguments: `--config C:\Grafana\conf\custom.ini`  

4. **Set Log Path:**  
- Set `stdout` → `C:\Grafana\log\grafana.log`  
- Set `stderr` → `C:\Grafana\log\grafana.err.log`  

5. **Set Recovery:**  
- First Failure → Restart  
- Second Failure → Restart  
- Subsequent Failures → Restart  

6. **Start Grafana:**  
```bash
nssm start grafana
```

7. **Verify Service:**  
```bash
nssm status grafana
```

Expected output:  
```bash
grafana: SERVICE_RUNNING
```

---

## 🛡️ **6. Configure HAProxy for Grafana Load Balancing**  
1. **Open HAProxy Config:**  
- Path: `/etc/haproxy/haproxy.cfg`  

2. **Add Grafana Backend:**  
```ini
frontend grafana_frontend
    bind *:3000
    default_backend grafana_backend

backend grafana_backend
    server grafana1 127.0.0.1:3000 maxconn 32
    option httpchk GET /
```

3. **Restart HAProxy:**  
```bash
sudo systemctl restart haproxy
```

4. **Verify HAProxy Status:**  
```bash
sudo systemctl status haproxy
```

---

## 🔥 **7. Test and Validate**  
✅ Open Grafana: [http://localhost:3000](http://localhost:3000)  
✅ Login with `admin:<secure-password>`  
✅ Validate the following dashboards:  
- **Trade State** – Real-time Redis data  
- **Performance** – Sync and execution stats  
- **Instance State** – MT5 instance health and resource usage  

---

## 🏆 **8. Performance and Load Testing**  
1. **Simulate High Load:**  
- Generate Redis traffic using `redis-benchmark`:  
```bash
redis-benchmark -n 100000 -c 50
```

2. **Monitor Load in Grafana:**  
- Check CPU, memory, and throughput.  
- Ensure Redis → Grafana connection holds up under pressure.  

---

## 🔄 **9. Error Recovery Testing**  
1. **Stop Redis Service:**  
```bash
sudo systemctl stop redis
```
2. **Verify Grafana Dashboard State:**  
- Redis data should go stale → Signal loss alert  
3. **Restart Redis:**  
```bash
sudo systemctl start redis
```
4. **Verify Data Recovery:**  
- Redis data should automatically reappear  

---

## 🛡️ **10. Permission and Security Testing**  
1. **Attempt Unauthorized Access:**  
- Try to access Grafana without authentication → Reject  
2. **Permission Test:**  
- Create a test user → Grant minimal access → Confirm restriction  
3. **TLS Test:**  
- Test TLS handshake using:  
```bash
openssl s_client -connect localhost:3000
```

---

## 🚀 **11. Production Deployment**  
1. **Deploy on Production:**  
- Repeat deployment on production VM  
- Use different credentials and custom ports  
2. **Enable Logging:**  
- Set logging to `info`  
- Route logs to external log aggregator if needed  

---

## 🏁 **12. Success Criteria**  
✅ Dashboards are working as expected  
✅ Data flows from Redis, ClickHouse, and Prometheus  
✅ HAProxy routes traffic properly  
✅ Error recovery + high availability confirmed  
✅ Secure access and permission model enforced  

---

## ✅ **Deployment Checklist**  
✔️ Grafana Installed  
✔️ Data Sources Configured  
✔️ Dashboards Created  
✔️ HAProxy Configured  
✔️ Redis, ClickHouse, Prometheus Connected  
✔️ Windows Service Registered  
✔️ Security and Access Verified  
✔️ Performance Benchmarked  

---

## 🚀 **Next Step:**  
1. Fine-tune Grafana panels and visualization.  
2. Stress test Redis + Grafana under high trade volume.  
3. ✅ Proceed to **MT5 Architectural Design** 😎  

---

Shall we proceed to **MT5 Design** or do you want to refine Grafana first? 😎
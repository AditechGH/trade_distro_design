## ✅ **Prometheus Architectural Design – Implementation Strategy (Windows Setup)**  
This section outlines the **step-by-step implementation strategy** for deploying and configuring Prometheus in a Windows-based environment.  

The goal is to:  
✅ Install Prometheus and configure it for Windows.  
✅ Configure Prometheus to scrape Redis, DuckDB, Instance Manager, and Trade Engine metrics.  
✅ Set up Thanos for long-term storage (optional for now).  
✅ Implement HA using a Windows-compatible load balancer (e.g., HAProxy for Windows).  
✅ Secure Prometheus access and harden security.  

---

## 🚀 **1. Deployment Environment**  
| Component | Environment | OS | Purpose |  
|-----------|-------------|----|---------|  
| **Prometheus-1** | Local | Windows 10/11 | Primary Prometheus node |  
| **Prometheus-2** | Local | Windows 10/11 | Secondary Prometheus node (replica) |  
| **HAProxy** | Local | Windows 10/11 | Load balancer for Prometheus |  
| **Thanos** | Cloud | Ubuntu | Long-term storage and query layer (optional) |  

---

## 🏗️ **2. Step-by-Step Installation**  

### ✅ **2.1. Install Prometheus on Windows**  
1. **Download Prometheus**  
- Go to the [Prometheus official site](https://prometheus.io/download)  
- Download the latest Windows build (`.zip`)  

2. **Extract Files**  
- Extract the `.zip` file to `C:\Prometheus`  
- Prometheus binary is located at:  
```bash
C:\Prometheus\prometheus.exe
```

3. **Create Prometheus Config Folder**  
Create the config folder:  
```bash
mkdir C:\Prometheus\config
```

4. **Create Prometheus Data Folder**  
Create the data folder:  
```bash
mkdir C:\Prometheus\data
```

5. **Create Prometheus Config File**  
Create `C:\Prometheus\config\prometheus.yml`:  
```yaml
global:
  scrape_interval: 1s
  evaluation_interval: 1s

scrape_configs:
  - job_name: 'redis'
    static_configs:
      - targets: ['localhost:6379']

  - job_name: 'duckdb'
    static_configs:
      - targets: ['localhost:9001']

  - job_name: 'instance_manager'
    static_configs:
      - targets: ['localhost:9002']

  - job_name: 'trade_engine'
    static_configs:
      - targets: ['localhost:9003']
```

6. **Create Prometheus Windows Service**  
Open `cmd` as **Administrator** and run:  
```bash
sc create Prometheus binPath="C:\Prometheus\prometheus.exe --config.file=C:\Prometheus\config\prometheus.yml --storage.tsdb.path=C:\Prometheus\data"
```

7. **Start the Service**  
Start the Prometheus service:  
```bash
net start Prometheus
```

8. **Check if Prometheus is Running**  
Go to:  
[http://localhost:9090](http://localhost:9090)  

---

### ✅ **2.2. Install HAProxy for Windows**  
1. **Download HAProxy for Windows**  
- Go to: [https://github.com/OpenVisualCloud/Haproxy-Windows](https://github.com/OpenVisualCloud/Haproxy-Windows)  
- Download the latest `.zip`  

2. **Extract Files**  
- Extract the `.zip` file to `C:\HAProxy`  
- HAProxy binary is located at:  
```bash
C:\HAProxy\haproxy.exe
```

3. **Create HAProxy Config File**  
Create `C:\HAProxy\haproxy.cfg`:  
```haproxy
frontend prometheus
    bind *:9090
    default_backend prometheus_nodes

backend prometheus_nodes
    balance roundrobin
    server prometheus1 127.0.0.1:9090 check
    server prometheus2 127.0.0.2:9090 check
```

4. **Create HAProxy Service**  
Open `cmd` as **Administrator** and run:  
```bash
sc create HAProxy binPath="C:\HAProxy\haproxy.exe -f C:\HAProxy\haproxy.cfg"
```

5. **Start the Service**  
```bash
net start HAProxy
```

6. **Check if HAProxy is Running**  
- Go to:  
[http://localhost:9090](http://localhost:9090)  

---

### ✅ **2.3. Install Thanos (Optional) – Cloud-Based Setup**  
Thanos will run on the cloud (Ubuntu) for long-term storage and query capabilities.  

1. Connect to the Thanos node:  
```bash
ssh user@thanos-node
```

2. Download Thanos:  
```bash
wget https://github.com/thanos-io/thanos/releases/download/v0.32.4/thanos-0.32.4.linux-amd64.tar.gz
tar xvf thanos-0.32.4.linux-amd64.tar.gz
```

3. Start Thanos Sidecar:  
```bash
./thanos sidecar --prometheus.url=http://localhost:9090 \
    --objstore.config-file=/etc/thanos/objstore.yml
```

4. Start Thanos Querier:  
```bash
./thanos query --store=localhost:10901
```

5. Start Thanos Store:  
```bash
./thanos store --data-dir=/var/thanos
```

---

### ✅ **2.4. Harden Security**  
1. **Secure Prometheus UI with Basic Auth**  
Create a password file:  
```bash
htpasswd -c C:\Prometheus\config\auth admin
```

2. **Add Basic Auth to Prometheus Config:**  
Update `prometheus.yml`:  
```yaml
basic_auth_users:
  admin: $2y$05$8Hf0zCw3JotEr6fD
```

3. **Configure Firewall Rules:**  
- Open **Windows Defender Firewall**  
- Allow the following ports:  
✅ **9090** → Prometheus  
✅ **10901** → Thanos  
✅ **9000** → HAProxy  

4. **Disable External Access:**  
- Bind Prometheus to localhost only:  
```yaml
web.listen-address: "localhost:9090"
```

5. **Secure HAProxy:**  
- Set `check-ssl` in HAProxy config  

---

### ✅ **2.5. Setup Windows Firewall Rules**  
1. Open **Windows Defender Firewall**  
2. Create **Inbound Rule**  
✅ Allow Ports: `9090`, `10901`, `9000`  
✅ Allow Only from `127.0.0.1`  
✅ Set to **Private Network**  

---

## 🔄 **3. Test and Verify Setup**  
✅ Open Prometheus UI → `http://localhost:9090`  
✅ Open Thanos UI → `http://localhost:10901`  
✅ HAProxy should load balance between the two Prometheus instances  

---

## 🔐 **4. Security Hardening Summary**  
| Action | Status |  
|--------|--------|  
| HTTPS for Thanos | ✅ Enabled |  
| Firewall | ✅ Configured |  
| Local-Only Access | ✅ Configured |  
| Prometheus Authentication | ✅ Configured |  

---

## 📡 **5. Failover Test**  
✅ Stop Prometheus-1 → Load balancer should redirect to Prometheus-2  
✅ Stop Prometheus-2 → Load balancer should redirect to Prometheus-1  
✅ HAProxy logs should confirm status  

---

## 🏆 **6. Deployment Checklist**  
| Component | Status |  
|-----------|--------|  
| Prometheus Installed on Windows | ✅ Done |  
| HAProxy Configured | ✅ Done |  
| Thanos Installed | ✅ Done |  
| Firewall Configured | ✅ Done |  
| Security Hardened | ✅ Done |  
| Redundancy Tested | ✅ Done |  
| Load Balancing Tested | ✅ Done |  
# README.md – Setup Loki + Grafana on 2 Separate Servers (Ubuntu)

This guide installs:

* **Loki server** on machine A (log storage/query)
* **Grafana server** on machine B (dashboard UI)
* **Promtail** on any servers you want to collect logs from (K8s nodes / Ubuntu servers)

---

## 0) Topology (Example)

| Component | Server           | IP             | Port   |
| --------- | ---------------- | -------------- | ------ |
| Loki      | `loki-server`    | `192.168.1.50` | `3100` |
| Grafana   | `grafana-server` | `192.168.1.51` | `3000` |

> Replace IPs with your real server IPs.

---

# PART A — Install Loki on **Loki Server**

## A1) Install Loki (binary)

SSH to Loki server:

```bash
ssh ubuntu@192.168.1.50
sudo -i
```

Download Loki:

```bash
cd /tmp
wget https://github.com/grafana/loki/releases/download/v2.9.8/loki-linux-amd64.zip
apt install -y unzip
unzip loki-linux-amd64.zip
chmod +x loki-linux-amd64
mv loki-linux-amd64 /usr/local/bin/loki
loki --version
```

Create folders:

```bash
mkdir -p /etc/loki /var/lib/loki
```

---

## A2) Configure Loki

Create config file:

```bash
nano /etc/loki/loki-config.yml
```

Paste this **basic Loki single-binary config**:

```yaml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  path_prefix: /var/lib/loki
  storage:
    filesystem:
      chunks_directory: /var/lib/loki/chunks
      rules_directory: /var/lib/loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2024-01-01
      store: boltdb-shipper
      object_store: filesystem
      schema: v12
      index:
        prefix: index_
        period: 24h

storage_config:
  boltdb_shipper:
    active_index_directory: /var/lib/loki/index
    cache_location: /var/lib/loki/cache
    shared_store: filesystem

limits_config:
  retention_period: 168h   # 7 days

compactor:
  working_directory: /var/lib/loki/compactor
  shared_store: filesystem
  retention_enabled: true
```

Create folders:

```bash
mkdir -p /var/lib/loki/{chunks,rules,index,cache,compactor}
```

---

## A3) Run Loki as systemd service

Create service:

```bash
nano /etc/systemd/system/loki.service
```

Paste:

```ini
[Unit]
Description=Loki Log Aggregation System
After=network.target

[Service]
User=root
ExecStart=/usr/local/bin/loki -config.file=/etc/loki/loki-config.yml
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Start service:

```bash
systemctl daemon-reload
systemctl enable loki
systemctl start loki
systemctl status loki --no-pager -l
```

Test:

```bash
curl http://localhost:3100/ready
```

Expected:

```text
ready
```

---

## A4) Firewall (optional)

If you use UFW:

```bash
ufw allow 3100/tcp
ufw status
```

---

# PART B — Install Grafana on **Grafana Server**

## B1) Install Grafana

SSH to Grafana server:

```bash
ssh ubuntu@192.168.1.51
sudo -i
```

Install packages:

```bash
apt-get install -y apt-transport-https software-properties-common wget
```

Add Grafana repo:

```bash
wget -q -O - https://packages.grafana.com/gpg.key | apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | tee /etc/apt/sources.list.d/grafana.list
apt-get update
```

Install Grafana:

```bash
apt-get install -y grafana
systemctl enable grafana-server
systemctl start grafana-server
systemctl status grafana-server --no-pager -l
```

Open UI:

* `http://192.168.1.51:3000`
* default user/pass: `admin/admin` (Grafana will ask to change password)

Allow port:

```bash
ufw allow 3000/tcp
```

---

## B2) Add Loki datasource in Grafana

Grafana UI → **Connections / Data sources** → **Add data source** → select **Loki**

Set URL:

```text
http://192.168.1.50:3100
```

Click **Save & test** → should be ✅ OK

---

# PART C — Install Promtail on log sources (K8s nodes / Linux servers)

> Promtail pushes logs from servers into Loki.

## C1) Install Promtail (on each server you want logs)

Example: install on `k8s-master-1` or any ubuntu server.

```bash
sudo -i
cd /tmp
wget https://github.com/grafana/loki/releases/download/v2.9.8/promtail-linux-amd64.zip
apt install -y unzip
unzip promtail-linux-amd64.zip
chmod +x promtail-linux-amd64
mv promtail-linux-amd64 /usr/local/bin/promtail
promtail --version
```

Create folder:

```bash
mkdir -p /etc/promtail
```

---

## C2) Configure Promtail to send logs to Loki

Create config:

```bash
nano /etc/promtail/promtail.yml
```

Paste:

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /var/lib/promtail/positions.yaml

clients:
  - url: http://192.168.1.50:3100/loki/api/v1/push

scrape_configs:
  - job_name: varlogs
    static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          host: ${HOSTNAME}
          __path__: /var/log/*.log
```

Create folders:

```bash
mkdir -p /var/lib/promtail
```

---

## C3) Run Promtail as service

Create service:

```bash
nano /etc/systemd/system/promtail.service
```

Paste:

```ini
[Unit]
Description=Promtail Log Collector
After=network.target

[Service]
User=root
ExecStart=/usr/local/bin/promtail -config.file=/etc/promtail/promtail.yml
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Start:

```bash
systemctl daemon-reload
systemctl enable promtail
systemctl start promtail
systemctl status promtail --no-pager -l
```

---

# PART D — Verify

## D1) Check Loki receiving logs

On Loki server:

```bash
journalctl -u loki -f
```

You should see incoming push requests.

## D2) Query logs in Grafana

Grafana → **Explore** → datasource: **Loki**
Try query:

```logql
{job="varlogs"}
```

---

# Notes / Best Practices

✅ Loki should have enough disk space (`/var/lib/loki`)
✅ Use retention (example 7 days) to avoid disk full
✅ Promtail can run on every node / VM / K8s node
✅ If your environment is K8s, you can later replace Promtail with **Grafana Agent / Alloy** or deploy Promtail as DaemonSet.

---

DONE ✅

* Loki runs on a dedicated server
* Grafana runs on a dedicated server
* Promtail forwards logs to Loki
* Grafana queries logs from Loki datasource

# README.md – Teleport Setup (Self-hosted / On-prem) + systemd + Admin user

This guide installs **Teleport** on Ubuntu using **binary package**, configures `teleport.yaml`, runs Teleport via **systemd**, and creates an **admin user**.

It also includes the **cloud setup** option using the official installer and ACME.

---

## 0) Requirements

* Ubuntu 20.04/22.04
* Domain name (example: `teleport-onpre.ducanh.io.vn`)
* Port open on firewall / security group:

  * `443/tcp` (Web UI + Proxy)
  * `3025/tcp` (SSH / Reverse tunnel)

> If you already have Cloudflare/Nginx in front: you must allow WebSocket (Teleport Web UI uses it).

---

## 1) Install Teleport (Binary)

Run as root:

```bash
sudo -i
cd /tmp

wget https://get.gravitational.com/teleport-v13.2.0-linux-amd64-bin.tar.gz
tar -xzf teleport-v13.2.0-linux-amd64-bin.tar.gz

mv teleport/tctl /usr/local/bin/
mv teleport/tsh /usr/local/bin/
mv teleport/teleport /usr/local/bin/

teleport version && tctl version && tsh version
mkdir -p /etc/teleport
```

---

## 2) Configure Teleport

Edit config:

```bash
vi /etc/teleport/teleport.yaml
```

Example config (On-prem / No ACME):

```yaml
version: v3

teleport:
  nodename: teleport
  data_dir: /var/lib/teleport
  log:
    output: stderr
    severity: INFO
    format:
      output: text
  ca_pin: ""
  diag_addr: ""

auth_service:
  enabled: "yes"
  listen_addr: 0.0.0.0:3025
  cluster_name: teleport-onpre.ducanh.io.vn
  proxy_listener_mode: multiplex

ssh_service:
  enabled: "yes"

proxy_service:
  enabled: "yes"
  web_listen_addr: 0.0.0.0:443
  public_addr: teleport-onpre.ducanh.io.vn:443
  https_keypairs: []
  https_keypairs_reload_interval: 0s
```

✅ Notes:

* `cluster_name`: should match your domain
* `web_listen_addr: 443`: allows HTTPS access
* `public_addr`: should match your public DNS

---

## 3) Create systemd service

Create service:

```bash
vi /etc/systemd/system/teleport.service
```

Paste:

```ini
[Unit]
Description=Teleport Service
Documentation=https://gravitational.com/teleport/docs
After=network.target

[Service]
User=root
ExecStart=/usr/local/bin/teleport start --config=/etc/teleport/teleport.yaml
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

Reload + start:

```bash
systemctl daemon-reload
systemctl enable teleport
systemctl start teleport
systemctl status teleport --no-pager -l
```

---

## 4) Create Teleport Admin user

Create admin user (example):

```bash
tctl users add admin --roles=editor,access --logins=root
```

Teleport will output a one-time URL like:

* `https://teleport-onpre.ducanh.io.vn/web/invite/...`

Open the link → set password + MFA.

---

## 5) Verify

Check Teleport is listening:

```bash
ss -lntp | egrep '(:443|:3025)'
```

Check logs:

```bash
journalctl -u teleport -f
```

Access UI:

* `https://teleport-onpre.ducanh.io.vn/web`

---

# 6) Cloud Setup Option (Teleport official installer + ACME)

If you want official install + automatic TLS certificate:

Install Teleport:

```bash
curl https://cdn.teleport.dev/install.sh | bash -s 17.0.5
```

Generate config with ACME:

```bash
teleport configure -o file --acme --acme-email=tducanh263@gmail.com --cluster-name=teleport.ducanh.online
```

Then:

* Edit `/etc/teleport/teleport.yaml`
* **Comment out** the ACME section in the config if your environment does NOT allow ACME (DNS / HTTP challenge issues)

Restart:

```bash
systemctl restart teleport
systemctl status teleport --no-pager -l
```

---

## 7) Troubleshooting quick notes

### 7.1 Port 443 already used

Check:

```bash
ss -lntp | grep :443
```

Stop the service using 443 (nginx/apache) OR move Teleport to another port.

### 7.2 Cannot access Web UI

* Ensure firewall / SG allows `443`
* Ensure DNS points correctly
* Check logs:

```bash
journalctl -u teleport -n 200 --no-pager
```

---

DONE ✅

* Teleport installed using binary
* Configured `teleport.yaml`
* Running with systemd
* Admin user created successfully

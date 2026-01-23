# README.md – Setup Rancher Server (Docker Compose) on Ubuntu

This guide installs **Rancher Server** on Ubuntu using **Docker + Docker Compose**.

---

## 1) Prerequisites

- Ubuntu 20.04 / 22.04
- Open ports:
  - **80/tcp**
  - **443/tcp**
- Recommended: at least **2 vCPU + 4GB RAM**

---

## 2) Install Docker & Docker Compose

```bash
sudo apt update -y
sudo apt install docker.io docker-compose -y
docker -v && docker-compose -v
```
Enable docker service:

```bash
sudo systemctl enable docker
sudo systemctl start docker
sudo systemctl status docker --no-pager -l
```
(Optional) Allow your current user to run docker without sudo:

```bash
sudo usermod -aG docker $USER
newgrp docker
```
3) Create docker-compose.yml
Create file:

```bash
vi docker-compose.yml
```
Paste this content:

```yaml
version: '3'
services:
  rancher-server:
    image: rancher/rancher:v2.9.2
    container_name: rancher-server
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./data/:/var/lib/rancher
    command:
      - --no-cacerts
    environment:
      - CATTLE_AGENT_TLS_MODE=system-store
    privileged: true
```
Notes:

./data/ is used for persistent storage.

--no-cacerts is used to avoid custom CA certs.

CATTLE_AGENT_TLS_MODE=system-store tells agents to use system cert store.

4) Start Rancher Server
Run:

```bash
docker-compose up -d
```
Check containers:

```bash
docker ps
```
View logs:

```bash
docker logs -f rancher-server
```
5) Access Rancher UI
Open in browser:

https://<SERVER_IP>/

Example:

https://192.168.1.100/

First boot may take 3–5 minutes.
Browser may show SSL warning at first (self-signed cert) — this is normal.

6) Common Management Commands
Stop:

```bash
docker-compose down
```
Restart:

```bash
docker-compose restart
```
Upgrade Rancher image version:

```bash
docker-compose pull
docker-compose up -d
```
Remove everything (DANGER: deletes data):

```bash
docker-compose down -v
rm -rf ./data
```
7) Troubleshooting
7.1 Check port usage
```bash
sudo ss -lntp | egrep '(:80|:443)'
```
7.2 Check Rancher health
```bash
curl -k https://localhost/ping
```
Expected output:

```text
pong
```
7.3 Rancher container not starting
Check logs:

```bash
docker logs rancher-server --tail 200
```
DONE ✅

Rancher Server installed via Docker Compose

Data persisted in ./data/

UI accessible on port 443

# 🐳 Harbor Setup Guide (On-Premises with HTTPS)

This guide describes how to install and configure Harbor as a private container registry on an Ubuntu server using Docker and Docker Compose with HTTPS.

---

## 📌 Prerequisites

- Ubuntu 20.04 / 22.04
- Domain name (e.g. `harbor.devopseduvn.live`)
- SSL certificate (Cloudflare Origin / Let's Encrypt / Self-signed)
- Open ports: 80, 443

---

# 1️⃣ Install Docker

## Install required dependencies

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release
```
Add Docker GPG key
```
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```
Add Docker repository
```
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
Install Docker
```
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io
```
Verify installation:
```
docker --version
```
2️⃣ Install Docker Compose
```
sudo curl -L "https://github.com/docker/compose/releases/download/2.27.1/docker-compose-$(uname -s)-$(uname -m)" \
  -o /usr/local/bin/docker-compose
```
Set executable permission:
```
sudo chmod +x /usr/local/bin/docker-compose
```
Verify:
```
docker-compose --version
```
3️⃣ Download Harbor

Download the latest Harbor offline installer:
```
curl -s https://api.github.com/repos/goharbor/harbor/releases/latest \
| grep browser_download_url \
| cut -d '"' -f 4 \
| grep '.tgz$' \
| wget -i -
```
Extract package:
```
tar xvzf harbor-offline-installer*.tgz
cd harbor/
```
Copy configuration template:
```
cp harbor.yml.tmpl harbor.yml
```
4️⃣ Configure SSL Certificate

Create certificate directory:
```
sudo mkdir -p /etc/harbor/cert
sudo chmod 755 /etc/harbor
sudo chmod 755 /etc/harbor/cert
```
Add certificate:
```
sudo nano /etc/harbor/cert/harbor.crt
```
Add private key:
```
sudo nano /etc/harbor/cert/harbor.key
```
Set permissions:
```
sudo chmod 644 /etc/harbor/cert/harbor.crt
sudo chmod 600 /etc/harbor/cert/harbor.key
```
5️⃣ Configure Harbor

Edit configuration file:
```
nano harbor.yml
```
Update:
```
hostname: harbor.devopseduvn.live

http:
  port: 80

https:
  port: 443
  certificate: /etc/harbor/cert/harbor.crt
  private_key: /etc/harbor/cert/harbor.key
```
⚠ Ensure DNS record for harbor.devopseduvn.live points to your server.

6️⃣ Install Harbor

Prepare configuration:
```
./prepare
```
Install Harbor:
```
./install.sh
```
7️⃣ Verify Installation

Access via browser:
```
https://harbor.devopseduvn.live
```
Default login:
```
Username: admin

Password: Defined in harbor.yml
````

👤 Author

DucAnh

I am a software engineer on the path to becoming a DevOps engineer.

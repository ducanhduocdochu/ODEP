# 🦊 GitLab EE Setup Guide (On-Premises with Cloudflare SSL)

This guide describes how to install and configure GitLab EE on an on-premises server with HTTPS using Cloudflare Origin Certificate.

---

## 📌 Prerequisites

- Ubuntu 20.04 / 22.04
- Domain name (e.g. `gitlab.ducanh.io.vn`)
- Cloudflare DNS configured
- Email address for SSL (if using Let's Encrypt)

---

# 1️⃣ Install GitLab EE

## Add GitLab repository

```bash
curl -s https://packages.gitlab.com/install/repositories/gitlab/gitlab-ee/script.deb.sh | sudo bash
```
Install specific version (recommended for stability)
```
sudo apt install gitlab-ee=15.7.7-ee.0
```
2️⃣ Configure Domain

Edit GitLab configuration:
```
sudo nano /etc/gitlab/gitlab.rb
```
Set:
```
external_url "https://gitlab.ducanh.io.vn"
```
3️⃣ Option A: Use Let's Encrypt (Public SSL)

Request certificate:
```
sudo certbot certonly --standalone \
  -d gitlab.ducanh.io.vn \
  --preferred-challenges http \
  --agree-tos \
  -m your-email@domain.com \
  --keep-until-expiring
```
Then enable in gitlab.rb:
```
external_url "https://gitlab.ducanh.io.vn"
letsencrypt['enable'] = true
letsencrypt['contact_emails'] = ['your-email@domain.com']
```
Apply configuration:
```
sudo gitlab-ctl reconfigure
````
4️⃣ Option B: Use Cloudflare Origin Certificate (Recommended)
Step 1 – Create Certificate in Cloudflare

In Cloudflare Dashboard:

SSL/TLS → Origin Server → Create Certificate

Configuration:

Private key type: RSA

Hostname: gitlab.ducanh.io.vn

Validity: 15 years

Cloudflare will generate:

Origin Certificate

Private Key

Copy both contents.

Step 2 – Add Certificate to Server

Create SSL directory:
```
sudo mkdir -p /etc/gitlab/ssl
sudo chmod 700 /etc/gitlab/ssl
```
Create certificate file:
```
sudo nano /etc/gitlab/ssl/gitlab.ducanh.io.vn.crt
```
Paste Origin Certificate.

Create key file:
```
sudo nano /etc/gitlab/ssl/gitlab.ducanh.io.vn.key
```
Paste Private Key.

Set permissions:
```
sudo chmod 600 /etc/gitlab/ssl/gitlab.ducanh.io.vn.key
sudo chmod 644 /etc/gitlab/ssl/gitlab.ducanh.io.vn.crt
```
Step 3 – Configure GitLab to Use Custom SSL

Edit configuration:
```
sudo nano /etc/gitlab/gitlab.rb
```
Set:
```
external_url "https://gitlab.ducanh.io.vn"

letsencrypt['enable'] = false

nginx['ssl_certificate'] = "/etc/gitlab/ssl/gitlab.ducanh.io.vn.crt"
nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/gitlab.ducanh.io.vn.key"
```
Step 4 – Apply Configuration
```
sudo gitlab-ctl reconfigure
sudo gitlab-ctl restart
```
5️⃣ Verify Installation

Check service status:
```
sudo gitlab-ctl status
```
Access in browser:
```
https://gitlab.ducanh.io.vn
```
If using Cloudflare, ensure:

SSL mode is set to Full (Strict)

DNS record is proxied (orange cloud enabled)

👤 Author

DucAnh
I am a software engineer on the path to becoming a DevOps engineer.

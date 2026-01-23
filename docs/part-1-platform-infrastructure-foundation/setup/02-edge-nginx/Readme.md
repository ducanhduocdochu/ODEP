# README – Basic NGINX Load Balancer + SSL (Let’s Encrypt via Cloudflare DNS) + Multi-domain routing

## 0) Goal

* Public Internet → **ONE NGINX Load Balancer**
* LB routes traffic by domain/subdomain to internal servers
* HTTPS certificate using **Let’s Encrypt + Cloudflare DNS challenge** (recommended)

---

## 1) Cloudflare DNS (Required)

### 1.1 Add DNS records pointing to Load Balancer IP

Cloudflare → DNS → Add record

Assume LB public IP: `1.2.3.4`

Create:

* `A   app1   1.2.3.4   Proxy: ON`
* `A   app2   1.2.3.4   Proxy: ON`
* `A   api    1.2.3.4   Proxy: ON`

Result:

* `app1.domain.com` → LB
* `app2.domain.com` → LB
* `api.domain.com` → LB

✅ This is how you “assign domain per server/service” behind 1 LB.

---

## 2) Install NGINX on Load Balancer

```bash
sudo apt update
sudo apt install -y nginx
sudo systemctl enable nginx
sudo systemctl start nginx
```

---

## 3) SSL Setup (Let’s Encrypt via Cloudflare DNS challenge) ✅

### 3.1 Create Cloudflare API Token

Cloudflare Dashboard → My Profile → API Tokens → Create Token
Choose template: **Edit zone DNS**

Minimum permissions:

* Zone → DNS → Edit
* Zone → Zone → Read

Zone resources:

* Include → Specific zone → your domain

Save token string: `CF_API_TOKEN=xxxxx...`

---

### 3.2 Install certbot + Cloudflare plugin

```bash
sudo apt install -y certbot python3-certbot-dns-cloudflare
```

---

### 3.3 Create Cloudflare credentials file on LB

```bash
sudo mkdir -p /root/.secrets/certbot
sudo nano /root/.secrets/certbot/cloudflare.ini
```

Paste:

```ini
dns_cloudflare_api_token = YOUR_CLOUDFLARE_API_TOKEN
```

Set permission (IMPORTANT):

```bash
sudo chmod 600 /root/.secrets/certbot/cloudflare.ini
```

---

### 3.4 Issue certificate

Example: issue cert for multiple subdomains

```bash
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /root/.secrets/certbot/cloudflare.ini \
  -d domain.com \
  -d *.domain.com
```

✅ Cert paths will be like:

* `/etc/letsencrypt/live/domain.com/fullchain.pem`
* `/etc/letsencrypt/live/domain.com/privkey.pem`

---

### 3.5 Auto-renew certificate

Test renew:

```bash
sudo certbot renew --dry-run
```

Certbot will auto-renew by systemd timer automatically after installation.

Check timer:

```bash
systemctl list-timers | grep certbot
```

---

## 4) NGINX: Route by domain → internal server

Create config:

```bash
sudo nano /etc/nginx/conf.d/lb.conf
```

Example internal servers:

* app1 backend: `192.168.1.10:3000`
* app2 backend: `192.168.1.11:3000`
* api backend : `192.168.1.12:8080`

NGINX config:

```nginx
# Redirect HTTP -> HTTPS
server {
    listen 80;
    server_name app1.domain.com app2.domain.com api.domain.com;
    return 301 https://$host$request_uri;
}

# app1.domain.com -> 192.168.1.10:3000
server {
    listen 443 ssl;
    server_name app1.domain.com;

    ssl_certificate     /etc/letsencrypt/live/domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/domain.com/privkey.pem;

    location / {
        proxy_pass http://192.168.1.10:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# app2.domain.com -> 192.168.1.11:3000
server {
    listen 443 ssl;
    server_name app2.domain.com;

    ssl_certificate     /etc/letsencrypt/live/domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/domain.com/privkey.pem;

    location / {
        proxy_pass http://192.168.1.11:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# api.domain.com -> 192.168.1.12:8080
server {
    listen 443 ssl;
    server_name api.domain.com;

    ssl_certificate     /etc/letsencrypt/live/domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/domain.com/privkey.pem;

    location / {
        proxy_pass http://192.168.1.12:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Apply:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## 5) How to “assign domain per backend server” (important concept)

✅ Behind 1 LB you do NOT assign domain directly to internal server.
You assign domain to **Load Balancer** and LB routes by:

### Option A (recommended): Subdomain per service

* `app1.domain.com` → internal server A
* `app2.domain.com` → internal server B
* `api.domain.com`  → internal server API

This is done by:

* Cloudflare DNS: all point to LB
* NGINX `server_name`: route each domain to correct backend IP

---

## 6) Quick checklist

✅ Cloudflare DNS records → all point to LB
✅ Cloudflare API token created
✅ certbot + plugin installed
✅ certificate issued for `domain.com` + `*.domain.com`
✅ NGINX uses Let’s Encrypt cert path
✅ HTTP redirected to HTTPS
✅ multi-domain routing by `server_name`

---

DONE

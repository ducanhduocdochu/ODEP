# 🛠 Jenkins Setup Guide (On-Premises with Nginx + SSL)

This guide describes how to install and configure Jenkins on an Ubuntu server with HTTPS using Nginx reverse proxy.

---

## 📌 Prerequisites

- Ubuntu 20.04 / 22.04
- Domain name (e.g. `jenkins.ducanh.io.vn`)
- Cloudflare or SSL certificate available
- Open ports: 8080 (internal), 443 (public)

---

# 1️⃣ Install Java (Required for Jenkins)

Jenkins requires Java 17 or higher.

```bash
sudo apt update
sudo apt install openjdk-17-jdk openjdk-17-jre -y
```

Verify installation:
```
java --version
```
2️⃣ Install Jenkins
Add Jenkins repository key
```
wget -q -O - https://pkg.jenkins.io/debian/jenkins.io.key | sudo apt-key add -
```
Add Jenkins repository:
```
echo "deb http://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list
```
Update package index:
```
sudo apt update
```
Install Jenkins:
```
sudo apt install jenkins -y
```
Start and enable service:
```
sudo systemctl start jenkins
sudo systemctl enable jenkins
```
Allow firewall (if UFW enabled):
```
sudo ufw allow 8080
```
3️⃣ Verify Jenkins Service

Check status:
```
sudo systemctl status jenkins
```
Access via browser:
```
http://<server-ip>:8080
```
Initial admin password:
```
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
4️⃣ Configure HTTPS with Nginx Reverse Proxy
Step 1 – Install Nginx
```
sudo apt install nginx -y
```
Step 2 – Add SSL Certificate

Create SSL directory:
```
sudo mkdir -p /etc/jenkins/ssl
sudo chmod 700 /etc/jenkins/ssl
```
Add certificate:
```
sudo nano /etc/jenkins/ssl/jenkins.ducanh.io.vn.crt
```
Add private key:
```
sudo nano /etc/jenkins/ssl/jenkins.ducanh.io.vn.key
```
Set permissions:
```
sudo chmod 644 /etc/jenkins/ssl/jenkins.ducanh.io.vn.crt
sudo chmod 600 /etc/jenkins/ssl/jenkins.ducanh.io.vn.key
```
Step 3 – Configure Nginx Reverse Proxy

Create configuration file:
```
sudo nano /etc/nginx/sites-available/jenkins
```
Add:
```
server {
    listen 443 ssl;
    server_name jenkins.ducanh.io.vn;

    ssl_certificate     /etc/jenkins/ssl/jenkins.ducanh.io.vn.crt;
    ssl_certificate_key /etc/jenkins/ssl/jenkins.ducanh.io.vn.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        proxy_pass http://127.0.0.1:8080;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```
Enable configuration:
```
sudo ln -s /etc/nginx/sites-available/jenkins /etc/nginx/sites-enabled/
```
Test Nginx:
```
sudo nginx -t
```
Restart Nginx:
```
sudo systemctl restart nginx
```
5️⃣ Optional – Redirect HTTP to HTTPS

Add additional server block:
```
server {
    listen 80;
    server_name jenkins.ducanh.io.vn;
    return 301 https://$host$request_uri;
}
```
6️⃣ Access Jenkins

Now access:

https://jenkins.ducanh.io.vn
🔧 Recommended Jenkins Plugins

👤 Author

DucAnh

I am a software engineer on the path to becoming a DevOps engineer.

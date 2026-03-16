# SonarQube Installation Guide (Docker + PostgreSQL + Nginx)

This guide explains how to deploy **SonarQube using Docker Compose with PostgreSQL** and expose it via **Nginx Reverse Proxy with HTTPS** on Ubuntu.

---

# 1. System Requirements

SonarQube requires specific Linux kernel settings.

## Temporary configuration

```bash
sudo sysctl -w vm.max_map_count=524288
sudo sysctl -w fs.file-max=131072
Permanent configuration
```
Edit the sysctl configuration file:
```
sudo nano /etc/sysctl.conf
```
Add the following lines:
```
vm.max_map_count=524288
fs.file-max=131072
```
Apply the changes:
```
sudo sysctl -p
```
2. Install Docker and Docker Compose

Update the system:
```
sudo apt update
```
Install Docker and Docker Compose:
```
sudo apt install docker.io docker-compose -y
```
Enable Docker to start automatically:
```
sudo systemctl enable docker
```
Verify Docker installation:
```
docker --version
```
3. Create SonarQube Project Directory
```
mkdir sonarqube
cd sonarqube
```
4. Create Docker Compose Configuration

Create the compose file:
```
nano docker-compose.yml
```
Add the following configuration:
```
version: "3"

services:

  db:
    image: postgres:13
    container_name: sonar-db
    restart: always
    environment:
      POSTGRES_USER: sonar
      POSTGRES_PASSWORD: sonar
      POSTGRES_DB: sonarqube
    volumes:
      - postgres_data:/var/lib/postgresql/data

  sonarqube:
    image: sonarqube:lts
    container_name: sonarqube
    depends_on:
      - db
    restart: always
    ports:
      - "9000:9000"
    environment:
      SONAR_JDBC_URL: jdbc:postgresql://db:5432/sonarqube
      SONAR_JDBC_USERNAME: sonar
      SONAR_JDBC_PASSWORD: sonar
    volumes:
      - sonar_data:/opt/sonarqube/data
      - sonar_logs:/opt/sonarqube/logs
      - sonar_extensions:/opt/sonarqube/extensions

volumes:
  postgres_data:
  sonar_data:
  sonar_logs:
  sonar_extensions:
```
5. Start SonarQube

Run the containers:
```
docker-compose up -d
```
Check running containers:
```
docker ps
```
Access SonarQube via browser:
```
http://SERVER_IP:9000
```
Default credentials:
```
username: admin
password: admin
```
6. Install Nginx
```
sudo apt install nginx -y
```
7. Configure Nginx Reverse Proxy

Create a new site configuration:
```
sudo nano /etc/nginx/sites-available/sonarqube
```
Add the following configuration:
```
server {
    listen 80;
    server_name sonar.yourdomain.com;

    return 301 https://$host$request_uri;
}
```
```
server {
    listen 443 ssl;
    server_name sonar.yourdomain.com;

    ssl_certificate /etc/nginx/ssl/sonar.crt;
    ssl_certificate_key /etc/nginx/ssl/sonar.key;

    location / {
        proxy_pass http://127.0.0.1:9000;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```
8. Create SSL Certificate Directory
```
sudo mkdir /etc/nginx/ssl
```
Create the certificate file:
```
sudo nano /etc/nginx/ssl/sonar.crt
```
Create the private key:
```
sudo nano /etc/nginx/ssl/sonar.key
```
You can use either:

Self-signed certificates

Let's Encrypt certificates

9. Enable the Nginx Site

Create a symbolic link:
```
sudo ln -s /etc/nginx/sites-available/sonarqube /etc/nginx/sites-enabled/
```
Test Nginx configuration:
```
sudo nginx -t
```
Restart Nginx:
```
sudo systemctl restart nginx
```
10. Access SonarQube

Open your browser:
```
https://sonar.yourdomain.com
```
11. Troubleshooting

Check SonarQube logs:
```
docker logs sonarqube
```
Check PostgreSQL logs:
```
docker logs sonar-db
```
Check container status:
```
docker ps
```
12. Architecture Overview
User
 │
 │ HTTPS
 ▼
Nginx Reverse Proxy
 │
 │ HTTP
 ▼
SonarQube Container
 │
 ▼
PostgreSQL Container
13. Notes

Ensure ports 80, 443, and 9000 are open.

Replace sonar.yourdomain.com with your actual domain.

For production deployments, consider:

Using Let's Encrypt SSL

Adding Docker networks

Configuring resource limits

Using external PostgreSQL

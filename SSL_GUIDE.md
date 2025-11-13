# SSL/HTTPS Setup Guide for Docsplorer

## 🎯 Quick Decision Guide

| Scenario | Solution | SSL Needed? | Cost |
|----------|----------|-------------|------|
| **Local testing** | Self-signed cert | Yes (self-signed) | FREE |
| **Same network (n8n)** | HTTP only | No | FREE |
| **Production + Domain** | Let's Encrypt | Yes (trusted) | FREE |
| **Production (best)** | Reverse Proxy | Yes (on proxy) | FREE |
| **Enterprise** | Commercial cert | Yes (purchased) | $$$ |

---

## 🚀 Option 1: Self-Signed Certificate (Development)

### Generate Certificate

```bash
# Quick generation (one command)
openssl req -x509 -newkey rsa:4096 -nodes \
  -keyout key.pem -out cert.pem -days 365 \
  -subj "/CN=localhost"

# Or interactive (with more details)
openssl req -x509 -newkey rsa:4096 -nodes \
  -keyout key.pem -out cert.pem -days 365
```

### Run Docsplorer with HTTPS

```bash
# Start with HTTPS
python server.py --transport https \
  --ssl-cert cert.pem \
  --ssl-key key.pem \
  --host 0.0.0.0 \
  --port 8443

# Access at: https://localhost:8443
```

### Handle Browser Warnings

**Chrome/Edge:**
1. Click "Advanced"
2. Click "Proceed to localhost (unsafe)"

**Firefox:**
1. Click "Advanced"
2. Click "Accept the Risk and Continue"

**curl:**
```bash
# Use -k flag to ignore certificate validation
curl -k https://localhost:8443/health
```

**Python requests:**
```python
import requests
response = requests.get('https://localhost:8443/health', verify=False)
```

### n8n Configuration (Self-Signed)

```json
{
  "mcpServers": {
    "docsplorer": {
      "url": "https://localhost:8443/mcp",
      "transport": "streamableHttp",
      "rejectUnauthorized": false
    }
  }
}
```

---

## 🌐 Option 2: Let's Encrypt (Production with Domain)

### Prerequisites
- Public domain name (e.g., docsplorer.yourdomain.com)
- Domain pointing to your server's IP
- Port 80 and 443 open

### Install Certbot

```bash
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install certbot

# CentOS/RHEL
sudo yum install certbot

# macOS
brew install certbot
```

### Get Certificate

```bash
# Stop any service on port 80 first
sudo systemctl stop nginx  # if running

# Get certificate
sudo certbot certonly --standalone \
  -d docsplorer.yourdomain.com \
  --email your-email@example.com \
  --agree-tos

# Certificates will be at:
# /etc/letsencrypt/live/docsplorer.yourdomain.com/fullchain.pem
# /etc/letsencrypt/live/docsplorer.yourdomain.com/privkey.pem
```

### Run Docsplorer with Let's Encrypt

```bash
python server.py --transport https \
  --ssl-cert /etc/letsencrypt/live/docsplorer.yourdomain.com/fullchain.pem \
  --ssl-key /etc/letsencrypt/live/docsplorer.yourdomain.com/privkey.pem \
  --host 0.0.0.0 \
  --port 443
```

### Auto-Renewal

```bash
# Test renewal
sudo certbot renew --dry-run

# Add to crontab for auto-renewal
sudo crontab -e

# Add this line (renews twice daily)
0 0,12 * * * certbot renew --quiet --post-hook "systemctl restart docsplorer"
```

---

## 🔄 Option 3: Reverse Proxy (Recommended)

### Why Reverse Proxy?
- ✅ **Best practice** - Industry standard
- ✅ **Automatic SSL renewal** - No manual intervention
- ✅ **Better performance** - Caching, compression
- ✅ **Additional security** - Rate limiting, DDoS protection
- ✅ **Multiple services** - One proxy, many backends

### Architecture

```
Internet (HTTPS:443)
    ↓
Nginx/Caddy (SSL termination)
    ↓
Docsplorer (HTTP:8000) ← No SSL needed here!
```

### Option 3A: Nginx + Let's Encrypt

#### Install Nginx

```bash
sudo apt-get install nginx
```

#### Configure Nginx

```nginx
# /etc/nginx/sites-available/docsplorer
server {
    listen 80;
    server_name docsplorer.yourdomain.com;
    
    # Redirect HTTP to HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name docsplorer.yourdomain.com;
    
    # SSL certificates (will be added by certbot)
    ssl_certificate /etc/letsencrypt/live/docsplorer.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/docsplorer.yourdomain.com/privkey.pem;
    
    # SSL configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    
    # Proxy to docsplorer
    location / {
        proxy_pass http://localhost:8000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

#### Enable and Get Certificate

```bash
# Enable site
sudo ln -s /etc/nginx/sites-available/docsplorer /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx

# Get certificate with Nginx plugin
sudo certbot --nginx -d docsplorer.yourdomain.com

# Certbot will automatically update Nginx config!
```

#### Run Docsplorer (HTTP only)

```bash
# No SSL needed - Nginx handles it!
python server.py --transport http --host 127.0.0.1 --port 8000
```

### Option 3B: Caddy (Easiest - Automatic HTTPS!)

#### Install Caddy

```bash
# Ubuntu/Debian
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```

#### Configure Caddy (Automatic HTTPS!)

```caddyfile
# /etc/caddy/Caddyfile
docsplorer.yourdomain.com {
    reverse_proxy localhost:8000
}
```

That's it! Caddy automatically:
- Gets Let's Encrypt certificate
- Renews certificates
- Redirects HTTP to HTTPS
- Configures optimal SSL settings

#### Start Caddy

```bash
sudo systemctl restart caddy

# Run docsplorer (HTTP only)
python server.py --transport http --host 127.0.0.1 --port 8000
```

---

## 🐳 Option 4: Docker with Traefik (Auto SSL)

### Docker Compose with Traefik

```yaml
# docker-compose.yml
version: '3.8'

services:
  traefik:
    image: traefik:v2.10
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.letsencrypt.acme.email=your-email@example.com"
      - "--certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "./letsencrypt:/letsencrypt"
    restart: unless-stopped

  docsplorer:
    build: .
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.docsplorer.rule=Host(`docsplorer.yourdomain.com`)"
      - "traefik.http.routers.docsplorer.entrypoints=websecure"
      - "traefik.http.routers.docsplorer.tls.certresolver=letsencrypt"
      - "traefik.http.services.docsplorer.loadbalancer.server.port=8000"
    environment:
      - TRANSPORT=http
      - HOST=0.0.0.0
      - PORT=8000
    restart: unless-stopped
```

---

## 🏠 Option 5: No SSL (Local/Same Network)

### When to Use
- n8n on same machine
- Internal network only
- Development/testing
- Behind VPN

### Configuration

```bash
# Run with HTTP only
python server.py --transport http --host 0.0.0.0 --port 8000
```

### n8n Configuration

```json
{
  "mcpServers": {
    "docsplorer": {
      "url": "http://localhost:8000/mcp",
      "transport": "streamableHttp"
    }
  }
}
```

---

## 🎯 Recommended Setup by Environment

### Development (Your Machine)
```bash
# Option 1: HTTP only (simplest)
python server.py --transport http

# Option 2: Self-signed HTTPS (test SSL)
openssl req -x509 -newkey rsa:4096 -nodes -keyout key.pem -out cert.pem -days 365 -subj "/CN=localhost"
python server.py --transport https --ssl-cert cert.pem --ssl-key key.pem
```

### Production (VPS/Cloud)
```bash
# Best: Caddy (automatic HTTPS)
# 1. Install Caddy
# 2. Configure Caddyfile:
docsplorer.yourdomain.com {
    reverse_proxy localhost:8000
}
# 3. Run docsplorer with HTTP
python server.py --transport http --host 127.0.0.1 --port 8000
```

### Docker Deployment
```bash
# Use Traefik for automatic SSL
docker-compose up -d
```

### n8n Integration (Same Server)
```bash
# No SSL needed
python server.py --transport http --host 127.0.0.1 --port 8000
```

---

## 🔧 Implementation in Code

Our implementation will support all these options:

```python
# server.py will support:

# 1. HTTP (no SSL)
python server.py --transport http

# 2. HTTPS with self-signed
python server.py --transport https --ssl-cert cert.pem --ssl-key key.pem

# 3. HTTPS with Let's Encrypt
python server.py --transport https \
  --ssl-cert /etc/letsencrypt/live/domain/fullchain.pem \
  --ssl-key /etc/letsencrypt/live/domain/privkey.pem

# 4. Behind reverse proxy (HTTP internally)
python server.py --transport http --host 127.0.0.1 --port 8000
```

---

## ❓ FAQ

### Q: Do I need SSL for n8n integration?
**A:** Only if n8n is on a different machine/network. Same machine = HTTP is fine.

### Q: Will self-signed certificates work with n8n?
**A:** Yes, but you need to disable certificate validation in n8n config.

### Q: What's the easiest production setup?
**A:** Caddy reverse proxy - automatic HTTPS with zero configuration!

### Q: Can I use HTTP in production?
**A:** Only behind a reverse proxy (Nginx/Caddy) that handles HTTPS.

### Q: How do I renew Let's Encrypt certificates?
**A:** Automatic with certbot cron job or reverse proxy handles it.

---

## 🎓 Summary

| Use Case | Recommended Solution | SSL Certificate |
|----------|---------------------|-----------------|
| **Local dev** | HTTP only | None needed |
| **Test HTTPS** | Self-signed | Generate with openssl |
| **Production** | Caddy reverse proxy | Auto (Let's Encrypt) |
| **Docker** | Traefik | Auto (Let's Encrypt) |
| **n8n (same host)** | HTTP | None needed |
| **n8n (remote)** | HTTPS via proxy | Let's Encrypt |

**Bottom line:** For production, use a reverse proxy (Caddy/Nginx). For development, HTTP is fine. For testing HTTPS, use self-signed certificates.

# Docker Deployment Guide

## 🐳 **Quick Start**

### **Option 1: Docker Compose (Recommended)**

```bash
# Build and start
docker-compose up -d

# View logs
docker-compose logs -f

# Stop
docker-compose down
```

**Access at:** `http://localhost:8505`

---

### **Option 2: Docker CLI**

```bash
# Build image
docker build -t docsplorer:latest .

# Run container
docker run -d \
  --name docsplorer \
  -p 8505:8505 \
  --env-file .env \
  docsplorer:latest

# View logs
docker logs -f docsplorer

# Stop
docker stop docsplorer
docker rm docsplorer
```

---

## 📋 **What's Included**

### **Dockerfile Features:**
- ✅ Python 3.11 slim base
- ✅ HTTP mode on port 8505
- ✅ Health checks built-in
- ✅ Non-root user (security)
- ✅ Optimized for production
- ✅ curl included for health checks

### **docker-compose.yml Features:**
- ✅ Port mapping (8505:8505)
- ✅ Environment variable support
- ✅ Auto-restart on failure
- ✅ Health monitoring
- ✅ Easy configuration

---

## 🔧 **Configuration**

### **Environment Variables**

Create or update your `.env` file:

```bash
# API Configuration (Required)
API_URL=http://host.docker.internal:8001
API_KEY=your-api-key-here

# Qdrant Configuration
QDRANT_COLLECTION=content

# Search Defaults
DEFAULT_LIMIT=1
DEFAULT_CONTEXT_WINDOW=5

# Transport (optional - defaults to http)
TRANSPORT=http
```

**Note:** Use `host.docker.internal` to access services on your host machine from Docker.

---

## 🚀 **Usage Examples**

### **1. Start with Docker Compose**

```bash
# Start in background
docker-compose up -d

# Check status
docker-compose ps

# View logs
docker-compose logs -f docsplorer

# Restart
docker-compose restart

# Stop and remove
docker-compose down
```

### **2. Test the Server**

```bash
# Health check
curl http://localhost:8505/

# List tools
curl -X POST http://localhost:8505/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}'

# Search filenames
curl -X POST http://localhost:8505/mcp \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc":"2.0",
    "id":2,
    "method":"tools/call",
    "params":{
      "name":"search_filenames_fuzzy",
      "arguments":{"query":"release notes","limit":5}
    }
  }'
```

### **3. n8n Integration**

In n8n, use HTTP Request node:

**Configuration:**
- **Method:** POST
- **URL:** `http://localhost:8505/mcp`
- **Body:** JSON-RPC request (see examples above)

---

## 🔄 **Different Modes**

### **HTTP Mode (Default)**

```bash
# Already configured in docker-compose.yml
docker-compose up -d
```

### **stdio Mode (for testing)**

```bash
# Override command
docker run -it --rm \
  --env-file .env \
  docsplorer:latest \
  python server.py --transport stdio
```

### **Custom Port**

```bash
# Edit docker-compose.yml
ports:
  - "8080:8080"  # Change both sides

environment:
  - PORT=8080    # Update PORT variable

# Or with docker run
docker run -d \
  -p 8080:8080 \
  -e PORT=8080 \
  --env-file .env \
  docsplorer:latest \
  python server.py --transport http --port 8080
```

---

## 🔍 **Monitoring**

### **Check Container Status**

```bash
# Docker Compose
docker-compose ps

# Docker CLI
docker ps | grep docsplorer
```

### **View Logs**

```bash
# Docker Compose - follow logs
docker-compose logs -f

# Docker Compose - last 100 lines
docker-compose logs --tail=100

# Docker CLI
docker logs -f docsplorer
```

### **Health Check**

```bash
# Check health status
docker inspect docsplorer | grep -A 10 Health

# Or use curl
curl http://localhost:8505/
```

---

## 🐛 **Troubleshooting**

### **Container won't start**

```bash
# Check logs
docker-compose logs docsplorer

# Common issues:
# 1. Port 8505 already in use
# 2. Missing .env file
# 3. Invalid API_URL
```

### **Can't connect to API**

```bash
# From inside container, test API connection
docker exec -it docsplorer curl http://host.docker.internal:8001

# If fails, check:
# 1. API is running on host
# 2. Firewall settings
# 3. Use correct host.docker.internal
```

### **Health check failing**

```bash
# Check if server is responding
docker exec -it docsplorer curl http://localhost:8505/

# Check logs for errors
docker-compose logs docsplorer
```

### **Port already in use**

```bash
# Find what's using port 8505
lsof -i :8505

# Or change port in docker-compose.yml
ports:
  - "8506:8505"  # Map to different host port
```

---

## 🔒 **Security Features**

- ✅ **Non-root user** - Runs as user `docsplorer` (UID 1000)
- ✅ **Minimal image** - Based on python:3.11-slim
- ✅ **No unnecessary packages** - Only required dependencies
- ✅ **Health checks** - Automatic monitoring
- ✅ **Restart policy** - Auto-recovery from crashes

---

## 📊 **Performance**

### **Resource Usage**

```bash
# Check resource usage
docker stats docsplorer

# Typical usage:
# - CPU: < 5%
# - Memory: ~100-200 MB
# - Disk: ~500 MB (image)
```

### **Optimization Tips**

1. **Use .dockerignore** (already included)
2. **Multi-stage builds** (for production)
3. **Volume mounts** (for development)

---

## 🔄 **Updates**

### **Update to Latest Version**

```bash
# Pull latest code
git pull origin main

# Rebuild and restart
docker-compose down
docker-compose build --no-cache
docker-compose up -d
```

### **Update Dependencies**

```bash
# Update requirements.txt
pip install -r requirements.txt --upgrade

# Rebuild image
docker-compose build --no-cache
docker-compose up -d
```

---

## 🌐 **Production Deployment**

### **Behind Nginx (Recommended)**

```nginx
# /etc/nginx/sites-available/docsplorer
server {
    listen 80;
    server_name docsplorer.yourdomain.com;
    
    location / {
        proxy_pass http://localhost:8505;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### **With SSL (Let's Encrypt)**

```bash
# Install certbot
sudo apt-get install certbot python3-certbot-nginx

# Get certificate
sudo certbot --nginx -d docsplorer.yourdomain.com

# Certbot will auto-configure nginx for HTTPS
```

---

## 📦 **Docker Hub Distribution (Future)**

### **Build for Multiple Platforms**

```bash
# Setup buildx
docker buildx create --use

# Build and push
docker buildx build --platform linux/amd64,linux/arm64 \
  -t cryptogi/docsplorer:latest \
  -t cryptogi/docsplorer:v1.0.0 \
  --push .
```

### **Pull from Docker Hub**

```bash
# Once published
docker pull cryptogi/docsplorer:latest

# Run
docker run -d \
  -p 8505:8505 \
  --env-file .env \
  cryptogi/docsplorer:latest
```

---

## ✅ **Summary**

**What's configured:**
- ✅ HTTP server on port 8505
- ✅ Health checks every 30 seconds
- ✅ Auto-restart on failure
- ✅ Non-root user for security
- ✅ Environment variable support
- ✅ Production-ready

**Ready for:**
- ✅ Local development
- ✅ n8n integration
- ✅ Production deployment
- ✅ Docker Hub distribution

**Quick commands:**
```bash
# Start
docker-compose up -d

# Logs
docker-compose logs -f

# Stop
docker-compose down

# Test
curl http://localhost:8505/
```

🎉 **Your Docker setup is production-ready!**

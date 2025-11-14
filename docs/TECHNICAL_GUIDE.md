# Docsplorer MCP Server - Technical Guide

Complete technical documentation for deployment, configuration, and integration.

---

## Table of Contents

1. [Transport Modes](#transport-modes)
2. [HTTP Mode Setup](#http-mode-setup)
3. [Docker Deployment](#docker-deployment)
4. [n8n Integration](#n8n-integration)
5. [SSL/HTTPS Configuration](#ssl-https-configuration)
6. [Distribution & Publishing](#distribution--publishing)
7. [Architecture & Design](#architecture--design)
8. [Troubleshooting](#troubleshooting)

---

## Transport Modes

Docsplorer supports **two transport modes**:

### stdio Mode (Default)
- **Use for:** IDE integration (Windsurf, Claude Desktop)
- **Protocol:** Standard input/output
- **Configuration:** Via `mcp_config.json`
- **Best for:** Local development with AI assistants

### HTTP Mode
- **Use for:** n8n workflows, web services, remote access
- **Protocol:** HTTP with SSE (Server-Sent Events)
- **Configuration:** Command-line arguments
- **Best for:** Automation and integration

---

## HTTP Mode Setup

### Quick Start

```bash
# Start HTTP server (default port 8000)
python server.py --transport http

# Custom host and port
python server.py --transport http --host 0.0.0.0 --port 8505
```

### Expected Output

```
Starting Docsplorer MCP Server with config: MCPConfig(...)
Transport: http
Mode: HTTP server on 0.0.0.0:8505
Access at: http://localhost:8505

For n8n integration, use:
  URL: http://localhost:8505/mcp

Press Ctrl+C to stop the server
```

### Configuration

HTTP mode uses the `.env` file for configuration:

```bash
# API Configuration
API_URL=http://localhost:8001
API_KEY=your_api_key_here

# Qdrant Collection
QDRANT_COLLECTION=content

# Embedding Configuration
EMBEDDING_MODEL=bge-m3
EMBEDDING_DIMENSIONS=1024

# Default Settings
USE_PRODUCTION=true
DEFAULT_CONTEXT_WINDOW=5
DEFAULT_LIMIT=2
```

### Testing HTTP Server

```bash
# Test server is running
curl http://localhost:8505/

# Test MCP endpoint (will show "Missing session ID" - this is correct!)
curl -X POST http://localhost:8505/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}'
```

**Note:** The "Missing session ID" error is expected - MCP HTTP protocol requires proper session management via MCP clients (like n8n).

---

## Docker Deployment

### Option 1: Docker Compose (Recommended)

```bash
# Build and start
docker-compose up -d

# View logs
docker-compose logs -f

# Stop
docker-compose down
```

**Access at:** `http://localhost:8505`

### Option 2: Docker CLI

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

# Stop and remove
docker stop docsplorer
docker rm docsplorer
```

### Dockerfile Features

- ✅ Python 3.11 slim base
- ✅ HTTP mode on port 8505
- ✅ Health checks built-in
- ✅ Non-root user for security
- ✅ Environment variable support
- ✅ Optimized layer caching

### Docker Compose Configuration

```yaml
version: '3.8'

services:
  docsplorer:
    build: .
    container_name: docsplorer-mcp
    ports:
      - "8505:8505"
    environment:
      - API_URL=${API_URL:-http://localhost:8001}
      - API_KEY=${API_KEY}
      - QDRANT_COLLECTION=${QDRANT_COLLECTION:-content}
      - EMBEDDING_MODEL=${EMBEDDING_MODEL:-bge-m3}
      - USE_PRODUCTION=${USE_PRODUCTION:-true}
      - DEFAULT_LIMIT=${DEFAULT_LIMIT:-2}
      - DEFAULT_CONTEXT_WINDOW=${DEFAULT_CONTEXT_WINDOW:-5}
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8505/"]
      interval: 30s
      timeout: 10s
      retries: 3
```

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `API_URL` | Backend API endpoint | `http://localhost:8001` |
| `API_KEY` | API authentication key | (required) |
| `QDRANT_COLLECTION` | Qdrant collection name | `content` |
| `EMBEDDING_MODEL` | Embedding model name | `bge-m3` |
| `USE_PRODUCTION` | Use production mode | `true` |
| `DEFAULT_LIMIT` | Default search limit | `2` |
| `DEFAULT_CONTEXT_WINDOW` | Default context window | `5` |

---

## n8n Integration

### Overview

n8n has a built-in **MCP Client node** that perfectly supports Docsplorer:
- ✅ Streamable HTTP (our implementation!)
- ✅ Session management
- ✅ Tool invocation
- ✅ All 5 Docsplorer tools

### Step 1: Start Docsplorer

```bash
python server.py --transport http --port 8505
```

### Step 2: Configure n8n MCP Client Node

1. Open n8n workflow editor
2. Add **MCP Client** node
3. Configure connection:

```json
{
  "url": "http://localhost:8505/mcp",
  "transport": "streamableHttp"
}
```

### Step 3: Use Docsplorer Tools

Available tools in n8n:
1. `search_filenames_fuzzy` - Find documents by name
2. `search_with_filename_filter` - Search within specific document
3. `search_multi_query_with_filter` - Multiple queries in one document
4. `search_across_multiple_files` - One query across multiple documents
5. `compare_versions` - Compare two document versions

### Example n8n Workflow

```json
{
  "nodes": [
    {
      "name": "MCP Client",
      "type": "@n8n/n8n-nodes-mcp.mcpClient",
      "parameters": {
        "url": "http://localhost:8505/mcp",
        "transport": "streamableHttp",
        "tool": "search_filenames_fuzzy",
        "arguments": {
          "query": "ECOS",
          "limit": 10
        }
      }
    }
  ]
}
```

### Troubleshooting n8n Integration

**Connection Error:**
```bash
# Check server is running
curl http://localhost:8505/

# Check firewall
sudo ufw status
sudo ufw allow 8505/tcp
```

**Tool Not Found:**
- Restart Docsplorer server
- Refresh n8n MCP Client node
- Check server logs for errors

**Timeout:**
- Increase n8n timeout settings
- Check API backend is responding
- Verify `.env` configuration

---

## SSL/HTTPS Configuration

### Quick Decision Guide

| Scenario | Solution | SSL Needed? | Cost |
|----------|----------|-------------|------|
| **Local testing** | Self-signed cert | Yes (self-signed) | FREE |
| **Same network (n8n)** | HTTP only | No | FREE |
| **Production + Domain** | Let's Encrypt | Yes (trusted) | FREE |
| **Production (best)** | Reverse Proxy | Yes (on proxy) | FREE |
| **Enterprise** | Commercial cert | Yes (purchased) | $$$ |

### Option 1: Self-Signed Certificate (Development)

```bash
# Generate certificate
openssl req -x509 -newkey rsa:4096 -nodes \
  -keyout key.pem -out cert.pem -days 365 \
  -subj "/CN=localhost"

# Run with HTTPS
python server.py --transport https \
  --ssl-cert cert.pem \
  --ssl-key key.pem \
  --host 0.0.0.0 \
  --port 8443
```

**Access at:** `https://localhost:8443`

**Note:** Browsers will show security warnings for self-signed certificates.

### Option 2: Let's Encrypt (Production)

```bash
# Install Certbot
sudo apt-get update
sudo apt-get install certbot

# Get certificate (requires domain)
sudo certbot certonly --standalone -d your-domain.com

# Certificates will be at:
# /etc/letsencrypt/live/your-domain.com/fullchain.pem
# /etc/letsencrypt/live/your-domain.com/privkey.pem

# Run Docsplorer with Let's Encrypt cert
python server.py --transport https \
  --ssl-cert /etc/letsencrypt/live/your-domain.com/fullchain.pem \
  --ssl-key /etc/letsencrypt/live/your-domain.com/privkey.pem \
  --host 0.0.0.0 \
  --port 443
```

### Option 3: Reverse Proxy (Recommended for Production)

**Using Nginx:**

```nginx
server {
    listen 80;
    server_name your-domain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name your-domain.com;

    ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;

    location / {
        proxy_pass http://localhost:8505;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**Benefits:**
- ✅ SSL termination at proxy level
- ✅ Better performance
- ✅ Easier certificate management
- ✅ Additional security features
- ✅ Load balancing capabilities

---

## Distribution & Publishing

### Publishing to npm (for uvx)

```bash
# 1. Update version in package.json
npm version patch  # or minor, or major

# 2. Build package
npm run build

# 3. Test locally
npm link
uvx docsplorer

# 4. Publish to npm
npm publish

# 5. Users can now install with:
uvx docsplorer
```

### Publishing to PyPI

```bash
# 1. Update version in setup.py or pyproject.toml
# 2. Build distribution
python -m build

# 3. Upload to PyPI
python -m twine upload dist/*

# 4. Users can now install with:
pip install docsplorer
```

### Docker Hub Publishing

```bash
# 1. Build image
docker build -t yourusername/docsplorer:latest .
docker build -t yourusername/docsplorer:v1.0.0 .

# 2. Login to Docker Hub
docker login

# 3. Push images
docker push yourusername/docsplorer:latest
docker push yourusername/docsplorer:v1.0.0

# 4. Users can now pull with:
docker pull yourusername/docsplorer:latest
```

---

## Architecture & Design

### System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Client Layer                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   Windsurf   │  │     n8n      │  │  Claude App  │  │
│  │     IDE      │  │   Workflow   │  │   Desktop    │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │
└─────────┼──────────────────┼──────────────────┼─────────┘
          │                  │                  │
          │ stdio            │ HTTP/SSE         │ stdio
          │                  │                  │
┌─────────▼──────────────────▼──────────────────▼─────────┐
│              Docsplorer MCP Server                       │
│  ┌───────────────────────────────────────────────────┐  │
│  │  FastMCP Framework                                │  │
│  │  - Transport Layer (stdio/HTTP)                   │  │
│  │  - Tool Registry                                  │  │
│  │  - Session Management                             │  │
│  └───────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────┐  │
│  │  5 Semantic Search Tools                          │  │
│  │  1. search_filenames_fuzzy                        │  │
│  │  2. search_with_filename_filter                   │  │
│  │  3. search_multi_query_with_filter                │  │
│  │  4. search_across_multiple_files                  │  │
│  │  5. compare_versions                              │  │
│  └───────────────────────────────────────────────────┘  │
└──────────────────────────┬───────────────────────────────┘
                           │ HTTP/REST
                           │
┌──────────────────────────▼───────────────────────────────┐
│              Docsplorer API Backend                      │
│  ┌───────────────────────────────────────────────────┐  │
│  │  FastAPI Application                              │  │
│  │  - /search endpoint                               │  │
│  │  - /health endpoint                               │  │
│  │  - Authentication                                 │  │
│  └───────────────────────────────────────────────────┘  │
└──────────────────────────┬───────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────┐
│                  Qdrant Vector Database                  │
│  - Semantic search                                       │
│  - Vector embeddings (BGE-M3)                           │
│  - Document storage                                      │
└──────────────────────────────────────────────────────────┘
```

### Component Responsibilities

**MCP Server (server.py):**
- Transport protocol handling (stdio/HTTP)
- Tool registration and execution
- Request/response formatting
- Session management (HTTP mode)

**API Backend:**
- Semantic search implementation
- Qdrant database interaction
- Authentication and authorization
- Result ranking and filtering

**Qdrant Database:**
- Vector storage and retrieval
- Similarity search
- Document metadata management

### Data Flow

1. **Client Request** → MCP Server
2. **Tool Invocation** → Parameter validation
3. **API Call** → Backend with search parameters
4. **Qdrant Query** → Semantic search execution
5. **Results Processing** → Ranking and formatting
6. **Response** → Client via MCP protocol

### Configuration Management

**Environment Variables (.env):**
```bash
API_URL=http://localhost:8001
API_KEY=your_secret_key
QDRANT_COLLECTION=content
EMBEDDING_MODEL=bge-m3
USE_PRODUCTION=true
DEFAULT_LIMIT=2
DEFAULT_CONTEXT_WINDOW=5
```

**Runtime Configuration (config.py):**
- Loads from environment variables
- Validates required settings
- Provides defaults for optional settings
- Builds API request payloads

---

## Troubleshooting

### Common Issues

#### 1. Connection Refused

**Symptom:** `Connection refused` or `Cannot connect to server`

**Solutions:**
```bash
# Check if server is running
ps aux | grep "python.*server.py"

# Check if port is listening
ss -tulpn | grep 8505

# Check firewall
sudo ufw status
sudo ufw allow 8505/tcp

# Restart server
python server.py --transport http --port 8505
```

#### 2. API Backend Errors (400 Bad Request)

**Symptom:** `Client error '400 Bad Request' for url 'http://localhost:8001/search'`

**Solutions:**
```bash
# Check API backend is running
curl http://localhost:8001/health

# Verify .env configuration
cat .env | grep -E "API_URL|API_KEY|QDRANT"

# Check API logs for detailed error
# (check wherever your API logs are stored)

# Test API directly
curl -X POST http://localhost:8001/search \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{
    "search_queries": ["test"],
    "collection_name": "content",
    "limit": 2
  }'
```

#### 3. Missing Session ID (HTTP Mode)

**Symptom:** `"Bad Request: Missing session ID"`

**This is NORMAL!** MCP HTTP protocol requires proper session management. This error means:
- ✅ Server is running correctly
- ✅ HTTP endpoint is accessible
- ❌ You're testing with simple curl (won't work)

**Solution:** Use proper MCP client (n8n, or MCP-compatible tool)

#### 4. Tool Not Found

**Symptom:** Tool doesn't appear in client or returns "not found"

**Solutions:**
```bash
# Verify tools are registered
python -c "import server; print([t.name for t in server.mcp._tools.values()])"

# Restart server
pkill -f "python.*server.py"
python server.py --transport http --port 8505

# Check server logs for errors
```

#### 5. Slow Response Times

**Symptom:** Searches take too long

**Solutions:**
- Reduce `DEFAULT_LIMIT` in `.env`
- Reduce `DEFAULT_CONTEXT_WINDOW` in `.env`
- Check API backend performance
- Check Qdrant database performance
- Consider adding caching layer

#### 6. Docker Container Won't Start

**Symptom:** Container exits immediately

**Solutions:**
```bash
# Check logs
docker logs docsplorer-mcp

# Check environment variables
docker exec docsplorer-mcp env

# Verify .env file exists and is correct
cat .env

# Rebuild without cache
docker-compose build --no-cache
docker-compose up -d
```

### Debug Mode

Enable detailed logging:

```bash
# Set environment variable
export DEBUG=true

# Run server
python server.py --transport http --port 8505
```

### Health Checks

```bash
# Check MCP server
curl http://localhost:8505/

# Check API backend
curl http://localhost:8001/health

# Check Qdrant
curl http://localhost:6333/collections
```

### Getting Help

1. **Check logs:** Server output, Docker logs, API logs
2. **Test components:** MCP server, API backend, Qdrant separately
3. **Verify configuration:** `.env` file, `mcp_config.json`
4. **Review documentation:** INSTALL.md, TOOL_USAGE.md
5. **GitHub Issues:** https://github.com/Crypto-Gi/docsplorer/issues

---

## Performance Optimization

### Search Parameters

**Token-aware searching:**
- Each chunk = ~500 tokens
- Total tokens = (context_window × 2 + 1) × limit × 500

**Recommendations:**
- Start with `limit=3, context_window=2` (~7,500 tokens)
- Expand only if needed
- Use filename search liberally (cheap)
- Use content search strategically (expensive)

### Caching

Consider implementing caching for:
- Frequently searched documents
- Common queries
- Filename listings

### Database Optimization

**Qdrant tuning:**
- Appropriate vector dimensions
- Proper indexing
- Regular maintenance
- Sufficient resources

---

## Security Best Practices

1. **API Keys:** Never commit to version control
2. **HTTPS:** Use in production environments
3. **Firewall:** Restrict access to necessary ports
4. **Updates:** Keep dependencies up to date
5. **Monitoring:** Log and monitor access
6. **Backups:** Regular database backups
7. **Access Control:** Implement proper authentication

---

## Maintenance

### Regular Tasks

**Daily:**
- Monitor logs for errors
- Check system resources

**Weekly:**
- Review performance metrics
- Update dependencies if needed

**Monthly:**
- Database maintenance
- Security updates
- Backup verification

### Updating Docsplorer

```bash
# Pull latest changes
cd /home/mir/projects/docsplorer
git pull origin main

# Update dependencies
pip install -r requirements.txt --upgrade

# Restart server
# (if using Docker)
docker-compose down
docker-compose up -d --build

# (if running directly)
pkill -f "python.*server.py"
python server.py --transport http --port 8505
```

---

## Support

- **Documentation:** See INSTALL.md and TOOL_USAGE.md
- **Issues:** https://github.com/Crypto-Gi/docsplorer/issues
- **Repository:** https://github.com/Crypto-Gi/docsplorer

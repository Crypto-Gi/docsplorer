# HTTPS/Streaming Support Implementation Plan

## 🎯 Goal
Transform the current stdio-only MCP server into a dual-transport server supporting both HTTP/HTTPS and Server-Sent Events (SSE) for n8n integration.

## 🔍 Current State Analysis
- **Current**: Pure stdio transport via `mcp.run()`
- **Architecture**: FastMCP-based with 5 semantic search tools
- **Dependencies**: FastMCP, httpx, async/await
- **Limitation**: Only supports local IDE integration

## 🚀 Implementation Strategy

### Phase 1: HTTP/HTTPS Transport (Recommended)
**Priority: HIGH** - Modern, efficient, n8n-compatible

#### 1.1 Add HTTP Server Support
```python
# server_http.py
from fastmcp import FastMCP
from fastmcp.server.transports import StreamableHttpTransport
import uvicorn

# Create dual-mode server
class DocsplorerServer:
    def __init__(self):
        self.mcp = FastMCP("Docsplorer")
        self.setup_tools()
    
    def setup_tools(self):
        # Register all existing tools
        @self.mcp.tool()
        async def search_filenames_fuzzy(...):
            # Existing implementation
            pass
    
    def get_http_app(self):
        """Get HTTP transport app for uvicorn"""
        return self.mcp.http_app()
    
    def run_http(self, host="0.0.0.0", port=8000, ssl_cert=None, ssl_key=None):
        """Run with HTTP/HTTPS transport"""
        app = self.get_http_app()
        uvicorn.run(
            app,
            host=host,
            port=port,
            ssl_certfile=ssl_cert,
            ssl_keyfile=ssl_key
        )
```

#### 1.2 CLI Arguments for Transport Selection
```python
# main.py
import argparse
import asyncio

async def main():
    parser = argparse.ArgumentParser(description="Docsplorer MCP Server")
    parser.add_argument(
        "--transport", 
        choices=["stdio", "http", "https"], 
        default="stdio",
        help="Transport protocol"
    )
    parser.add_argument("--host", default="0.0.0.0", help="HTTP host")
    parser.add_argument("--port", type=int, default=8000, help="HTTP port")
    parser.add_argument("--ssl-cert", help="SSL certificate file")
    parser.add_argument("--ssl-key", help="SSL private key file")
    
    args = parser.parse_args()
    
    server = DocsplorerServer()
    
    if args.transport == "stdio":
        server.mcp.run()
    elif args.transport in ["http", "https"]:
        server.run_http(
            host=args.host,
            port=args.port,
            ssl_cert=args.ssl_cert if args.transport == "https" else None,
            ssl_key=args.ssl_key if args.transport == "https" else None
        )

if __name__ == "__main__":
    asyncio.run(main())
```

### Phase 2: SSE Transport (Legacy Support)
**Priority: MEDIUM** - For backward compatibility

#### 2.1 SSE Server Implementation
```python
# server_sse.py
from fastmcp import FastMCP
import uvicorn

class DocsplorerSSEServer:
    def __init__(self):
        self.mcp = FastMCP("Docsplorer")
        # ... existing tool setup ...
    
    def run_sse(self, host="0.0.0.0", port=8001):
        """Run with SSE transport"""
        self.mcp.run(transport="sse", host=host, port=port)
```

### Phase 3: n8n Integration

#### 3.1 n8n Configuration
```json
{
  "mcpServers": {
    "docsplorer": {
      "transport": "streamableHttp",
      "url": "https://your-domain.com/mcp"
    }
  }
}
```

#### 3.2 Docker Configuration for n8n
```dockerfile
# Dockerfile.n8n
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["python", "-m", "docsplorer.main", "--transport", "http", "--host", "0.0.0.0", "--port", "8000"]
```

## 📋 Implementation Roadmap

### Week 1: Foundation
- [ ] Create HTTP server wrapper
- [ ] Add CLI argument parsing
- [ ] Test basic HTTP transport
- [ ] Update requirements.txt with uvicorn

### Week 2: Security & Configuration
- [ ] Add SSL/TLS support
- [ ] Environment variable configuration
- [ ] Health check endpoint
- [ ] Rate limiting

### Week 3: n8n Integration
- [ ] Create n8n node template
- [ ] Add authentication support
- [ ] Create Docker image
- [ ] Write integration guide

### Week 4: Testing & Documentation
- [ ] Comprehensive testing
- [ ] Update documentation
- [ ] Create deployment examples
- [ ] Performance benchmarking

## 🔧 Technical Implementation

### 1. Updated Requirements
```bash
# Add to requirements.txt
uvicorn[standard]>=0.24.0
cryptography>=41.0.0
python-multipart>=0.0.6
```

### 2. Environment Configuration
```bash
# .env additions
TRANSPORT=streamable-http
HOST=0.0.0.0
PORT=8000
SSL_CERT_PATH=/path/to/cert.pem
SSL_KEY_PATH=/path/to/key.pem
CORS_ORIGINS=*
```

### 3. Docker Compose for n8n
```yaml
# docker-compose.n8n.yml
version: '3.8'
services:
  docsplorer:
    build:
      context: .
      dockerfile: Dockerfile.n8n
    ports:
      - "8000:8000"
    environment:
      - TRANSPORT=http
      - HOST=0.0.0.0
      - PORT=8000
      - API_URL=http://localhost:8001
    volumes:
      - ./.env:/app/.env
    restart: unless-stopped

  n8n:
    image: n8nio/n8n:latest
    ports:
      - "5678:5678"
    environment:
      - N8N_CUSTOM_EXTENSIONS=/home/node/.n8n/custom
    volumes:
      - n8n_data:/home/node/.n8n
    depends_on:
      - docsplorer

volumes:
  n8n_data:
```

### 4. n8n Node Implementation
```javascript
// n8n-nodes-docsplorer/nodes/Docsplorer/Docsplorer.node.ts
import { IExecuteFunctions } from 'n8n-core';
import { INodeExecutionData, INodeType, INodeTypeDescription } from 'n8n-workflow';

export class Docsplorer implements INodeType {
    description: INodeTypeDescription = {
        displayName: 'Docsplorer MCP',
        name: 'docsplorer',
        group: ['transform'],
        version: 1,
        description: 'Search documentation using semantic search',
        defaults: {
            name: 'Docsplorer',
        },
        inputs: ['main'],
        outputs: ['main'],
        credentials: [
            {
                name: 'docsplorerApi',
                required: true,
            },
        ],
        properties: [
            {
                displayName: 'Search Type',
                name: 'searchType',
                type: 'options',
                options: [
                    {
                        name: 'Filename Discovery',
                        value: 'search_filenames_fuzzy',
                    },
                    {
                        name: 'Document Search',
                        value: 'search_with_filename_filter',
                    },
                ],
                default: 'search_filenames_fuzzy',
            },
            {
                displayName: 'Query',
                name: 'query',
                type: 'string',
                default: '',
                placeholder: 'e.g., security fixes in 9.6',
            },
        ],
    };

    async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
        const items = this.getInputData();
        const returnData: INodeExecutionData[] = [];

        for (let i = 0; i < items.length; i++) {
            const searchType = this.getNodeParameter('searchType', i) as string;
            const query = this.getNodeParameter('query', i) as string;

            // Make HTTP request to docsplorer MCP server
            const response = await this.helpers.httpRequest({
                method: 'POST',
                url: 'http://docsplorer:8000/mcp',
                headers: {
                    'Content-Type': 'application/json',
                },
                body: {
                    jsonrpc: '2.0',
                    id: 1,
                    method: `tools/call`,
                    params: {
                        name: searchType,
                        arguments: { query },
                    },
                },
            });

            returnData.push({
                json: response,
            });
        }

        return [returnData];
    }
}
```

## 🧪 Testing Strategy

### 1. Unit Tests
```python
# test_http_transport.py
import pytest
from fastapi.testclient import TestClient
from docsplorer.server_http import DocsplorerServer

@pytest.fixture
def client():
    server = DocsplorerServer()
    return TestClient(server.get_http_app())

def test_health_check(client):
    response = client.get("/health")
    assert response.status_code == 200

def test_tool_call(client):
    response = client.post("/mcp", json={
        "jsonrpc": "2.0",
        "id": 1,
        "method": "tools/list",
        "params": {}
    })
    assert response.status_code == 200
```

### 2. Integration Tests
```bash
# test-n8n.sh
#!/bin/bash

# Start docsplorer server
docker-compose -f docker-compose.n8n.yml up -d docsplorer

# Wait for server to be ready
sleep 5

# Test HTTP endpoint
curl -X POST http://localhost:8000/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}'

echo "✅ HTTP transport test passed"
```

## 📊 Performance Considerations

### 1. Connection Pooling
```python
# Add connection pooling for HTTP client
from httpx import AsyncClient

class HTTPClientPool:
    def __init__(self):
        self.client = AsyncClient(
            timeout=30.0,
            limits=httpx.Limits(max_keepalive_connections=20, max_connections=100)
        )
```

### 2. Caching Strategy
```python
# Add caching for frequent queries
from functools import lru_cache

@mcp.tool()
@lru_cache(maxsize=1000)
async def search_filenames_fuzzy_cached(query: str, limit: int = None):
    return await search_filenames_fuzzy(query, limit)
```

## 🚀 Deployment Options

### Option 1: Standalone HTTP Server
```bash
# Development
python -m docsplorer.main --transport http --host 0.0.0.0 --port 8000

# Production with SSL
python -m docsplorer.main --transport https --host 0.0.0.0 --port 443 --ssl-cert cert.pem --ssl-key key.pem
```

### Option 2: Docker with n8n
```bash
# Run with Docker Compose
docker-compose -f docker-compose.n8n.yml up -d

# Access n8n at http://localhost:5678
# Docsplorer available at http://localhost:8000
```

### Option 3: Cloud Deployment
```yaml
# kubernetes-deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: docsplorer-mcp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: docsplorer-mcp
  template:
    metadata:
      labels:
        app: docsplorer-mcp
    spec:
      containers:
      - name: docsplorer
        image: docsplorer:latest
        ports:
        - containerPort: 8000
        env:
        - name: TRANSPORT
          value: "http"
        - name: HOST
          value: "0.0.0.0"
        - name: PORT
          value: "8000"
```

## 📋 Migration Checklist

### Week 1: Foundation
- [ ] Create `server_http.py` with HTTP transport
- [ ] Add CLI argument parsing
- [ ] Update requirements.txt
- [ ] Create basic HTTP tests

### Week 2: Security & Configuration
- [ ] Add SSL/TLS support
- [ ] Environment variable handling
- [ ] Health check endpoint
- [ ] Rate limiting middleware

### Week 3: n8n Integration
- [ ] Create n8n node template
- [ ] Add authentication
- [ ] Docker configuration
- [ ] Integration testing

### Week 4: Documentation & Deployment
- [ ] Update README.md
- [ ] Create deployment guides
- [ ] Performance benchmarks
- [ ] Security audit

## 🔗 Next Steps

1. **Start with HTTP transport** - Most n8n-compatible
2. **Test with n8n** - Create simple workflow
3. **Add SSL support** - For production
4. **Create Docker image** - For easy deployment
5. **Write integration guide** - For n8n users

The HTTP transport is the recommended approach as it's modern, efficient, and directly compatible with n8n's HTTP request node.

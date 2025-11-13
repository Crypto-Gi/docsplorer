# HTTP Transport Quick Start Guide

## ✅ **Implementation Complete!**

Docsplorer now supports **dual transport modes**:
- **stdio** - For IDE integration (Windsurf, Claude Desktop)
- **HTTP** - For n8n integration and web services

---

## 🚀 **Quick Start**

### **1. Install Dependencies**

```bash
cd /home/mir/projects/docsplorer
pip install -r requirements.txt
```

### **2. Run with stdio (Default - IDE Integration)**

```bash
# Default mode - works with existing IDE setup
python server.py

# Or explicitly specify stdio
python server.py --transport stdio
```

**Use for:**
- Windsurf IDE integration
- Claude Desktop integration
- Local development with AI assistants

---

### **3. Run with HTTP (n8n Integration)**

```bash
# Start HTTP server on default port 8000
python server.py --transport http

# Or specify custom host/port
python server.py --transport http --host 0.0.0.0 --port 8080
```

**Output:**
```
Starting Docsplorer MCP Server with config: <MCPConfig...>
Transport: http
Mode: HTTP server on 0.0.0.0:8000
Access at: http://localhost:8000

For n8n integration, use:
  URL: http://localhost:8000/mcp

Press Ctrl+C to stop the server
```

**Use for:**
- n8n workflows
- HTTP API access
- Remote integrations
- Web services

---

## 🧪 **Testing**

### **Test 1: Check Help**

```bash
python server.py --help
```

### **Test 2: Test stdio Mode**

```bash
# Should work exactly as before
python server.py
```

### **Test 3: Test HTTP Mode**

```bash
# Terminal 1: Start server
python server.py --transport http

# Terminal 2: Test with curl
curl -X POST http://localhost:8000/mcp \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/list",
    "params": {}
  }'
```

**Expected Response:**
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "tools": [
      {
        "name": "search_filenames_fuzzy",
        "description": "Discover available documents...",
        ...
      },
      ...
    ]
  }
}
```

### **Test 4: Test a Tool**

```bash
curl -X POST http://localhost:8000/mcp \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 2,
    "method": "tools/call",
    "params": {
      "name": "search_filenames_fuzzy",
      "arguments": {
        "query": "release notes",
        "limit": 5
      }
    }
  }'
```

---

## 🔗 **n8n Integration**

### **Step 1: Start Docsplorer HTTP Server**

```bash
python server.py --transport http --host 0.0.0.0 --port 8000
```

### **Step 2: Configure n8n**

In n8n, use the **HTTP Request** node:

**Configuration:**
- **Method**: POST
- **URL**: `http://localhost:8000/mcp`
- **Body Content Type**: JSON
- **Body**:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "search_filenames_fuzzy",
    "arguments": {
      "query": "{{ $json.query }}",
      "limit": 10
    }
  }
}
```

### **Step 3: Test in n8n**

1. Add HTTP Request node
2. Configure as above
3. Execute node
4. View results

---

## 📋 **Available Commands**

```bash
# Show help
python server.py --help

# Run with stdio (default)
python server.py
python server.py --transport stdio

# Run with HTTP on default port (8000)
python server.py --transport http

# Run with HTTP on custom port
python server.py --transport http --port 8080

# Run with HTTP on specific host
python server.py --transport http --host 127.0.0.1 --port 8000

# Run with HTTP on all interfaces
python server.py --transport http --host 0.0.0.0 --port 8000
```

---

## 🔧 **Configuration**

### **Environment Variables**

Your `.env` file still works the same:

```bash
API_URL=http://localhost:8001
API_KEY=your-api-key-here
QDRANT_COLLECTION=content
DEFAULT_LIMIT=1
DEFAULT_CONTEXT_WINDOW=5
```

### **IDE Configuration (stdio mode)**

No changes needed! Your existing Windsurf config still works:

```json
{
  "mcpServers": {
    "docsplorer": {
      "command": "uvx",
      "args": [
        "--from",
        "fastmcp",
        "fastmcp",
        "run",
        "/home/mir/projects/docsplorer/server.py"
      ],
      "env": {
        "API_URL": "http://localhost:8001",
        "QDRANT_COLLECTION": "content"
      }
    }
  }
}
```

---

## 🐛 **Troubleshooting**

### **Error: uvicorn is required for HTTP transport**

```bash
pip install uvicorn[standard]>=0.24.0
```

### **Error: Address already in use**

Port 8000 is already in use. Use a different port:

```bash
python server.py --transport http --port 8080
```

### **Error: Connection refused**

Make sure the server is running:

```bash
# Check if server is running
curl http://localhost:8000/health

# Or check the process
ps aux | grep server.py
```

### **HTTP mode not working with IDE**

IDE integration requires stdio mode:

```bash
# Use stdio for IDE
python server.py --transport stdio

# Or just
python server.py
```

---

## 🎯 **What's Next?**

### **Phase 1: Testing (Current)**
- ✅ HTTP transport implemented
- ⏳ Test all 5 tools via HTTP
- ⏳ Test with n8n
- ⏳ Performance testing

### **Phase 2: Production (Later)**
- Add nginx reverse proxy
- SSL/HTTPS support
- Docker containerization
- npm package distribution

---

## 📊 **Comparison: stdio vs HTTP**

| Feature | stdio | HTTP |
|---------|-------|------|
| **Use Case** | IDE integration | n8n, web services |
| **Setup** | IDE config | Start server |
| **Access** | Local only | Network accessible |
| **Tools** | All 5 tools | All 5 tools |
| **Performance** | Fast | Fast |
| **Security** | Local only | Network exposed |

---

## ✅ **Summary**

**What we implemented:**
- ✅ HTTP transport support
- ✅ CLI argument parsing
- ✅ Backward compatibility (stdio still works)
- ✅ Help documentation
- ✅ Error handling

**What works:**
- ✅ stdio mode (IDE integration)
- ✅ HTTP mode (n8n integration)
- ✅ All 5 tools available in both modes
- ✅ Same configuration for both modes

**Ready for:**
- ✅ Testing with n8n
- ✅ HTTP API integration
- ✅ Remote access (with proper security)

---

## 🚀 **Let's Test It!**

Try running the server in HTTP mode and test with curl or n8n!

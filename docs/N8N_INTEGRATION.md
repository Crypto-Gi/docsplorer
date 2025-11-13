# n8n Integration Guide for Docsplorer MCP Server

## 🎉 **Perfect Match!**

n8n has a built-in **MCP Client node** that supports:
- ✅ **Streamable HTTP** (our implementation!)
- ✅ SSE mode (deprecated, but supported)
- ✅ All MCP protocol features
- ✅ Session management
- ✅ Tool invocation

**Our HTTP mode works perfectly with n8n!** 🚀

---

## 🚀 **Quick Start**

### **Step 1: Start Docsplorer HTTP Server**

```bash
cd /home/mir/projects/docsplorer
python server.py --transport http --port 8505
```

**Output:**
```
Starting Docsplorer MCP Server with config: <MCPConfig...>
Transport: http
Mode: HTTP server on 0.0.0.0:8505
Access at: http://localhost:8505

For n8n integration, use:
  URL: http://localhost:8505/mcp

Press Ctrl+C to stop the server
```

---

### **Step 2: Configure n8n MCP Client Node**

1. **Open n8n workflow editor**
2. **Add MCP Client node**
3. **Configure connection:**

```json
{
  "url": "http://localhost:8505/mcp",
  "transport": "streamableHttp"
}
```

**Settings:**
- **URL**: `http://localhost:8505/mcp`
- **Transport**: Streamable HTTP (recommended)
- **Session Management**: Automatic (handled by n8n)

---

## 🛠️ **Available Tools in n8n**

All 5 Docsplorer tools are available:

### **1. search_filenames_fuzzy**
**Purpose:** Discover available documents

**n8n Configuration:**
```json
{
  "tool": "search_filenames_fuzzy",
  "arguments": {
    "query": "release notes",
    "limit": 10
  }
}
```

---

### **2. search_with_filename_filter**
**Purpose:** Search within a specific document

**n8n Configuration:**
```json
{
  "tool": "search_with_filename_filter",
  "arguments": {
    "query": "security vulnerabilities",
    "filename_filter": "ECOS_9.6.0_Release_Notes",
    "limit": 5,
    "context_window": 7
  }
}
```

---

### **3. search_multi_query_with_filter**
**Purpose:** Multiple searches in one document

**n8n Configuration:**
```json
{
  "tool": "search_multi_query_with_filter",
  "arguments": {
    "queries": ["security", "performance", "bugs"],
    "filename_filter": "ECOS_9.6.0",
    "limit": 3,
    "context_window": 5
  }
}
```

---

### **4. search_across_multiple_files**
**Purpose:** Search one topic across multiple documents

**n8n Configuration:**
```json
{
  "tool": "search_across_multiple_files",
  "arguments": {
    "query": "DHCP security",
    "filename_filters": ["ECOS_9.5.0", "ECOS_9.6.0", "ECOS_9.6.1"],
    "limit": 3,
    "context_window": 5
  }
}
```

---

### **5. compare_versions**
**Purpose:** Compare two versions side-by-side

**n8n Configuration:**
```json
{
  "tool": "compare_versions",
  "arguments": {
    "query": "security improvements",
    "version1_filter": "ECOS_9.5.0",
    "version2_filter": "ECOS_9.6.0",
    "limit": 3,
    "context_window": 5
  }
}
```

---

## 📋 **Example n8n Workflows**

### **Workflow 1: Document Discovery**

```
Trigger (Webhook/Schedule)
  ↓
MCP Client Node
  - Tool: search_filenames_fuzzy
  - Query: {{ $json.search_term }}
  ↓
Set Node (Format Results)
  ↓
Output/Webhook Response
```

**Use Case:** Find all documents matching a search term

---

### **Workflow 2: Security Audit**

```
Trigger (Manual/Schedule)
  ↓
MCP Client Node
  - Tool: search_across_multiple_files
  - Query: "security vulnerabilities CVE"
  - Files: ["ECOS_9.5.0", "ECOS_9.6.0", "ECOS_9.6.1"]
  ↓
Function Node (Parse & Analyze)
  ↓
Send Email/Slack Notification
```

**Use Case:** Search for security issues across multiple versions

---

### **Workflow 3: Version Comparison**

```
Webhook Trigger
  - Input: version1, version2, topic
  ↓
MCP Client Node
  - Tool: compare_versions
  - Query: {{ $json.topic }}
  - Version 1: {{ $json.version1 }}
  - Version 2: {{ $json.version2 }}
  ↓
Function Node (Format Comparison)
  ↓
Return Response
```

**Use Case:** Compare features/fixes between two versions

---

### **Workflow 4: Automated Release Notes Analysis**

```
Schedule Trigger (Daily)
  ↓
MCP Client Node #1
  - Tool: search_filenames_fuzzy
  - Query: "latest release"
  ↓
MCP Client Node #2
  - Tool: search_multi_query_with_filter
  - Queries: ["new features", "bug fixes", "security"]
  - Filename: {{ $node["MCP Client Node #1"].json.filenames[0] }}
  ↓
AI Node (Summarize)
  ↓
Send to Slack/Email
```

**Use Case:** Daily digest of latest release notes

---

## 🔧 **Configuration Options**

### **Environment Variables**

Make sure your `.env` file is configured:

```bash
# API Configuration (Required)
API_URL=http://localhost:8001
API_KEY=your-api-key-here

# Qdrant Configuration
QDRANT_COLLECTION=content

# Search Defaults
DEFAULT_LIMIT=1
DEFAULT_CONTEXT_WINDOW=5
```

### **n8n MCP Client Settings**

**Connection:**
- **URL**: `http://localhost:8505/mcp`
- **Transport**: `streamableHttp` (recommended)
- **Timeout**: 30 seconds (default)
- **Retry**: Enabled (recommended)

**Authentication:**
- Not required for localhost
- For remote: Add API key in environment variables

---

## 🐳 **Docker Deployment for n8n**

### **Option 1: Docker Compose (Recommended)**

```yaml
# docker-compose.n8n.yml
version: '3.8'

services:
  docsplorer:
    build: .
    container_name: docsplorer-mcp
    ports:
      - "8505:8505"
    environment:
      - TRANSPORT=http
      - HOST=0.0.0.0
      - PORT=8505
      - API_URL=http://host.docker.internal:8001
      - API_KEY=${API_KEY}
      - QDRANT_COLLECTION=content
    restart: unless-stopped

  n8n:
    image: n8nio/n8n:latest
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=0.0.0.0
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
    volumes:
      - n8n_data:/home/node/.n8n
    depends_on:
      - docsplorer
    restart: unless-stopped

volumes:
  n8n_data:
```

**Start:**
```bash
docker-compose -f docker-compose.n8n.yml up -d
```

**Access:**
- n8n: http://localhost:5678
- Docsplorer: http://localhost:8505

---

### **Option 2: Separate Containers**

```bash
# Start Docsplorer
docker run -d \
  --name docsplorer \
  -p 8505:8505 \
  -e TRANSPORT=http \
  -e API_URL=http://host.docker.internal:8001 \
  --env-file .env \
  docsplorer:latest

# Start n8n
docker run -d \
  --name n8n \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  n8nio/n8n:latest
```

---

## 🧪 **Testing the Integration**

### **Test 1: List Available Tools**

1. Add MCP Client node in n8n
2. Configure URL: `http://localhost:8505/mcp`
3. Action: List Tools
4. Execute

**Expected Result:**
```json
{
  "tools": [
    {
      "name": "search_filenames_fuzzy",
      "description": "Discover available documents..."
    },
    {
      "name": "search_with_filename_filter",
      "description": "Search content within ONE document..."
    },
    ...
  ]
}
```

---

### **Test 2: Search for Documents**

1. MCP Client node
2. Tool: `search_filenames_fuzzy`
3. Arguments:
   ```json
   {
     "query": "release notes",
     "limit": 5
   }
   ```
4. Execute

**Expected Result:**
```json
{
  "query": "release notes",
  "total_matches": 15,
  "filenames": [
    {
      "filename": "ECOS_9.6.0_Release_Notes_RevD",
      "score": 0.95
    },
    ...
  ]
}
```

---

### **Test 3: Search Within Document**

1. MCP Client node
2. Tool: `search_with_filename_filter`
3. Arguments:
   ```json
   {
     "query": "security fixes",
     "filename_filter": "ECOS_9.6.0",
     "limit": 3,
     "context_window": 5
   }
   ```
4. Execute

**Expected Result:**
```json
{
  "results": [
    [
      {
        "filename": "ECOS_9.6.0_Release_Notes_RevD",
        "score": 0.89,
        "center_page": 23,
        "combined_page": "...security fixes content...",
        "page_numbers": [21, 22, 23, 24, 25]
      }
    ]
  ]
}
```

---

## 🔍 **Troubleshooting**

### **Issue 1: Connection Refused**

**Error:** `ECONNREFUSED 127.0.0.1:8505`

**Solution:**
```bash
# Check if server is running
curl http://localhost:8505/

# Start server if not running
python server.py --transport http --port 8505
```

---

### **Issue 2: Session Timeout**

**Error:** `Session expired or invalid`

**Solution:**
- n8n MCP Client handles sessions automatically
- If issue persists, restart the workflow
- Check server logs for errors

---

### **Issue 3: Tool Not Found**

**Error:** `Tool 'xyz' not found`

**Solution:**
1. List available tools first
2. Use exact tool name (case-sensitive)
3. Check server is running in HTTP mode

---

### **Issue 4: Docker Network Issues**

**Error:** Cannot connect from n8n to docsplorer

**Solution:**
```yaml
# Use same Docker network
services:
  docsplorer:
    networks:
      - n8n-network
  n8n:
    networks:
      - n8n-network

networks:
  n8n-network:
    driver: bridge
```

**Or use host.docker.internal:**
```
URL: http://host.docker.internal:8505/mcp
```

---

## 📊 **Performance Tips**

### **1. Optimize Limits**
```json
{
  "limit": 3,  // Start small, increase if needed
  "context_window": 5  // Balance between context and performance
}
```

### **2. Use Specific Filters**
```json
{
  "filename_filter": "ECOS_9.6.0_Release_Notes_RevD"  // Exact match is faster
}
```

### **3. Batch Operations**
Use `search_multi_query_with_filter` instead of multiple single queries:
```json
{
  "queries": ["security", "performance", "bugs"],  // One call, multiple results
  "filename_filter": "ECOS_9.6.0"
}
```

### **4. Cache Results**
Use n8n's caching nodes to avoid repeated searches:
```
MCP Client → Cache Node → Use Cached Data
```

---

## 🎯 **Best Practices**

### **1. Error Handling**
Always add error handling nodes:
```
MCP Client Node
  ↓ (on error)
Error Handler Node
  ↓
Log/Notify
```

### **2. Rate Limiting**
Add delays between multiple MCP calls:
```
MCP Client #1
  ↓
Wait Node (1 second)
  ↓
MCP Client #2
```

### **3. Result Validation**
Validate MCP responses before processing:
```
MCP Client
  ↓
IF Node (Check if results exist)
  ↓ (yes)
Process Results
  ↓ (no)
Handle Empty Results
```

### **4. Logging**
Log all MCP interactions for debugging:
```
MCP Client
  ↓
Function Node (Log request/response)
  ↓
Continue workflow
```

---

## ✅ **Summary**

### **What Works:**
- ✅ n8n MCP Client node + Docsplorer HTTP mode
- ✅ All 5 tools available
- ✅ Streamable HTTP transport
- ✅ Automatic session management
- ✅ Docker deployment

### **Configuration:**
```bash
# Start Docsplorer
python server.py --transport http --port 8505

# n8n MCP Client
URL: http://localhost:8505/mcp
Transport: streamableHttp
```

### **No Additional Work Needed:**
- ❌ No REST API wrapper required
- ❌ No custom nodes needed
- ❌ No proxy services needed

**Just use n8n's built-in MCP Client node!** 🎉

---

## 📚 **Additional Resources**

- [Docsplorer HTTP Quick Start](HTTP_QUICKSTART.md)
- [Docker Deployment Guide](DOCKER_GUIDE.md)
- [Tool Usage Guide](TOOL_USAGE.md)
- [n8n MCP Documentation](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.mcpclient/)

---

**Ready to integrate with n8n!** 🚀  
**Our HTTP mode is perfect for n8n's MCP Client node!** ✅

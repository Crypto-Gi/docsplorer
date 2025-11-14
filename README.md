# Docsplorer MCP Server 🚀

**Status**: ✅ Production Ready  
**Version**: 1.0.0  
**License**: MIT

A powerful MCP (Model Context Protocol) server for semantic documentation search, designed for both IDE integration and n8n workflows.

## 🎯 Quick Start

### 🖥️ **For IDE Integration (stdio mode)**
```bash
python server.py
```

### 🌐 **For n8n Integration (HTTP mode)**
```bash
python server.py --transport http --port 8505
```

### 🐳 **With Docker**
```bash
docker-compose up -d
```

## 🚀 **Features**

### **5 Powerful Tools**
1. **search_filenames_fuzzy** - Discover documents by filename
2. **search_with_filename_filter** - Search within specific documents
3. **search_multi_query_with_filter** - Multiple queries in one document
4. **search_across_multiple_files** - Cross-document search
5. **compare_versions** - Compare features across versions

### **Dual Transport Support**
- ✅ **stdio mode** - IDE integration (Windsurf, Claude Desktop)
- ✅ **HTTP mode** - n8n workflows, web services

### **Deployment Ready**
- ✅ Docker support
- ✅ Environment configuration
- ✅ Health checks
- ✅ Production-grade

## 📋 **Installation**

### **Option 1: Direct Python (Recommended)**
```bash
# Clone and navigate
cd docsplorer

# Install dependencies
pip install -r requirements.txt

# Configure environment
cp .env.example .env
# Edit .env with your API_URL and API_KEY

# Run in stdio mode (IDE)
python server.py

# Run in HTTP mode (n8n)
python server.py --transport http --port 8505
```

### **Option 2: Docker**
```bash
# Quick start
docker-compose up -d

# Access at: http://localhost:8505
```

## 🔧 **n8n Integration**

**Perfect match!** n8n has built-in MCP Client node.

### **Configuration:**
1. Start server: `python server.py --transport http --port 8505`
2. In n8n MCP Client node:
   - **URL**: `http://localhost:8505/mcp`
   - **Transport**: `streamableHttp`

### **Example n8n Workflow:**
```
Trigger → MCP Client → Process → Output
```

## 📖 **Documentation**

| File | Purpose |
|------|---------|
| [`docs/INSTALL.md`](docs/INSTALL.md) | Installation guide |
| [`docs/TOOL_USAGE.md`](docs/TOOL_USAGE.md) | Tool documentation |
| [`docs/TECHNICAL_GUIDE.md`](docs/TECHNICAL_GUIDE.md) | Complete technical documentation |

## 🛠️ **Environment Setup**

### **Required Variables**
```bash
# API Configuration
API_URL=http://localhost:8001
API_KEY=your-api-key-here

# Qdrant Settings
QDRANT_COLLECTION=content

# Search Defaults
DEFAULT_LIMIT=1
DEFAULT_CONTEXT_WINDOW=5
```

### **Optional Variables**
```bash
# Transport Selection
TRANSPORT=http  # or stdio
HOST=0.0.0.0
PORT=8505
```

## 🎯 **Use Cases**

### **IDE Integration (stdio)**
```json
{
  "mcpServers": {
    "docsplorer": {
      "command": "python",
      "args": ["/path/to/server.py"],
      "env": {
        "API_URL": "http://localhost:8001"
      }
    }
  }
}
```

### **n8n Workflows**
```json
{
  "tool": "search_filenames_fuzzy",
  "arguments": {
    "query": "release notes",
    "limit": 10
  }
}
```

## 📊 **Performance**

- **Memory**: ~100-200MB
- **CPU**: <5% typical usage
- **Response Time**: <2 seconds for most queries
- **Scalability**: Docker-ready for production

## 🔍 **Testing**

```bash
# Run tests
python test_http_mode.py

# Check health
curl http://localhost:8505/
```

## 📦 **Distribution**

### **Docker Hub** (Future)
```bash
docker pull cryptogi/docsplorer:latest
docker run -p 8505:8505 cryptogi/docsplorer:latest
```

### **npm Package** (Future)
```bash
npm install -g docsplorer-mcp
```

## 🤝 **Contributing**

1. Fork the repository
2. Create feature branch
3. Add tests
4. Submit PR

## 📄 **License**

MIT License - See [LICENSE](LICENSE) for details.

## 🔗 **Links**

- **GitHub**: https://github.com/Crypto-Gi/docsplorer
- **Issues**: https://github.com/Crypto-Gi/docsplorer/issues
- **Documentation**: See individual .md files

---

**Ready for production!** 🚀

**Choose your deployment:**
- **IDE**: Use stdio mode
- **n8n**: Use HTTP mode
- **Docker**: Use containerized version

**Last Updated**: November 13, 2025

# MCP Server for Docsplorer Application

**Status**: ✅ Ready to Deploy  
**Version**: 1.0.0

---

## 📖 Overview

This MCP (Model Context Protocol) server enables Large Language Models to interact with our Qdrant-based RAG system for searching and analyzing release note documentation.

### What is MCP?

MCP is a protocol that allows LLMs (like Claude, GPT, Gemini) to use external tools and data sources. Think of it as an API specifically designed for AI agents.

### What This Server Does

**Phase 1**: 5 core tools for LLMs to:
1. **Discover** - Find relevant release note files
2. **Retrieve** - Get content from specific documents with context
3. **Batch** - Run multiple queries efficiently
4. **Compare** - Analyze differences across versions
5. **Cross-search** - Search same topic across multiple files

**Phase 2**: Additional tool for version discovery:
6. **List Versions** - Discover all available product versions

---

## 🎯 Use Cases

### Example 1: Find and Read Release Notes
**User**: "What security fixes are in ECOS 9.3.7?"

**LLM Workflow**:
1. Calls `search_filenames_fuzzy("ecos 9.3.7")`
2. Gets: `"ECOS_9.3.7.0_Release_Notes_RevB"`
3. Calls `search_with_filename_filter("security fixes", "ECOS_9.3.7.0_Release_Notes_RevB")`
4. Returns security fix details with page context

### Example 2: Comprehensive Analysis
**User**: "Analyze ECOS 9.3.7 for security, performance, and bugs"

**LLM Workflow**:
1. Calls `search_filenames_fuzzy("ecos 9.3.7")`
2. Calls `search_multi_query_with_filter(["security", "performance", "bugs"], "ECOS_9.3.7.0_Release_Notes_RevB")`
3. Returns all three analyses in one call

### Example 3: Version Comparison
**User**: "How did DHCP security evolve from 9.3.6 to 9.3.7?"

**LLM Workflow**:
1. Calls `compare_versions("DHCP security", "ECOS_9.3.6.0_Release_Notes_RevB", "ECOS_9.3.7.0_Release_Notes_RevB")`
2. Returns side-by-side comparison

---

## 📁 Project Structure

```
docsplorer/
├── README.md                    # This file
├── DESIGN.md                    # Comprehensive design document
├── INSTALL.md                   # Installation guide
├── DEPLOYMENT.md                # Deployment options
├── API_TESTING_GUIDE.md         # Testing methodology
├── TOOL_USAGE.md                # Tool usage guide
├── test_results/                # API test outputs (gitignored)
├── server.py                    # Main MCP server
├── config.py                    # Configuration management
├── requirements.txt             # Python dependencies
├── Dockerfile                   # Docker image
└── docker-compose.yml           # Docker compose config
```

---

## 🛠️ Tools

### Tool 1: `search_filenames_fuzzy`
**Purpose**: Find release note files using fuzzy text matching

**Parameters**:
- `query` (string, required): Search term
- `collection_name` (string, optional): Qdrant collection (default: "content")
- `limit` (integer, optional): Max results (default: 10, range: 1-100)

**Example**:
```python
search_filenames_fuzzy(
    query="ecos 9.3",
    limit=5
)
```

---

### Tool 2: `search_with_filename_filter`
**Purpose**: Search within a specific release note file with context

**Parameters**:
- `query` (string, required): What to search for

### Choose Your Deployment Method:

1. **uvx** (Recommended) - Like npx for Python
2. **FastMCP CLI** - Direct Python execution  
3. **Docker** - Isolated container

**See [DEPLOYMENT.md](DEPLOYMENT.md) for detailed setup instructions!**

### 30-Second Setup (uvx):

```bash
# 1. Navigate to docsplorer directory
cd /home/mir/projects/docsplorer

# 2. Install uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# 3. Configure
cp .env.example .env
# Edit .env with your API_URL and API_KEY

# 4. Add to IDE config (Windsurf/Claude Desktop)
# File: ~/.codeium/windsurf/mcp_config.json
# Use: /home/mir/projects/docsplorer/server.py
# (See INSTALL.md for detailed setup)

# 5. Restart your AI assistant
```

## Files

### Core Files
- `server.py` - Main MCP server with 6 tools (Phase 1 + Phase 2)
- `config.py` - Configuration management
- `requirements.txt` - Python dependencies
- `.env.example` - Configuration template

### Deployment Files
- `Dockerfile` - Docker image definition
- `docker-compose.yml` - Docker compose config

### Documentation
- `INSTALL.md` - **Installation guide for Windsurf** (includes IDE setup)
- `TOOL_USAGE.md` - **Comprehensive tool usage guide for LLMs**
- `DEPLOYMENT.md` - Deployment options and configuration
- `DESIGN.md` - Architecture and design details
- `README.md` - This file (consolidated quick reference)

## Related Documentation

- [DEPLOYMENT.md](DEPLOYMENT.md) - **Start here for setup!**
- [DESIGN.md](DESIGN.md) - Architecture and design
- [API Test Results](test_results/) - API testing documentation
- [FastAPI Server](../app/main.py) - Backend API
- [API Documentation](http://localhost:8001/docs) - Interactive API docs
- [FastMCP Documentation](https://github.com/jlowin/fastmcp) - MCP framework
- [MCP Protocol](https://modelcontextprotocol.io) - Protocol specification

---

## 📧 Support

- **Setup Issues**: See [DEPLOYMENT.md](DEPLOYMENT.md) troubleshooting section
- **Design Questions**: Check [DESIGN.md](DESIGN.md)
- **API Issues**: See [test_results/](test_results/) for API testing docs

---

**Last Updated**: November 12, 2025  
**Status**: ✅ Production Ready - All 3 deployment options available!

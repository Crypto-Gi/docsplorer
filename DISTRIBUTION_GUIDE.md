# Docsplorer Distribution Guide

## 📦 Overview

This guide covers two primary distribution methods for the Docsplorer MCP server:

1. **Docker Hub/GitHub Container Registry** - Container-based distribution
2. **npm Package** - Node.js package distribution (with Python wrapper)

Both are **standard practices** in the MCP ecosystem!

---

## 🐳 Option 1: Docker Distribution (Recommended for Production)

### Why Docker?
- ✅ **Zero dependency issues** - Everything bundled
- ✅ **Cross-platform** - Works on any OS
- ✅ **Easy updates** - `docker pull` to update
- ✅ **Isolation** - No conflicts with system packages
- ✅ **Standard practice** - Many MCP servers use Docker

### 1.1 Create Production Dockerfile

```dockerfile
# Dockerfile
FROM python:3.11-slim

# Set working directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements first for better caching
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY server.py config.py ./
COPY .env.example .env.example

# Create non-root user for security
RUN useradd -m -u 1000 docsplorer && \
    chown -R docsplorer:docsplorer /app
USER docsplorer

# Expose ports
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Default to stdio, but allow override
ENV TRANSPORT=stdio
ENV HOST=0.0.0.0
ENV PORT=8000

# Entry point
ENTRYPOINT ["python", "server.py"]
CMD ["--transport", "stdio"]
```

### 1.2 Create Multi-Stage Build (Optimized)

```dockerfile
# Dockerfile.optimized
FROM python:3.11-slim as builder

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

FROM python:3.11-slim

WORKDIR /app

# Copy only necessary files from builder
COPY --from=builder /root/.local /root/.local
COPY server.py config.py ./
COPY .env.example .env.example

# Make sure scripts in .local are usable
ENV PATH=/root/.local/bin:$PATH

# Create non-root user
RUN useradd -m -u 1000 docsplorer && \
    chown -R docsplorer:docsplorer /app
USER docsplorer

EXPOSE 8000

ENV TRANSPORT=stdio
ENV HOST=0.0.0.0
ENV PORT=8000

ENTRYPOINT ["python", "server.py"]
CMD ["--transport", "stdio"]
```

### 1.3 Docker Compose for Easy Deployment

```yaml
# docker-compose.yml
version: '3.8'

services:
  docsplorer:
    image: cryptogi/docsplorer:latest
    container_name: docsplorer-mcp
    ports:
      - "8000:8000"
    environment:
      - TRANSPORT=http
      - HOST=0.0.0.0
      - PORT=8000
      - API_URL=${API_URL:-http://localhost:8001}
      - API_KEY=${API_KEY}
      - QDRANT_COLLECTION=${QDRANT_COLLECTION:-content}
    env_file:
      - .env
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

### 1.4 Build and Push to Docker Hub

```bash
# Build for multiple platforms
docker buildx create --use
docker buildx build --platform linux/amd64,linux/arm64 \
  -t cryptogi/docsplorer:latest \
  -t cryptogi/docsplorer:v1.0.0 \
  --push .

# Or build locally
docker build -t cryptogi/docsplorer:latest .

# Push to Docker Hub
docker login
docker push cryptogi/docsplorer:latest
docker push cryptogi/docsplorer:v1.0.0
```

### 1.5 Push to GitHub Container Registry

```bash
# Login to GitHub Container Registry
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin

# Tag for GitHub
docker tag cryptogi/docsplorer:latest ghcr.io/crypto-gi/docsplorer:latest
docker tag cryptogi/docsplorer:v1.0.0 ghcr.io/crypto-gi/docsplorer:v1.0.0

# Push to GitHub Container Registry
docker push ghcr.io/crypto-gi/docsplorer:latest
docker push ghcr.io/crypto-gi/docsplorer:v1.0.0
```

### 1.6 User Installation (Docker)

Users can now easily run your MCP server:

```bash
# Pull from Docker Hub
docker pull cryptogi/docsplorer:latest

# Or from GitHub Container Registry
docker pull ghcr.io/crypto-gi/docsplorer:latest

# Run with stdio (for IDE integration)
docker run -it --rm \
  -e API_URL=http://localhost:8001 \
  -e API_KEY=your-key \
  cryptogi/docsplorer:latest

# Run with HTTP transport (for n8n)
docker run -d \
  -p 8000:8000 \
  -e TRANSPORT=http \
  -e API_URL=http://localhost:8001 \
  -e API_KEY=your-key \
  cryptogi/docsplorer:latest --transport http

# Or use docker-compose
curl -O https://raw.githubusercontent.com/Crypto-Gi/docsplorer/main/docker-compose.yml
docker-compose up -d
```

---

## 📦 Option 2: npm Package Distribution (Standard Practice!)

### Why npm for Python MCP Servers?
- ✅ **Standard in MCP ecosystem** - Most MCP servers use npm
- ✅ **Easy installation** - `npx -y @yourorg/mcp-server`
- ✅ **No manual setup** - Handles dependencies automatically
- ✅ **Version management** - Easy updates with npm
- ✅ **IDE integration** - Works seamlessly with Claude Desktop/Windsurf

### 2.1 Create npm Wrapper Package

```javascript
// package.json
{
  "name": "@cryptogi/docsplorer-mcp",
  "version": "1.0.0",
  "description": "Elite MCP server for semantic documentation search with precision citations",
  "main": "dist/index.js",
  "bin": {
    "docsplorer-mcp": "./bin/docsplorer-mcp.js"
  },
  "files": [
    "bin/",
    "dist/",
    "python/",
    "README.md",
    "LICENSE"
  ],
  "scripts": {
    "build": "tsc",
    "prepublishOnly": "npm run build",
    "postinstall": "node scripts/install-python-deps.js"
  },
  "keywords": [
    "mcp",
    "mcp-server",
    "documentation",
    "semantic-search",
    "qdrant",
    "rag",
    "claude",
    "ai-tools"
  ],
  "author": "Crypto-Gi",
  "license": "MIT",
  "repository": {
    "type": "git",
    "url": "https://github.com/Crypto-Gi/docsplorer.git"
  },
  "engines": {
    "node": ">=18.0.0"
  },
  "publishConfig": {
    "access": "public",
    "registry": "https://registry.npmjs.org"
  },
  "dependencies": {
    "cross-spawn": "^7.0.3"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "typescript": "^5.0.0"
  }
}
```

### 2.2 Create CLI Wrapper

```javascript
#!/usr/bin/env node
// bin/docsplorer-mcp.js

const { spawn } = require('cross-spawn');
const path = require('path');
const fs = require('fs');

// Get the directory where this package is installed
const packageDir = path.join(__dirname, '..');
const pythonDir = path.join(packageDir, 'python');

// Check if Python is available
function checkPython() {
  const pythonCommands = ['python3', 'python'];
  
  for (const cmd of pythonCommands) {
    try {
      const result = spawn.sync(cmd, ['--version'], { stdio: 'pipe' });
      if (result.status === 0) {
        return cmd;
      }
    } catch (e) {
      continue;
    }
  }
  
  console.error('Error: Python 3.8+ is required but not found.');
  console.error('Please install Python from https://www.python.org/downloads/');
  process.exit(1);
}

// Check if uvx is available (preferred)
function checkUvx() {
  try {
    const result = spawn.sync('uvx', ['--version'], { stdio: 'pipe' });
    return result.status === 0;
  } catch (e) {
    return false;
  }
}

// Main execution
function main() {
  const args = process.argv.slice(2);
  
  // Check for uvx first (recommended)
  if (checkUvx()) {
    console.log('Using uvx to run docsplorer...');
    const serverPath = path.join(pythonDir, 'server.py');
    const child = spawn('uvx', ['--from', 'fastmcp', 'fastmcp', 'run', serverPath, ...args], {
      stdio: 'inherit',
      env: process.env
    });
    
    child.on('exit', (code) => {
      process.exit(code || 0);
    });
  } else {
    // Fallback to direct Python execution
    const pythonCmd = checkPython();
    console.log(`Using ${pythonCmd} to run docsplorer...`);
    
    const serverPath = path.join(pythonDir, 'server.py');
    const child = spawn(pythonCmd, [serverPath, ...args], {
      stdio: 'inherit',
      env: process.env
    });
    
    child.on('exit', (code) => {
      process.exit(code || 0);
    });
  }
}

main();
```

### 2.3 Post-Install Script

```javascript
// scripts/install-python-deps.js
const { execSync } = require('child_process');
const path = require('path');
const fs = require('fs');

const pythonDir = path.join(__dirname, '..', 'python');
const requirementsPath = path.join(pythonDir, 'requirements.txt');

console.log('Installing Python dependencies...');

try {
  // Try uv first (fastest)
  try {
    execSync('uv --version', { stdio: 'ignore' });
    console.log('Using uv for fast Python dependency installation...');
    execSync(`uv pip install -r ${requirementsPath}`, { stdio: 'inherit' });
    console.log('✅ Python dependencies installed successfully with uv!');
    return;
  } catch (e) {
    // uv not available, try pip
  }
  
  // Fallback to pip
  const pythonCmd = process.platform === 'win32' ? 'python' : 'python3';
  execSync(`${pythonCmd} -m pip install -r ${requirementsPath}`, { stdio: 'inherit' });
  console.log('✅ Python dependencies installed successfully!');
} catch (error) {
  console.warn('⚠️  Could not install Python dependencies automatically.');
  console.warn('Please run manually: pip install -r python/requirements.txt');
}
```

### 2.4 TypeScript Definitions (Optional)

```typescript
// src/index.ts
export interface DocsplorerConfig {
  apiUrl: string;
  apiKey?: string;
  qdrantCollection?: string;
  transport?: 'stdio' | 'http' | 'https';
  host?: string;
  port?: number;
}

export function startDocsplorer(config: DocsplorerConfig): void;
```

### 2.5 Package Structure

```
docsplorer-npm/
├── package.json
├── README.md
├── LICENSE
├── bin/
│   └── docsplorer-mcp.js          # CLI entry point
├── scripts/
│   └── install-python-deps.js     # Post-install script
├── python/
│   ├── server.py                  # Your MCP server
│   ├── config.py
│   ├── requirements.txt
│   └── .env.example
├── src/
│   └── index.ts                   # TypeScript definitions
└── dist/
    └── index.js                   # Compiled JS
```

### 2.6 Publish to npm

```bash
# Login to npm
npm login

# Build the package
npm run build

# Publish (first time)
npm publish --access public

# Publish updates
npm version patch  # or minor, or major
npm publish
```

### 2.7 User Installation (npm)

Users can now install via npm/npx:

```bash
# Run directly with npx (no installation needed)
npx -y @cryptogi/docsplorer-mcp

# Or install globally
npm install -g @cryptogi/docsplorer-mcp
docsplorer-mcp

# Or install locally in project
npm install @cryptogi/docsplorer-mcp
npx docsplorer-mcp
```

### 2.8 IDE Configuration (npm)

```json
// ~/.codeium/windsurf/mcp_config.json
{
  "mcpServers": {
    "docsplorer": {
      "command": "npx",
      "args": ["-y", "@cryptogi/docsplorer-mcp@latest"],
      "env": {
        "API_URL": "http://localhost:8001",
        "API_KEY": "your-api-key",
        "QDRANT_COLLECTION": "content"
      }
    }
  }
}
```

---

## 🎯 Comparison: Docker vs npm

| Feature | Docker | npm |
|---------|--------|-----|
| **Installation** | `docker pull` | `npx -y` |
| **Dependencies** | All bundled | Requires Python |
| **Size** | ~200-500 MB | ~10-50 MB |
| **Startup Time** | 2-5 seconds | 1-2 seconds |
| **Updates** | `docker pull` | `npx -y @latest` |
| **IDE Integration** | Requires wrapper | Native support |
| **n8n Integration** | Native | Requires wrapper |
| **Cross-platform** | ✅ Perfect | ⚠️ Needs Python |
| **Standard Practice** | ✅ Yes | ✅ Yes (more common) |

---

## 🚀 Recommended Distribution Strategy

### For Maximum Reach: **Use Both!**

1. **npm Package** - Primary distribution method
   - Easiest for IDE users
   - Standard in MCP ecosystem
   - Quick installation with npx

2. **Docker Image** - For production/n8n
   - Best for HTTP/HTTPS transport
   - Perfect for n8n workflows
   - No dependency issues

### Distribution Checklist

#### npm Package
- [ ] Create package.json with proper metadata
- [ ] Add CLI wrapper script
- [ ] Include post-install for Python deps
- [ ] Test with `npx` locally
- [ ] Publish to npm registry
- [ ] Add npm badge to README

#### Docker Image
- [ ] Create optimized Dockerfile
- [ ] Build multi-platform images
- [ ] Push to Docker Hub
- [ ] Push to GitHub Container Registry
- [ ] Create docker-compose.yml
- [ ] Add Docker badge to README

#### Documentation
- [ ] Update README with both installation methods
- [ ] Add quick start guides
- [ ] Include configuration examples
- [ ] Document environment variables
- [ ] Add troubleshooting section

---

## 📚 Real-World Examples

### MCP Servers Using npm
- `@modelcontextprotocol/server-filesystem`
- `@modelcontextprotocol/server-github`
- `@stripe/mcp`
- `@upstash/context7-mcp`
- `@playwright/mcp`

### MCP Servers Using Docker
- Many custom enterprise MCP servers
- Production deployments
- n8n integrations
- Cloud-hosted MCP servers

---

## 🎓 Best Practices

### npm Package Best Practices
1. **Use scoped packages** - `@yourorg/package-name`
2. **Semantic versioning** - Follow semver strictly
3. **Include keywords** - "mcp", "mcp-server", "claude"
4. **Add repository link** - Link to GitHub
5. **Provide TypeScript types** - Even for Python servers
6. **Test with npx** - Ensure it works without installation

### Docker Best Practices
1. **Multi-stage builds** - Reduce image size
2. **Non-root user** - Security best practice
3. **Health checks** - Enable monitoring
4. **Multi-platform** - Support amd64 and arm64
5. **Version tags** - Tag with version numbers
6. **Environment variables** - Make it configurable

---

## 🔄 Continuous Deployment

### GitHub Actions for npm

```yaml
# .github/workflows/npm-publish.yml
name: Publish to npm

on:
  release:
    types: [created]

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          registry-url: 'https://registry.npmjs.org'
      - run: npm ci
      - run: npm run build
      - run: npm publish --access public
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

### GitHub Actions for Docker

```yaml
# .github/workflows/docker-publish.yml
name: Publish Docker Image

on:
  release:
    types: [created]

jobs:
  docker:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Login to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      
      - name: Login to GitHub Container Registry
        uses: docker/login-action@v2
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: |
            cryptogi/docsplorer
            ghcr.io/crypto-gi/docsplorer
      
      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```

---

## 📝 Updated README Badges

Add these badges to your README.md:

```markdown
[![npm version](https://badge.fury.io/js/@cryptogi%2Fdocsplorer-mcp.svg)](https://www.npmjs.com/package/@cryptogi/docsplorer-mcp)
[![Docker Pulls](https://img.shields.io/docker/pulls/cryptogi/docsplorer)](https://hub.docker.com/r/cryptogi/docsplorer)
[![GitHub Container Registry](https://img.shields.io/badge/ghcr.io-docsplorer-blue)](https://github.com/Crypto-Gi/docsplorer/pkgs/container/docsplorer)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
```

---

## ✅ Answer to Your Questions

### 1. Docker Container Distribution?
**YES! Absolutely recommended!** 

- ✅ Standard practice for production MCP servers
- ✅ Perfect for n8n integration
- ✅ Easy to maintain and update
- ✅ Works on Docker Hub and GitHub Container Registry

### 2. npm Package Distribution?
**YES! This is the MOST common practice!**

- ✅ **Standard in MCP ecosystem** - Most MCP servers use npm
- ✅ Easy installation with `npx -y @yourorg/mcp-server`
- ✅ Perfect for IDE integration (Claude Desktop, Windsurf)
- ✅ Even Python-based MCP servers use npm wrappers
- ✅ Examples: Stripe MCP, Context7, Playwright MCP all use npm

**Recommendation: Use BOTH for maximum reach!**
- npm for IDE users (primary)
- Docker for production/n8n (secondary)

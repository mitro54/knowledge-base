# Containerization

## Title & Summary

**Containerization** is the practice of packaging applications and their dependencies into standardized units called containers, enabling consistent deployment across different environments. This pattern covers Docker fundamentals, container images, layering, optimization techniques, and security best practices.

---

## Problem Statement

Traditional application deployment faces significant challenges:

- **Environment inconsistency** between development, staging, and production
- **Dependency conflicts** when multiple applications share a system
- **Resource inefficiency** from running full VMs for single applications
- **Slow deployment** due to environment setup and configuration
- **Reproducibility issues** when debugging production problems

### Example Scenario

```
Developer: "It works on my machine!"
Production: Application fails due to:
  - Different library versions
  - Missing environment variables
  - Different OS configurations
  - Database connection issues
```

---

## Solution

### Container Fundamentals

#### Container vs Virtual Machine

```
Virtual Machine Architecture:
┌─────────────────────────────────────────┐
│           Host Machine                  │
│  ┌───────────────────────────────────┐  │
│  │         Hypervisor                │  │
│  │  ┌─────────────┐  ┌─────────────┐ │  │
│  │  │   VM 1      │  │   VM 2      │ │  │
│  │  │ ┌────────┐  │  │ ┌────────┐  │ │  │
│  │  │ │  OS    │  │  │ │  OS    │  │ │  │
│  │  │ └────────┘  │  │ └────────┘  │ │  │
│  │  │ ┌────────┐  │  │ ┌────────┐  │ │  │
│  │  │ │ App 1  │  │  │ │ App 2  │  │ │  │
│  │  │ └────────┘  │  │ └────────┘  │ │  │
│  │  └─────────────┘  └─────────────┘ │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
Heavyweight: Each VM has full OS

Container Architecture:
┌─────────────────────────────────────────┐
│           Host Machine                  │
│  ┌───────────────────────────────────┐  │
│  │         Container Engine          │  │
│  │  ┌─────────────┐  ┌─────────────┐ │  │
│  │  │  Container  │  │  Container  │ │  │
│  │  │   App 1     │  │   App 2     │ │  │
│  │  └─────────────┘  └─────────────┘ │  │
│  └───────────────────────────────────┘  │
│           Host OS Kernel                │
└─────────────────────────────────────────┘
Lightweight: Containers share host kernel
```

#### Container Anatomy

```
┌─────────────────────────────────────────┐
│              Container                  │
│  ┌───────────────────────────────────┐  │
│  │         Application               │  │
│  │         Dependencies              │  │
│  │         Configuration             │  │
│  │         Runtime                   │  │
│  └───────────────────────────────────┘  │
│  ┌───────────────────────────────────┐  │
│  │         Container Runtime         │  │
│  │    (Docker, containerd, CRI-O)    │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│           Host Operating System         │
│           (Linux Kernel)                │
└─────────────────────────────────────────┘
```

### Docker Fundamentals

#### Dockerfile Best Practices

```dockerfile
# Multi-stage build for optimized image size
# Stage 1: Build stage
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files first (better caching)
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy source code
COPY . .

# Build application
RUN npm run build

# Stage 2: Production stage
FROM node:18-alpine AS production

# Add non-root user for security
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

WORKDIR /app

# Copy only built artifacts from builder
COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules

# Switch to non-root user
USER nodejs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD node -e "require('http').get('http://localhost:3000/health', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})"

# Start application
CMD ["node", "dist/server.js"]
```

#### Dockerfile Anti-Patterns vs Best Practices

```dockerfile
# ❌ Bad: Inefficient Dockerfile
FROM ubuntu:latest
RUN apt-get update && apt-get install -y nodejs npm
COPY . /app
WORKDIR /app
RUN npm install
RUN npm run build
CMD ["node", "server.js"]

# Problems:
# - Uses heavy base image
# - No layer caching optimization
# - Runs as root
# - No health check
# - Latest tag (non-deterministic)
# - All files copied before install

# ✅ Good: Optimized Dockerfile
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
USER node
EXPOSE 3000
HEALTHCHECK --interval=30s CMD curl -f http://localhost:3000/health || exit 1
CMD ["node", "dist/server.js"]
```

### Container Image Layers

#### Understanding Layer Caching

```
Layer Structure:
┌─────────────────────────────────────┐
│ Layer 5: CMD ["node", "server.js"]  │ ← 1KB
├─────────────────────────────────────┤
│ Layer 4: EXPOSE 3000                │ ← 1KB
├─────────────────────────────────────┤
│ Layer 3: COPY app/                  │ ← 50MB (changes often)
├─────────────────────────────────────┤
│ Layer 2: RUN npm install            │ ← 100MB (cacheable)
├─────────────────────────────────────┤
│ Layer 1: COPY package.json          │ ← 5KB (changes rarely)
├─────────────────────────────────────┤
│ Base Image: node:18-alpine          │ ← 150MB
└─────────────────────────────────────┘

Build Optimization:
- Change package.json → Rebuild layers 1-5
- Change app code → Rebuild only layer 3-5 (layers 1-2 cached!)
```

### Container Optimization

#### Image Size Comparison

```
Image Size Analysis:

Ubuntu-based:
FROM ubuntu:latest → 77MB
+ nodejs installation → +150MB
+ application → +50MB
Total: ~277MB

Alpine-based:
FROM alpine:latest → 5MB
+ nodejs installation → +45MB
+ application → +50MB
Total: ~100MB

Multi-stage build:
FROM node:18-alpine (builder) → 150MB
FROM node:18-alpine (runtime) → 150MB
+ only app artifacts → +10MB
Total: ~160MB (but no build tools in final image)

Distroless:
FROM gcr.io/distroless/nodejs18 → 130MB
+ application → +10MB
Total: ~140MB (minimal attack surface)
```

#### Docker Compose for Local Development

```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: development
    ports:
      - "3000:3000"
    volumes:
      - .:/app
      - /app/node_modules
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgresql://postgres:postgres@database:5432/app
    depends_on:
      - database
    networks:
      - app-network

  database:
    image: postgres:15-alpine
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_DB=app
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - app-network

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    networks:
      - app-network

volumes:
  postgres_data:

networks:
  app-network:
    driver: bridge
```

---

## When to Use

### Use Containers When:
- ✅ Need consistent environments across deployment stages
- ✅ Want to improve resource utilization
- ✅ Need fast startup times
- ✅ Building microservices architectures
- ✅ Implementing CI/CD pipelines

### Use Multi-stage Builds When:
- ✅ Want to reduce final image size
- ✅ Build tools not needed at runtime
- ✅ Security is a concern (smaller attack surface)

### Use Alpine Images When:
- ✅ Image size is critical
- ✅ Application is compatible with musl libc
- ✅ Willing to troubleshoot potential compatibility issues

---

## Tradeoffs

| Approach | Size | Security | Compatibility | Complexity |
|---------|------|----------|---------------|------------|
| **VM** | Large | High | Perfect | Medium |
| **Container (Ubuntu)** | Medium | Medium | Good | Low |
| **Container (Alpine)** | Small | Good | Fair | Medium |
| **Distroless** | Small | High | Fair | Medium |

### Quantitative Comparison

```
Resource Utilization:

Virtual Machine:
- Memory overhead: 500MB-2GB per VM
- Boot time: 30-60 seconds
- Disk: 10-20GB minimum
- CPU: Dedicated cores

Container:
- Memory overhead: 10-50MB per container
- Boot time: 1-3 seconds
- Disk: 100MB-500MB
- CPU: Shared, scheduled

Density Comparison:
- VM: ~10-20 per host
- Container: ~100-500 per host
```

---

## Implementation Example

### Production-Ready Docker Setup

```dockerfile
# Dockerfile for production Node.js application
FROM node:18-alpine AS dependencies

WORKDIR /app

# Install build dependencies for native modules
RUN apk add --no-cache python3 make g++

# Copy package files
COPY package*.json ./

# Install dependencies with cache
RUN npm ci --only=production && npm cache clean --force

FROM dependencies AS builder

# Copy source
COPY . .

# Run tests
RUN npm test

# Build application
RUN npm run build

FROM node:18-alpine AS production

# Security: Add non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

WORKDIR /app

# Copy only necessary files
COPY --from=dependencies --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist
COPY --chown=nodejs:nodejs /app/package.json ./

# Security: Minimal environment
ENV NODE_ENV=production \
    NODE_OPTIONS="--max-old-space-size=512"

# Switch to non-root user
USER nodejs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD node -e "require('http').get('http://localhost:3000/health', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})"

# Start application
CMD ["node", "dist/server.js"]
```

### Docker Build and Run Commands

```bash
# Build with cache
docker build -t myapp:latest .

# Build without cache (clean build)
docker build --no-cache -t myapp:latest .

# Build with specific target
docker build --target production -t myapp:latest .

# Run container
docker run -d \
  --name myapp \
  -p 3000:3000 \
  -e NODE_ENV=production \
  --restart unless-stopped \
  --memory=512m \
  --cpus=1 \
  myapp:latest

# Run with health check
docker run -d \
  --name myapp \
  -p 3000:3000 \
  --health-cmd="curl -f http://localhost:3000/health || exit 1" \
  --health-interval=30s \
  --health-timeout=3s \
  --health-retries=3 \
  myapp:latest
```

---

## Anti-Pattern

### ❌ Anti-Pattern: Running Database in Container

```yaml
# Bad: Storing database data in container
services:
  database:
    image: postgres:15
    environment:
      - POSTGRES_DB=app
    # Data lost when container is removed!
```

### ❌ Anti-Pattern: Running Container as Root

```dockerfile
# Bad: No user specified, runs as root
FROM node:18
COPY . /app
CMD ["node", "server.js"]
```

### ❌ Anti-Pattern: Using Latest Tag

```dockerfile
# Bad: Non-deterministic builds
FROM node:latest
FROM postgres:latest
```

### ❌ Anti-Pattern: Installing Dev Dependencies in Production

```dockerfile
# Bad: Unnecessary bloat
FROM node:18
COPY . /app
RUN npm install  # Installs ALL dependencies including dev
CMD ["node", "server.js"]
```

### ✅ Correct Approach

```yaml
# Good: Database with persistent volume
services:
  database:
    image: postgres:15-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=app

volumes:
  postgres_data:
```

```dockerfile
# Good: Non-root user, specific version, production deps only
FROM node:18-alpine
RUN adduser -D -u 1001 appuser
USER appuser
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
CMD ["node", "server.js"]
```

---

## Related Patterns

- **[Orchestration](02-Orchestration.md)** - Managing containers at scale with Kubernetes
- **[Infrastructure as Code](03-IaC.md)** - Declarative infrastructure definition
- **[CI/CD](../11-DevOps/01-CI-CD.md)** - Container builds in pipelines
- **[Service Mesh](04-Service-Mesh.md)** - Container-to-container communication
- **[Security Patterns](../05-Safety-Engineering/01-Security-Patterns.md)** - Container security
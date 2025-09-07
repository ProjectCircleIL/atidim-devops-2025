# Exercise 2: Dockerfile Keywords Mastery

**Duration:** 120 minutes  
**Difficulty:** Intermediate to Advanced  
**Type:** Hands-on Implementation + Deep Learning

## Objective
Master all essential Dockerfile keywords through practical implementation, understanding the purpose, syntax, and best practices for each directive. Build comprehensive knowledge of Docker image construction techniques.

## Prerequisites
- Completed Exercise 1 (Application Containerization)
- Docker Desktop installed and running
- Basic understanding of Linux commands
- Your application code from previous modules

## Docker Keywords to Master

We'll thoroughly exercise these critical Dockerfile keywords:
1. **FROM** - Base image selection
2. **RUN** - Execute commands during build
3. **CMD** - Default command execution
4. **ENTRYPOINT** - Container entry point configuration
5. **ENV** - Environment variables
6. **ARG** - Build-time arguments
7. **WORKDIR** - Working directory
8. **COPY/ADD** - File and directory operations
9. **EXPOSE** - Port documentation
10. **USER** - Security and permissions
11. **VOLUME** - Data persistence
12. **LABEL** - Metadata and documentation
13. **HEALTHCHECK** - Container health monitoring
14. **ONBUILD** - Trigger instructions for derived images
15. **SHELL** - Default shell configuration
16. **STOPSIGNAL** - Graceful shutdown configuration

## Tasks

### Task 1: Base Image Selection and Arguments (25 minutes)

**1.1 FROM - Base Image Strategies**

Create examples demonstrating different FROM usage patterns:

```dockerfile
# File: examples/dockerfile-examples/01-from-variations/Dockerfile.alpine
# Alpine-based image (smallest size)
FROM node:18-alpine3.17 AS alpine-base

# Install additional packages that might be needed
RUN apk add --no-cache \
    curl \
    git \
    python3 \
    make \
    g++

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

CMD ["node", "server.js"]
```

```dockerfile
# File: examples/dockerfile-examples/01-from-variations/Dockerfile.ubuntu
# Ubuntu-based image (more compatibility)
FROM node:18-bullseye AS ubuntu-base

# Install system packages
RUN apt-get update && apt-get install -y \
    curl \
    git \
    python3 \
    build-essential \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

CMD ["node", "server.js"]
```

```dockerfile
# File: examples/dockerfile-examples/01-from-variations/Dockerfile.scratch
# Scratch image (absolute minimal) - for Go binaries
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o main .

# Final stage from scratch
FROM scratch
COPY --from=builder /app/main /
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
ENTRYPOINT ["/main"]
```

**1.2 ARG - Build-time Arguments**

Create a Dockerfile demonstrating comprehensive ARG usage:

```dockerfile
# File: examples/dockerfile-examples/02-args-demo/Dockerfile
# Demonstrate all ARG features and best practices

# Global scope ARGs (available to all stages)
ARG BUILDPLATFORM
ARG TARGETPLATFORM
ARG BUILDOS
ARG TARGETARCH

# Application-specific build arguments
ARG NODE_VERSION=18
ARG APP_VERSION=1.0.0
ARG BUILD_DATE
ARG VCS_REF
ARG ENVIRONMENT=production

# Use ARG in FROM instruction
FROM node:${NODE_VERSION}-alpine AS base

# ARG values need to be redeclared in each stage where used
ARG APP_VERSION
ARG BUILD_DATE
ARG VCS_REF
ARG ENVIRONMENT

# Show build information
RUN echo "Building for platform: $TARGETPLATFORM" && \
    echo "Node version: $NODE_VERSION" && \
    echo "App version: $APP_VERSION" && \
    echo "Environment: $ENVIRONMENT"

# Install dependencies based on environment
RUN if [ "$ENVIRONMENT" = "development" ] ; then \
        apk add --no-cache git curl vim ; \
    else \
        apk add --no-cache curl ; \
    fi

# Use ARGs to set LABELs for metadata
LABEL maintainer="DevOps Team" \
      version="$APP_VERSION" \
      build-date="$BUILD_DATE" \
      vcs-ref="$VCS_REF" \
      environment="$ENVIRONMENT" \
      description="Sample application demonstrating ARG usage"

# Convert ARG to ENV for runtime availability (if needed)
ENV APP_VERSION=$APP_VERSION
ENV ENVIRONMENT=$ENVIRONMENT

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies based on environment
RUN if [ "$ENVIRONMENT" = "development" ] ; then \
        npm ci ; \
    else \
        npm ci --only=production && npm cache clean --force ; \
    fi

# Copy application code
COPY . .

# Default command
CMD ["npm", "start"]
```

Build script to demonstrate ARG usage:

```bash
#!/bin/bash
# File: examples/dockerfile-examples/02-args-demo/build.sh

echo "=== Building with different ARG values ==="

# Development build
echo "Building development image..."
docker build \
    --build-arg NODE_VERSION=18 \
    --build-arg APP_VERSION=1.0.0-dev \
    --build-arg BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
    --build-arg VCS_REF=$(git rev-parse --short HEAD) \
    --build-arg ENVIRONMENT=development \
    -t my-app:dev \
    -f Dockerfile .

# Production build
echo "Building production image..."
docker build \
    --build-arg NODE_VERSION=18 \
    --build-arg APP_VERSION=1.0.0 \
    --build-arg BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
    --build-arg VCS_REF=$(git rev-parse --short HEAD) \
    --build-arg ENVIRONMENT=production \
    -t my-app:prod \
    -f Dockerfile .

# Show the difference
echo "Comparing image metadata..."
docker inspect my-app:dev | jq '.[0].Config.Labels'
docker inspect my-app:prod | jq '.[0].Config.Labels'
```

### Task 2: Runtime Configuration - CMD vs ENTRYPOINT (20 minutes)

**2.1 Understanding CMD**

Create examples showing CMD variations:

```dockerfile
# File: examples/dockerfile-examples/03-cmd-variations/Dockerfile.cmd-forms
FROM node:18-alpine

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .

# CMD in exec form (recommended)
CMD ["node", "server.js"]

# Alternative CMD examples (commented out - only one CMD per Dockerfile)
# CMD ["npm", "start"]  # Using npm script
# CMD node server.js    # Shell form (creates shell process)
# CMD ["sh", "-c", "echo Starting application && node server.js"]  # With shell commands
```

```dockerfile
# File: examples/dockerfile-examples/03-cmd-variations/Dockerfile.cmd-override
FROM node:18-alpine

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .

# This CMD can be completely overridden at runtime
CMD ["node", "server.js"]

# Runtime examples:
# docker run my-app                           # Runs: node server.js
# docker run my-app npm test                  # Runs: npm test (overrides CMD)
# docker run my-app sh -c "ls -la"           # Runs: sh -c "ls -la" (overrides CMD)
```

**2.2 Understanding ENTRYPOINT**

Create examples showing ENTRYPOINT usage and combination with CMD:

```dockerfile
# File: examples/dockerfile-examples/04-entrypoint-variations/Dockerfile.entrypoint-fixed
FROM node:18-alpine

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .

# ENTRYPOINT cannot be overridden (without --entrypoint flag)
ENTRYPOINT ["node"]
CMD ["server.js"]

# Runtime examples:
# docker run my-app                           # Runs: node server.js
# docker run my-app app.js                    # Runs: node app.js (CMD overridden)
# docker run my-app --version                 # Runs: node --version (CMD overridden)
```

```dockerfile
# File: examples/dockerfile-examples/04-entrypoint-variations/Dockerfile.entrypoint-script
FROM node:18-alpine

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .

# Create entrypoint script for complex startup logic
COPY docker-entrypoint.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/docker-entrypoint.sh

ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["node", "server.js"]
```

Create the entrypoint script:

```bash
#!/bin/sh
# File: examples/dockerfile-examples/04-entrypoint-variations/docker-entrypoint.sh

set -e

# Function to handle shutdown gracefully
shutdown() {
    echo "Received shutdown signal, stopping application..."
    if [ -n "$APP_PID" ]; then
        kill -TERM "$APP_PID" 2>/dev/null || true
        wait "$APP_PID"
    fi
    exit 0
}

# Set up signal handlers
trap shutdown SIGTERM SIGINT

echo "Starting application with environment: ${NODE_ENV:-development}"

# Database connectivity check (if database URL provided)
if [ -n "$DATABASE_URL" ]; then
    echo "Checking database connectivity..."
    # Add database check logic here
    echo "Database check completed"
fi

# Initialize application data if needed
if [ "$1" = "init" ]; then
    echo "Initializing application..."
    # Add initialization logic here
    shift  # Remove 'init' from arguments
fi

# Start the application
echo "Starting: $@"
exec "$@" &
APP_PID=$!

# Wait for the application
wait $APP_PID
```

### Task 3: Environment Management - ENV (15 minutes)

**3.1 ENV - Environment Variables**

```dockerfile
# File: examples/dockerfile-examples/05-env-management/Dockerfile
FROM node:18-alpine

# Set environment variables for application
ENV NODE_ENV=production \
    PORT=3000 \
    APP_NAME="DevOps Learning App" \
    LOG_LEVEL=info

# Environment variables for paths
ENV APP_HOME=/app \
    USER_HOME=/home/appuser

# Create application user and directory
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001 -G nodejs -h $USER_HOME

# Use environment variable in WORKDIR
WORKDIR $APP_HOME

# Copy package files
COPY package*.json ./

# Use environment variable in RUN command
RUN npm ci --only=production && \
    chown -R nodejs:nodejs $APP_HOME

# Copy application code
COPY . .
RUN chown -R nodejs:nodejs $APP_HOME

# Switch to non-root user
USER nodejs

# Environment variables are available at runtime
# Can be overridden with docker run -e or --env-file

EXPOSE $PORT

# Use environment variable in CMD
CMD ["sh", "-c", "echo Starting $APP_NAME on port $PORT && exec node server.js"]
```

Create an environment file example:

```bash
# File: examples/dockerfile-examples/05-env-management/.env.example
NODE_ENV=production
PORT=3000
APP_NAME=DevOps Learning App
LOG_LEVEL=info
DATABASE_URL=postgres://user:pass@localhost:5432/appdb
JWT_SECRET=your-secret-key-here
REDIS_URL=redis://localhost:6379
```

### Task 4: File Operations - COPY vs ADD (15 minutes)

**4.1 COPY - Simple File Operations**

```dockerfile
# File: examples/dockerfile-examples/06-file-operations/Dockerfile.copy
FROM node:18-alpine

WORKDIR /app

# COPY specific files
COPY package.json ./
COPY package-lock.json ./

# COPY directories
COPY src/ ./src/
COPY public/ ./public/

# COPY with ownership (available in newer Docker versions)
COPY --chown=node:node . .

# COPY with permissions
RUN chmod +x scripts/*.sh

# Multiple COPY commands vs single COPY
# Single COPY is more efficient for related files
COPY package*.json ./
# vs
# COPY package.json ./
# COPY package-lock.json ./
```

**4.2 ADD - Advanced File Operations**

```dockerfile
# File: examples/dockerfile-examples/06-file-operations/Dockerfile.add
FROM ubuntu:20.04

WORKDIR /app

# ADD can download from URLs
ADD https://github.com/golang/go/archive/refs/tags/go1.21.0.tar.gz /tmp/

# ADD can extract tar archives automatically
ADD app.tar.gz ./

# ADD vs COPY best practices:
# Use COPY for local files (preferred)
COPY local-file.txt ./

# Use ADD only when you need URL download or extraction
ADD https://releases.example.com/app.tar.gz /opt/

# Example: Setting up a downloaded archive
ADD --chown=1000:1000 https://nodejs.org/dist/v18.17.0/node-v18.17.0-linux-x64.tar.xz /tmp/
RUN cd /tmp && \
    tar -xJf node-*.tar.xz && \
    mv node-*/ /opt/node && \
    ln -s /opt/node/bin/* /usr/local/bin/ && \
    rm -rf /tmp/node-*.tar.xz
```

### Task 5: Advanced Configuration - USER, VOLUME, HEALTHCHECK (20 minutes)

**5.1 USER - Security Best Practices**

```dockerfile
# File: examples/dockerfile-examples/07-user-security/Dockerfile
FROM node:18-alpine

# Create user and group with specific IDs
RUN addgroup -g 1001 -S appgroup && \
    adduser -S appuser -u 1001 -G appgroup

# Alternative: using groupadd/useradd (in Ubuntu/Debian images)
# RUN groupadd -r appgroup --gid=1001 && \
#     useradd -r -g appgroup --uid=1001 --home-dir=/app --shell=/bin/bash appuser

# Set up directories with proper ownership
WORKDIR /app
RUN chown -R appuser:appgroup /app

# Install dependencies as root (if needed)
COPY package*.json ./
RUN npm ci --only=production && \
    chown -R appuser:appgroup /app

# Copy application code
COPY . .
RUN chown -R appuser:appgroup /app

# Switch to non-root user for security
USER appuser

# From this point, all commands run as appuser
EXPOSE 3000

CMD ["node", "server.js"]

# Note: If you need to run commands as root after USER directive:
# USER root
# RUN some-root-command
# USER appuser
```

**5.2 VOLUME - Data Persistence**

```dockerfile
# File: examples/dockerfile-examples/08-volumes/Dockerfile
FROM node:18-alpine

WORKDIR /app

# Create directories for volumes
RUN mkdir -p /app/data /app/logs /app/uploads && \
    chown -R node:node /app

# Declare volumes for data that should persist
VOLUME ["/app/data"]
VOLUME ["/app/logs"]
VOLUME ["/app/uploads"]

# Alternative single VOLUME declaration
# VOLUME /app/data /app/logs /app/uploads

COPY package*.json ./
RUN npm ci --only=production

COPY . .

# Set permissions for volume directories
RUN chown -R node:node /app/data /app/logs /app/uploads

USER node

# Application will write to these volume-mounted directories
CMD ["node", "server.js"]

# Runtime volume mounting examples:
# docker run -v $(pwd)/data:/app/data my-app
# docker run -v my-data-volume:/app/data my-app
# docker run --mount type=volume,source=my-data,target=/app/data my-app
```

**5.3 HEALTHCHECK - Container Health Monitoring**

```dockerfile
# File: examples/dockerfile-examples/09-healthcheck/Dockerfile
FROM node:18-alpine

# Install curl for health checks
RUN apk add --no-cache curl

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .

EXPOSE 3000

# Basic HTTP health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:3000/health || exit 1

# Alternative health check using wget
# HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
#     CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

# Custom health check script
COPY healthcheck.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/healthcheck.sh

# Advanced health check with custom script
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD /usr/local/bin/healthcheck.sh

# Disable inherited health check (if extending an image with health check)
# HEALTHCHECK NONE

USER node

CMD ["node", "server.js"]
```

Create the custom health check script:

```bash
#!/bin/sh
# File: examples/dockerfile-examples/09-healthcheck/healthcheck.sh

# Custom health check script with multiple validations

# Check if main port is responding
if ! curl -f http://localhost:3000/health >/dev/null 2>&1; then
    echo "Main health endpoint not responding"
    exit 1
fi

# Check if critical services are available
if ! curl -f http://localhost:3000/api/status >/dev/null 2>&1; then
    echo "API status endpoint not responding"
    exit 1
fi

# Check disk space (optional)
DISK_USAGE=$(df /app | tail -1 | awk '{print $5}' | sed 's/%//')
if [ "$DISK_USAGE" -gt 95 ]; then
    echo "Disk usage too high: ${DISK_USAGE}%"
    exit 1
fi

# All checks passed
echo "All health checks passed"
exit 0
```

### Task 6: Advanced Keywords - LABEL, SHELL, STOPSIGNAL, ONBUILD (25 minutes)

**6.1 LABEL - Metadata and Documentation**

```dockerfile
# File: examples/dockerfile-examples/10-labels/Dockerfile
FROM node:18-alpine

# Standard OCI labels
LABEL org.opencontainers.image.title="DevOps Learning Application"
LABEL org.opencontainers.image.description="A sample 3-tier application for learning DevOps practices"
LABEL org.opencontainers.image.version="1.0.0"
LABEL org.opencontainers.image.authors="DevOps Team <devops@example.com>"
LABEL org.opencontainers.image.url="https://github.com/example/devops-learning-app"
LABEL org.opencontainers.image.source="https://github.com/example/devops-learning-app"
LABEL org.opencontainers.image.documentation="https://docs.example.com/devops-app"
LABEL org.opencontainers.image.created="2024-01-15T10:30:00Z"
LABEL org.opencontainers.image.revision="abc123def456"
LABEL org.opencontainers.image.licenses="MIT"

# Custom application labels
LABEL application.name="devops-learning-app"
LABEL application.component="backend"
LABEL application.part-of="devops-bootcamp"
LABEL application.managed-by="kubernetes"
LABEL application.instance="production"

# Team and ownership labels
LABEL team="platform"
LABEL owner="devops-team"
LABEL environment="production"
LABEL cost-center="engineering"

# Technical labels
LABEL dockerfile.version="2.0"
LABEL base.image="node:18-alpine"
LABEL security.scan="enabled"
LABEL monitoring.metrics="prometheus"

# Build information labels (typically set via build args)
ARG BUILD_DATE
ARG VCS_REF
ARG VERSION

LABEL build.date=$BUILD_DATE
LABEL build.vcs-ref=$VCS_REF
LABEL build.version=$VERSION

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .

CMD ["node", "server.js"]
```

**6.2 SHELL - Default Shell Configuration**

```dockerfile
# File: examples/dockerfile-examples/11-shell/Dockerfile
FROM ubuntu:20.04

# Default shell is ["/bin/sh", "-c"] on Linux
# Change default shell to bash with error handling
SHELL ["/bin/bash", "-euxo", "pipefail", "-c"]

# Now all RUN commands use bash with strict error handling
RUN echo "Using bash with strict error handling"

# Install packages with error handling
RUN apt-get update && \
    apt-get install -y \
        curl \
        nodejs \
        npm && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Example of PowerShell on Windows containers
# SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop';"]

# Reset shell back to default if needed
SHELL ["/bin/sh", "-c"]

RUN echo "Back to default shell"

WORKDIR /app
COPY . .

CMD ["node", "server.js"]
```

**6.3 STOPSIGNAL - Graceful Shutdown**

```dockerfile
# File: examples/dockerfile-examples/12-stopsignal/Dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production
COPY . .

# Set the signal used to stop the container
# Default is SIGTERM, but some applications handle different signals better
STOPSIGNAL SIGTERM

# Alternative signals:
# STOPSIGNAL SIGINT   # For applications that handle Ctrl+C
# STOPSIGNAL SIGUSR1  # Custom signal handling
# STOPSIGNAL 15       # Numeric signal (SIGTERM = 15)

# Create graceful shutdown handler
COPY graceful-shutdown.js ./

EXPOSE 3000

CMD ["node", "graceful-shutdown.js"]
```

Create graceful shutdown handler:

```javascript
// File: examples/dockerfile-examples/12-stopsignal/graceful-shutdown.js
const http = require('http');

// Create HTTP server
const server = http.createServer((req, res) => {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({
        message: 'Hello from DevOps learning app!',
        timestamp: new Date().toISOString(),
        pid: process.pid
    }));
});

// Graceful shutdown handler
let isShuttingDown = false;

const gracefulShutdown = (signal) => {
    if (isShuttingDown) return;
    isShuttingDown = true;
    
    console.log(`Received ${signal}. Starting graceful shutdown...`);
    
    // Stop accepting new connections
    server.close((err) => {
        if (err) {
            console.error('Error during server shutdown:', err);
            process.exit(1);
        }
        
        console.log('HTTP server closed');
        
        // Perform cleanup tasks here
        // - Close database connections
        // - Finish processing requests
        // - Save application state
        
        console.log('Graceful shutdown completed');
        process.exit(0);
    });
    
    // Force shutdown after timeout
    setTimeout(() => {
        console.error('Forced shutdown after timeout');
        process.exit(1);
    }, 30000); // 30 second timeout
};

// Handle shutdown signals
process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));

// Handle uncaught exceptions
process.on('uncaughtException', (err) => {
    console.error('Uncaught Exception:', err);
    gracefulShutdown('UNCAUGHT_EXCEPTION');
});

process.on('unhandledRejection', (reason, promise) => {
    console.error('Unhandled Rejection at:', promise, 'reason:', reason);
    gracefulShutdown('UNHANDLED_REJECTION');
});

// Start server
const PORT = process.env.PORT || 3000;
server.listen(PORT, () => {
    console.log(`Server running on port ${PORT}, PID: ${process.pid}`);
});
```

**6.4 ONBUILD - Trigger Instructions**

```dockerfile
# File: examples/dockerfile-examples/13-onbuild/Dockerfile.base
# Base image for Node.js applications with ONBUILD triggers
FROM node:18-alpine

# Install common dependencies
RUN apk add --no-cache curl git

# Set up common directory structure
WORKDIR /app
RUN mkdir -p logs data uploads

# Create application user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001 -G nodejs

# ONBUILD instructions execute when this image is used as a base
ONBUILD COPY package*.json ./
ONBUILD RUN npm ci --only=production
ONBUILD COPY . .
ONBUILD RUN chown -R nodejs:nodejs /app
ONBUILD USER nodejs

# Default configuration
EXPOSE 3000
CMD ["node", "server.js"]
```

```dockerfile
# File: examples/dockerfile-examples/13-onbuild/Dockerfile.derived
# Child image that uses the base with ONBUILD
FROM my-node-base:latest

# The ONBUILD instructions from the base image are executed here automatically:
# 1. ONBUILD COPY package*.json ./
# 2. ONBUILD RUN npm ci --only=production  
# 3. ONBUILD COPY . .
# 4. ONBUILD RUN chown -R nodejs:nodejs /app
# 5. ONBUILD USER nodejs

# Add any additional configuration specific to this application
ENV NODE_ENV=production
ENV LOG_LEVEL=info

# The CMD from base image is inherited unless overridden
# CMD ["node", "server.js"]
```

## Deliverables

1. **Comprehensive Dockerfile Examples:**
   - Individual files demonstrating each keyword
   - Practical examples with real-world use cases
   - Before/after comparisons showing best practices

2. **Build and Test Scripts:**
   - Scripts to build and test each example
   - Validation commands for each keyword
   - Comparison tools to see differences

3. **Documentation:**
   - Summary of when to use each keyword
   - Best practices and common pitfalls
   - Performance implications of different approaches

4. **Working Examples:**
   - All Dockerfiles build successfully
   - Each keyword demonstrated with practical purpose
   - Integration with your application from previous modules

## Success Criteria

- Successfully built Docker images using all specified keywords
- Demonstrated understanding of when and how to use each directive
- Created secure, efficient, and maintainable Dockerfiles
- Integrated keywords appropriately for production use
- Documented best practices and common patterns

## Verification Commands

```bash
# Build examples
for dir in examples/dockerfile-examples/*/; do
    echo "Building $dir"
    docker build -t "example-$(basename $dir)" "$dir"
done

# Test ARG functionality
cd examples/dockerfile-examples/02-args-demo
./build.sh

# Verify labels
docker inspect my-app:prod | jq '.[0].Config.Labels'

# Test health checks
docker run -d --name health-test my-app:latest
sleep 30
docker inspect health-test | jq '.[0].State.Health'

# Test graceful shutdown
docker run -d --name shutdown-test my-app:latest
docker stop shutdown-test  # Should shutdown gracefully
docker logs shutdown-test

# Verify user security
docker run --rm my-app:latest whoami
docker run --rm my-app:latest id
```

## Integration with Next Exercise

These Dockerfile skills prepare for:
- Multi-stage builds for optimization
- Docker Compose integration
- Production deployment strategies
- Container security hardening
- CI/CD pipeline integration

## Best Practices Summary

1. **Order Instructions by Frequency of Change** (least to most)
2. **Use .dockerignore** to exclude unnecessary files
3. **Minimize Layers** by combining RUN commands
4. **Run as Non-Root User** for security
5. **Use Specific Tags** instead of 'latest'
6. **Set Health Checks** for production containers
7. **Handle Signals Gracefully** for proper shutdown
8. **Add Proper Labels** for documentation and automation
9. **Use Multi-Stage Builds** for smaller final images
10. **Cache Dependencies** by copying package files first

This comprehensive exercise ensures students master every essential aspect of Dockerfile creation and container configuration!
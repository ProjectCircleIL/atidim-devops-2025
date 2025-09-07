# Exercise 3: Multi-Stage Builds Advanced Techniques

**Duration:** 90 minutes  
**Difficulty:** Advanced  
**Type:** Hands-on Implementation + Optimization

## Objective
Master advanced multi-stage build techniques to create highly optimized, secure, and efficient Docker images. Learn complex build patterns, cross-compilation, and sophisticated caching strategies.

## Prerequisites
- Completed Exercise 2 (Dockerfile Keywords Mastery)
- Understanding of Docker build context and layers
- Knowledge of your application's build requirements
- Familiarity with different base image types

## Multi-Stage Build Concepts

### Advanced Patterns to Master:
1. **Build-Test-Deploy Pipeline**: Separate stages for building, testing, and final deployment
2. **Cross-Platform Compilation**: Building for different architectures
3. **Dependency Caching**: Optimizing build speed with smart caching
4. **Security Scanning Integration**: Vulnerability scanning in build pipeline  
5. **Asset Optimization**: Minification and compression in build stages
6. **Multi-Target Builds**: Different targets for different environments
7. **Build Tool Isolation**: Separating build tools from runtime
8. **Secret Management**: Handling secrets securely during build

## Tasks

### Task 1: Complex Multi-Stage Frontend Build (25 minutes)

**1.1 Advanced React/Vue.js Build Pipeline**

Create a comprehensive frontend build with multiple stages:

```dockerfile
# File: frontend/Dockerfile.multistage
# Advanced multi-stage build for React/Vue.js application

#################################
# Stage 1: Base dependencies
#################################
FROM node:18-alpine AS base

# Install system dependencies needed for native modules
RUN apk add --no-cache \
    libc6-compat \
    python3 \
    make \
    g++

# Set working directory
WORKDIR /app

# Copy package files for dependency caching
COPY package*.json ./
COPY yarn.lock* ./

#################################
# Stage 2: Development dependencies
#################################
FROM base AS deps

# Install all dependencies (including dev dependencies)
RUN npm ci

#################################
# Stage 3: Build stage with testing
#################################
FROM base AS builder

# Copy dependencies from deps stage
COPY --from=deps /app/node_modules ./node_modules

# Copy source code
COPY . .

# Set build-time environment variables
ARG NODE_ENV=production
ARG REACT_APP_API_URL
ARG REACT_APP_VERSION
ARG BUILD_DATE
ARG GIT_COMMIT

ENV NODE_ENV=$NODE_ENV
ENV REACT_APP_API_URL=$REACT_APP_API_URL
ENV REACT_APP_VERSION=$REACT_APP_VERSION
ENV REACT_APP_BUILD_DATE=$BUILD_DATE
ENV REACT_APP_GIT_COMMIT=$GIT_COMMIT

# Run linting and type checking
RUN npm run lint
RUN npm run type-check

# Run unit tests
RUN npm test -- --coverage --watchAll=false

# Build the application
RUN npm run build

# Verify build output
RUN ls -la build/

#################################
# Stage 4: Asset optimization
#################################
FROM node:18-alpine AS optimizer

WORKDIR /app

# Install optimization tools
RUN npm install -g \
    imagemin-cli \
    svgo

# Copy built assets
COPY --from=builder /app/build ./build

# Optimize images
RUN find build -name "*.png" -exec imagemin {} --out-dir=build/optimized \;
RUN find build -name "*.jpg" -exec imagemin {} --out-dir=build/optimized \;
RUN find build -name "*.svg" -exec svgo {} --output=build/optimized/{} \;

# Optimize CSS and JS (if not already done by build tool)
RUN find build -name "*.css" -exec npx clean-css-cli {} -o build/optimized/{} \;

#################################
# Stage 5: Security scanning
#################################
FROM aquasec/trivy:latest AS security-scan

# Copy the built application for vulnerability scanning
COPY --from=builder /app /scan-target

# Run security scan (in CI, this would fail the build if issues found)
RUN trivy fs --exit-code 0 --no-progress --format table /scan-target

#################################
# Stage 6: Production runtime (Nginx)
#################################
FROM nginx:alpine AS production

# Install additional tools for health checks
RUN apk add --no-cache curl

# Copy optimized build assets
COPY --from=optimizer /app/build/optimized /usr/share/nginx/html
COPY --from=builder /app/build /usr/share/nginx/html

# Copy custom Nginx configuration
COPY nginx/nginx.conf /etc/nginx/nginx.conf
COPY nginx/default.conf /etc/nginx/conf.d/default.conf

# Add build information as environment variables
ARG BUILD_DATE
ARG GIT_COMMIT
ARG VERSION

ENV BUILD_DATE=$BUILD_DATE
ENV GIT_COMMIT=$GIT_COMMIT
ENV VERSION=$VERSION

# Create health check endpoint
RUN echo '{"status":"healthy","buildDate":"'$BUILD_DATE'","version":"'$VERSION'","commit":"'$GIT_COMMIT'"}' > /usr/share/nginx/html/health.json

# Set up proper permissions
RUN chown -R nginx:nginx /usr/share/nginx/html && \
    chmod -R 755 /usr/share/nginx/html

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:80/health.json || exit 1

# Use non-root user
USER nginx

EXPOSE 80

#################################
# Stage 7: Development runtime
#################################
FROM base AS development

# Install development dependencies
COPY --from=deps /app/node_modules ./node_modules

# Copy source code
COPY . .

# Development environment variables
ENV NODE_ENV=development
ENV CHOKIDAR_USEPOLLING=true

EXPOSE 3000

CMD ["npm", "start"]

#################################
# Stage 8: Testing runtime
#################################
FROM base AS testing

# Install all dependencies including test dependencies
COPY --from=deps /app/node_modules ./node_modules

# Copy source code
COPY . .

# Set testing environment
ENV NODE_ENV=test
ENV CI=true

# Run tests and generate coverage
CMD ["npm", "run", "test:ci"]
```

**1.2 Advanced Build Script with Multi-Target Support**

```bash
#!/bin/bash
# File: frontend/build-multistage.sh

set -e

# Build configuration
APP_NAME="devops-frontend"
VERSION=${VERSION:-"1.0.0"}
BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ')
GIT_COMMIT=$(git rev-parse --short HEAD)
REACT_APP_API_URL=${REACT_APP_API_URL:-"http://localhost:3001"}

# Build arguments
BUILD_ARGS="
--build-arg BUILD_DATE=$BUILD_DATE
--build-arg GIT_COMMIT=$GIT_COMMIT
--build-arg VERSION=$VERSION
--build-arg REACT_APP_API_URL=$REACT_APP_API_URL
--build-arg REACT_APP_VERSION=$VERSION
"

echo "=== Building Multi-Stage Frontend Application ==="
echo "Version: $VERSION"
echo "Build Date: $BUILD_DATE"
echo "Git Commit: $GIT_COMMIT"
echo "API URL: $REACT_APP_API_URL"

# Development build
echo "Building development image..."
docker build \
    $BUILD_ARGS \
    --target development \
    -t $APP_NAME:dev \
    -f Dockerfile.multistage .

# Testing build
echo "Building testing image..."  
docker build \
    $BUILD_ARGS \
    --target testing \
    -t $APP_NAME:test \
    -f Dockerfile.multistage .

# Production build
echo "Building production image..."
docker build \
    $BUILD_ARGS \
    --target production \
    -t $APP_NAME:latest \
    -t $APP_NAME:$VERSION \
    -f Dockerfile.multistage .

# Display image sizes
echo "=== Image Sizes ==="
docker images | grep $APP_NAME

# Run tests
echo "=== Running Tests ==="
docker run --rm $APP_NAME:test

# Verify production build
echo "=== Verifying Production Build ==="
docker run -d --name temp-frontend -p 8080:80 $APP_NAME:latest
sleep 5

# Test health endpoint
curl -f http://localhost:8080/health.json || echo "Health check failed"

# Test main application
curl -f http://localhost:8080/ || echo "App not responding"

# Cleanup
docker stop temp-frontend
docker rm temp-frontend

echo "Build completed successfully!"
```

### Task 2: Complex Backend Multi-Stage Build (25 minutes)

**2.1 Advanced Node.js/Python Backend Build**

```dockerfile
# File: backend/Dockerfile.multistage
# Advanced multi-stage build for Node.js backend with comprehensive tooling

#################################
# Stage 1: Base image with system dependencies  
#################################
FROM node:18-alpine AS base

# Install system dependencies
RUN apk add --no-cache \
    dumb-init \
    curl \
    ca-certificates \
    tzdata \
    && update-ca-certificates

# Create app directory and user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001 -G nodejs

WORKDIR /app

# Copy package files for better caching
COPY package*.json ./
COPY yarn.lock* ./

#################################
# Stage 2: Development dependencies
#################################
FROM base AS deps-dev

# Install all dependencies
RUN npm ci

#################################
# Stage 3: Production dependencies
#################################
FROM base AS deps-prod

# Install only production dependencies
RUN npm ci --only=production && \
    npm cache clean --force

#################################
# Stage 4: Build and compile stage
#################################
FROM base AS builder

# Copy dev dependencies
COPY --from=deps-dev /app/node_modules ./node_modules

# Copy source code
COPY . .

# Build arguments for compilation
ARG NODE_ENV=production
ARG API_VERSION=v1
ARG BUILD_DATE
ARG GIT_COMMIT

ENV NODE_ENV=$NODE_ENV

# Run linting and type checking
RUN npm run lint
RUN npm run type-check

# Run unit tests with coverage
RUN npm test -- --coverage

# Build/compile application (if using TypeScript, Babel, etc.)
RUN npm run build

# Verify build
RUN ls -la dist/

#################################
# Stage 5: Security and vulnerability scanning
#################################
FROM aquasec/trivy:latest AS security

# Copy node modules for scanning
COPY --from=deps-prod /app/node_modules /scan/node_modules
COPY --from=builder /app /scan/app

# Scan for vulnerabilities
RUN trivy fs --exit-code 0 --no-progress /scan

#################################
# Stage 6: Integration testing
#################################
FROM base AS integration-test

# Copy production dependencies
COPY --from=deps-dev /app/node_modules ./node_modules

# Copy built application
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package*.json ./

# Copy test files
COPY tests/ ./tests/
COPY jest.config.js ./

# Set test environment
ENV NODE_ENV=test
ENV DATABASE_URL=sqlite:///:memory:

# Run integration tests
RUN npm run test:integration

#################################
# Stage 7: Production runtime
#################################
FROM base AS production

# Copy production dependencies
COPY --from=deps-prod /app/node_modules ./node_modules

# Copy built application
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package*.json ./

# Copy production configuration
COPY --from=builder /app/config ./config

# Set production environment
ENV NODE_ENV=production
ENV PORT=3000

# Add build information
ARG BUILD_DATE
ARG GIT_COMMIT
ARG VERSION

LABEL build.date=$BUILD_DATE \
      build.version=$VERSION \
      build.commit=$GIT_COMMIT

# Create health check script
RUN echo '#!/bin/sh\ncurl -f http://localhost:$PORT/health || exit 1' > /usr/local/bin/healthcheck && \
    chmod +x /usr/local/bin/healthcheck

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD /usr/local/bin/healthcheck

# Switch to non-root user
USER nodejs

EXPOSE $PORT

# Use dumb-init for proper signal handling
ENTRYPOINT ["dumb-init", "--"]

CMD ["node", "dist/server.js"]

#################################
# Stage 8: Debug/Development runtime
#################################
FROM base AS debug

# Copy all dependencies (including dev)
COPY --from=deps-dev /app/node_modules ./node_modules

# Copy source code (not built)
COPY . .

# Install debugging tools
RUN npm install -g nodemon

# Development environment
ENV NODE_ENV=development
ENV DEBUG=*

EXPOSE 3000 9229

# Start with nodemon and debugging enabled
CMD ["nodemon", "--inspect=0.0.0.0:9229", "src/server.js"]
```

### Task 3: Cross-Platform and Architecture Builds (20 minutes)

**3.1 Multi-Architecture Build with BuildKit**

```dockerfile
# File: Dockerfile.multiarch
# Multi-architecture build supporting AMD64, ARM64, and ARM/v7

#################################
# Stage 1: Base stage with architecture detection
#################################
FROM --platform=$BUILDPLATFORM node:18-alpine AS base

# Build arguments for cross-compilation
ARG BUILDPLATFORM
ARG TARGETPLATFORM
ARG BUILDOS
ARG BUILDARCH
ARG TARGETARCH
ARG TARGETVARIANT

# Display build information
RUN echo "Building on $BUILDPLATFORM, targeting $TARGETPLATFORM" && \
    echo "Build arch: $BUILDARCH, Target arch: $TARGETARCH"

WORKDIR /app

# Install architecture-specific dependencies
RUN case "$TARGETARCH" in \
        amd64) ARCH_DEPS="libc6-compat" ;; \
        arm64) ARCH_DEPS="libc6-compat" ;; \
        arm) ARCH_DEPS="libc6-compat" ;; \
        *) echo "Unsupported architecture: $TARGETARCH" && exit 1 ;; \
    esac && \
    apk add --no-cache $ARCH_DEPS curl ca-certificates

#################################
# Stage 2: Native builder for cross-compilation
#################################
FROM --platform=$BUILDPLATFORM node:18-alpine AS builder

ARG TARGETARCH
ARG TARGETOS

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies with cross-compilation support
RUN npm ci

# Copy source code
COPY . .

# Build for target architecture
ENV TARGET_ARCH=$TARGETARCH
ENV TARGET_OS=$TARGETOS

# Run architecture-specific build if needed
RUN npm run build

#################################
# Stage 3: Runtime stage for target platform
#################################
FROM base AS runtime

# Copy built application from builder
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package*.json ./

# Create non-root user
RUN addgroup -g 1001 -S appgroup && \
    adduser -S appuser -u 1001 -G appgroup

USER appuser

EXPOSE 3000

CMD ["node", "dist/server.js"]
```

**3.2 Build Script for Multi-Architecture**

```bash
#!/bin/bash
# File: build-multiarch.sh

set -e

APP_NAME="devops-backend"
VERSION=${VERSION:-"1.0.0"}

echo "=== Building Multi-Architecture Images ==="

# Enable Docker BuildKit
export DOCKER_BUILDKIT=1

# Create builder instance for multi-platform builds
docker buildx create --name multiarch-builder --use --bootstrap || true

# Build for multiple platforms
docker buildx build \
    --platform linux/amd64,linux/arm64,linux/arm/v7 \
    --build-arg VERSION=$VERSION \
    --build-arg BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
    --build-arg GIT_COMMIT=$(git rev-parse --short HEAD) \
    --target runtime \
    -t $APP_NAME:$VERSION \
    -t $APP_NAME:latest \
    --push \
    -f Dockerfile.multiarch .

# Build development image for local platform only
docker buildx build \
    --platform linux/amd64 \
    --target debug \
    -t $APP_NAME:debug \
    --load \
    -f Dockerfile.multiarch .

echo "Multi-architecture build completed!"

# Display manifest information
docker buildx imagetools inspect $APP_NAME:$VERSION
```

### Task 4: Advanced Caching and Build Optimization (20 minutes)

**4.1 BuildKit Cache Mounts and Secrets**

```dockerfile
# File: Dockerfile.optimized
# Advanced BuildKit features for optimization

#################################
# Syntax directive for BuildKit features
#################################
# syntax=docker/dockerfile:1.4

FROM node:18-alpine AS base

WORKDIR /app

#################################
# Stage with cache mounts for package managers
#################################
FROM base AS deps

# Copy package files
COPY package*.json ./

# Use cache mount for npm cache to speed up builds
RUN --mount=type=cache,target=/root/.npm \
    npm ci --prefer-offline

#################################
# Stage with secret mounts for private repositories
#################################
FROM deps AS private-deps

# Mount secret for accessing private repositories
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    --mount=type=cache,target=/root/.npm \
    npm install @private/internal-package

#################################
# Build stage with cache mounts
#################################
FROM deps AS builder

# Copy source code
COPY . .

# Build with cache mount for build artifacts
RUN --mount=type=cache,target=/app/.next/cache \
    --mount=type=cache,target=/app/node_modules/.cache \
    npm run build

#################################
# Production stage
#################################
FROM base AS production

# Copy production dependencies and built app
COPY --from=deps /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package*.json ./

USER node

CMD ["node", "dist/server.js"]
```

**4.2 Advanced Build Script with Caching**

```bash
#!/bin/bash
# File: build-optimized.sh

set -e

APP_NAME="optimized-app"
VERSION=${VERSION:-"1.0.0"}

echo "=== Building with Advanced Caching ==="

# Enable BuildKit
export DOCKER_BUILDKIT=1

# Create .npmrc secret for private packages (if needed)
if [ -f ~/.npmrc ]; then
    echo "Using .npmrc for private package access"
    SECRETS="--secret id=npmrc,src=$HOME/.npmrc"
else
    SECRETS=""
fi

# Build with cache from registry
docker buildx build \
    --cache-from type=registry,ref=$APP_NAME:cache \
    --cache-to type=registry,ref=$APP_NAME:cache,mode=max \
    $SECRETS \
    --target production \
    -t $APP_NAME:$VERSION \
    -t $APP_NAME:latest \
    -f Dockerfile.optimized .

# Build debug version with local cache only
docker build \
    --target debug \
    -t $APP_NAME:debug \
    -f Dockerfile.optimized .

echo "Optimized build completed!"

# Show image layers and sizes
docker history $APP_NAME:latest
```

## Deliverables

1. **Advanced Multi-Stage Dockerfiles:**
   - Frontend build with optimization and testing stages
   - Backend build with security scanning and integration testing
   - Multi-architecture support for cross-platform deployment
   - Advanced caching and build optimization examples

2. **Build Scripts and Automation:**
   - Comprehensive build scripts for different targets
   - Multi-architecture build automation
   - Cache optimization and secret management
   - CI/CD integration examples

3. **Performance Analysis:**
   - Build time comparisons between approaches
   - Image size analysis and optimization results
   - Cache hit rate improvements
   - Documentation of optimization techniques

4. **Security Integration:**
   - Vulnerability scanning integration in build process
   - Secret management during builds
   - Non-root user implementation
   - Security best practices documentation

## Success Criteria

- Successfully built multi-stage images with significant size reduction (>50%)
- Implemented secure build practices with non-root users and secret management
- Achieved faster build times through effective caching strategies
- Created images that support multiple architectures
- Integrated security scanning and testing into build pipeline
- Demonstrated production-ready container optimization techniques

## Verification Commands

```bash
# Build and analyze all variants
./build-multistage.sh
./build-multiarch.sh
./build-optimized.sh

# Compare image sizes
docker images | grep -E "(dev|test|debug|latest)" | sort -k7 -h

# Analyze layers
docker history devops-frontend:latest
docker history devops-backend:latest

# Test multi-arch support
docker buildx imagetools inspect devops-backend:latest

# Verify security scanning
docker run --rm aquasec/trivy:latest image devops-backend:latest

# Test production containers
docker run -d --name prod-test devops-backend:latest
docker logs prod-test
docker exec prod-test ps aux  # Verify non-root execution
docker stop prod-test && docker rm prod-test
```

## Integration with DevOps Pipeline

These advanced multi-stage builds enable:
- **CI/CD Optimization**: Faster builds through intelligent caching
- **Security Integration**: Automated vulnerability scanning
- **Multi-Environment Support**: Different targets for different environments  
- **Cross-Platform Deployment**: Support for various architectures
- **Production Readiness**: Optimized, secure, and monitored containers

## Real-World Applications

**Enterprise Use Cases:**
- **Microservices Architecture**: Consistent build patterns across services
- **Multi-Cloud Deployment**: Cross-architecture support for different cloud providers
- **Security Compliance**: Integrated vulnerability scanning and secret management
- **Performance Optimization**: Minimal runtime images for cost and security benefits
- **Development Workflow**: Unified build process for development, testing, and production

This advanced multi-stage build exercise prepares students for sophisticated container deployment scenarios in enterprise environments!
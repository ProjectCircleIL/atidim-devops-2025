# Exercise 5: Docker Networking and Security Hardening

**Duration:** 150 minutes  
**Difficulty:** Advanced  
**Type:** Security Implementation + Network Architecture

## Objective
Master Docker networking concepts, implement container security best practices, and build secure, production-ready containerized applications with proper network isolation, secret management, and security hardening.

## Prerequisites
- Completed Exercise 4 (Docker Compose Development Environment)
- Understanding of networking fundamentals (TCP/IP, DNS, firewalls)
- Basic security concepts and threat modeling
- Working multi-tier application from previous exercises

## Docker Security and Networking Concepts

### Security Topics to Master:
1. **Container Security**: Non-root users, minimal attack surface, vulnerability scanning
2. **Network Security**: Network isolation, custom networks, firewall rules
3. **Secret Management**: Secure handling of credentials and sensitive data
4. **Image Security**: Base image selection, dependency scanning, signing
5. **Runtime Security**: Security contexts, capabilities, AppArmor/SELinux
6. **Compliance**: CIS benchmarks, security scanning automation
7. **Monitoring**: Security event logging and anomaly detection

### Networking Topics to Master:
1. **Bridge Networks**: Custom networks and isolation
2. **Overlay Networks**: Multi-host networking
3. **Host Networking**: Performance vs security trade-offs
4. **Network Policies**: Traffic filtering and access control
5. **Service Discovery**: DNS-based and registry-based discovery
6. **Load Balancing**: Traffic distribution and health checking
7. **Network Troubleshooting**: Debugging connectivity issues

## Tasks

### Task 1: Advanced Docker Networking (60 minutes)

**1.1 Custom Network Architecture**

Create a sophisticated network topology for your application:

```yaml
# File: docker-compose.secure-networks.yml
version: '3.8'

# Define multiple isolated networks
networks:
  # Public-facing network (DMZ-like)
  public-network:
    driver: bridge
    driver_opts:
      com.docker.network.bridge.name: br-public
    ipam:
      config:
        - subnet: 172.30.0.0/24
          gateway: 172.30.0.1
    labels:
      - "security.zone=dmz"
      - "access.level=public"

  # Application tier network
  app-network:
    driver: bridge
    driver_opts:
      com.docker.network.bridge.name: br-app
    ipam:
      config:
        - subnet: 172.31.0.0/24
          gateway: 172.31.0.1
    labels:
      - "security.zone=application"
      - "access.level=private"

  # Database tier network (most restricted)
  data-network:
    driver: bridge
    driver_opts:
      com.docker.network.bridge.name: br-data
    ipam:
      config:
        - subnet: 172.32.0.0/24
          gateway: 172.32.0.1
    labels:
      - "security.zone=data"
      - "access.level=restricted"

  # Management network (for monitoring and admin tools)
  mgmt-network:
    driver: bridge
    driver_opts:
      com.docker.network.bridge.name: br-mgmt
    ipam:
      config:
        - subnet: 172.33.0.0/24
          gateway: 172.33.0.1
    labels:
      - "security.zone=management"
      - "access.level=admin"

services:
  # Reverse proxy in DMZ (public-facing)
  nginx-proxy:
    build:
      context: ./nginx
      dockerfile: Dockerfile.secure
    container_name: secure_nginx_proxy
    
    # Only expose necessary ports
    ports:
      - "80:80"
      - "443:443"
    
    networks:
      public-network:
        ipv4_address: 172.30.0.10
      app-network:
        ipv4_address: 172.31.0.10
    
    volumes:
      - ./nginx/nginx.secure.conf:/etc/nginx/nginx.conf:ro
      - ./ssl-certs:/etc/ssl/certs:ro
      - nginx_logs:/var/log/nginx
    
    environment:
      - NGINX_USER=nginx
      - NGINX_WORKER_PROCESSES=auto
    
    # Security settings
    security_opt:
      - no-new-privileges:true
    
    # Read-only root filesystem
    read_only: true
    
    # Temporary filesystems for writable directories
    tmpfs:
      - /tmp:noexec,nosuid,size=1G
      - /var/cache/nginx:noexec,nosuid,size=100M
      - /var/run:noexec,nosuid,size=100M
    
    # Run as non-root user
    user: "101:101"  # nginx user
    
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:80/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
    
    restart: unless-stopped
    
    labels:
      - "security.role=proxy"
      - "network.zone=dmz"

  # Frontend application (in app network)
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile.secure
    container_name: secure_frontend
    
    # No direct port exposure (accessed through proxy)
    expose:
      - "3000"
    
    networks:
      app-network:
        ipv4_address: 172.31.0.20
    
    volumes:
      - frontend_logs:/app/logs
    
    environment:
      - NODE_ENV=production
      - REACT_APP_API_URL=/api
    
    # Security hardening
    security_opt:
      - no-new-privileges:true
      - apparmor:docker-default
    
    read_only: true
    tmpfs:
      - /tmp:noexec,nosuid,size=100M
      - /app/.next/cache:noexec,nosuid,size=200M
    
    # Run as non-root user
    user: "1001:1001"
    
    # Resource limits
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '0.5'
        reservations:
          memory: 256M
          cpus: '0.25'
    
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    
    depends_on:
      - backend
    
    labels:
      - "security.role=frontend"
      - "network.zone=app"

  # Backend API (in app network, can access database network)
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile.secure
    container_name: secure_backend
    
    expose:
      - "3001"
    
    networks:
      app-network:
        ipv4_address: 172.31.0.30
      data-network:
        ipv4_address: 172.32.0.30
    
    volumes:
      - backend_logs:/app/logs
    
    environment:
      - NODE_ENV=production
      - PORT=3001
    
    # Use secrets for sensitive data
    secrets:
      - database_url
      - jwt_secret
      - api_keys
    
    # Security hardening
    security_opt:
      - no-new-privileges:true
      - apparmor:docker-default
    
    # Drop all capabilities and add only necessary ones
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    
    read_only: true
    tmpfs:
      - /tmp:noexec,nosuid,size=100M
      - /app/logs:noexec,nosuid,size=50M
    
    user: "1001:1001"
    
    deploy:
      resources:
        limits:
          memory: 1G
          cpus: '1.0'
        reservations:
          memory: 512M
          cpus: '0.5'
    
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3001/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    
    depends_on:
      - postgres
      - redis
    
    labels:
      - "security.role=api"
      - "network.zone=app"

  # Database (isolated in data network)
  postgres:
    image: postgres:15-alpine
    container_name: secure_postgres
    
    # No external port exposure
    expose:
      - "5432"
    
    networks:
      data-network:
        ipv4_address: 172.32.0.40
    
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - postgres_logs:/var/log/postgresql
    
    environment:
      - POSTGRES_DB=devops_app
      - POSTGRES_USER=appuser
    
    # Use secrets for password
    secrets:
      - postgres_password
    
    # Security hardening
    security_opt:
      - no-new-privileges:true
      - apparmor:docker-default
    
    # Custom user with limited privileges
    user: "999:999"  # postgres user
    
    # Resource limits
    deploy:
      resources:
        limits:
          memory: 2G
          cpus: '1.0'
        reservations:
          memory: 1G
          cpus: '0.5'
    
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d devops_app"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    
    labels:
      - "security.role=database"
      - "network.zone=data"

  # Redis cache (isolated in data network)
  redis:
    image: redis:7-alpine
    container_name: secure_redis
    
    expose:
      - "6379"
    
    networks:
      data-network:
        ipv4_address: 172.32.0.50
    
    volumes:
      - redis_data:/data
      - ./redis/redis.secure.conf:/etc/redis/redis.conf:ro
    
    # Security hardening
    security_opt:
      - no-new-privileges:true
      - apparmor:docker-default
    
    user: "999:999"  # redis user
    
    # Resource limits
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '0.5'
        reservations:
          memory: 256M
          cpus: '0.25'
    
    command: ["redis-server", "/etc/redis/redis.conf", "--requirepass", "$(cat /run/secrets/redis_password)"]
    
    secrets:
      - redis_password
    
    healthcheck:
      test: ["CMD", "redis-cli", "--no-auth-warning", "-a", "$(cat /run/secrets/redis_password)", "ping"]
      interval: 30s
      timeout: 10s
      retries: 3
    
    labels:
      - "security.role=cache"
      - "network.zone=data"

  # Security monitoring (in management network)
  falco:
    image: falcosecurity/falco:latest
    container_name: security_monitor
    
    networks:
      - mgmt-network
    
    privileged: true
    
    volumes:
      - /var/run/docker.sock:/host/var/run/docker.sock:ro
      - /dev:/host/dev:ro
      - /proc:/host/proc:ro
      - /boot:/host/boot:ro
      - /lib/modules:/host/lib/modules:ro
      - /usr:/host/usr:ro
      - ./falco/falco.yaml:/etc/falco/falco.yaml:ro
      - falco_logs:/var/log/falco
    
    environment:
      - HOST_ROOT=/host
    
    labels:
      - "security.role=monitor"
      - "network.zone=mgmt"

# Define secrets
secrets:
  database_url:
    external: true
  jwt_secret:
    external: true
  api_keys:
    external: true
  postgres_password:
    external: true
  redis_password:
    external: true

# Define volumes
volumes:
  nginx_logs:
  frontend_logs:
  backend_logs:
  postgres_data:
  postgres_logs:
  redis_data:
  falco_logs:
```

**1.2 Network Security Configuration**

Create secure Nginx configuration:

```nginx
# File: nginx/nginx.secure.conf
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

# Security: Hide version information
server_tokens off;

events {
    worker_connections 1024;
    use epoll;
    multi_accept on;
}

http {
    # Basic settings
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;
    add_header Content-Security-Policy "default-src 'self' http: https: data: blob: 'unsafe-inline'" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    
    # Rate limiting
    limit_req_zone $binary_remote_addr zone=login:10m rate=1r/s;
    limit_req_zone $binary_remote_addr zone=general:10m rate=10r/s;
    
    # Connection limiting
    limit_conn_zone $binary_remote_addr zone=conn_limit_per_ip:10m;
    
    # Logging format with security information
    log_format security '$remote_addr - $remote_user [$time_local] '
                       '"$request" $status $body_bytes_sent '
                       '"$http_referer" "$http_user_agent" '
                       '$request_time $upstream_response_time '
                       '$scheme $server_name';
    
    access_log /var/log/nginx/access.log security;
    
    # Performance optimizations
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    client_max_body_size 10M;
    
    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml+rss text/javascript;
    
    # Upstream backend servers with health checks
    upstream backend_servers {
        least_conn;
        server 172.31.0.30:3001 max_fails=3 fail_timeout=30s;
        keepalive 32;
    }
    
    upstream frontend_servers {
        least_conn;
        server 172.31.0.20:3000 max_fails=3 fail_timeout=30s;
        keepalive 32;
    }
    
    # Security: Block common attack patterns
    map $request_uri $blocked_uri {
        default 0;
        ~*\.(php|aspx|jsp)$ 1;
        ~*/wp-admin 1;
        ~*/phpmyadmin 1;
        ~*/\.env 1;
        ~*/\.git 1;
    }
    
    # Main server block
    server {
        listen 80;
        listen [::]:80;
        server_name _;
        
        # Connection limits
        limit_conn conn_limit_per_ip 10;
        
        # Block malicious requests
        if ($blocked_uri) {
            return 444;
        }
        
        # Security: Hide server information
        server_tokens off;
        
        # Health check endpoint
        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
        
        # API routes with rate limiting
        location /api/ {
            limit_req zone=general burst=20 nodelay;
            
            proxy_pass http://backend_servers/api/;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_cache_bypass $http_upgrade;
            
            # Timeout settings
            proxy_connect_timeout 30s;
            proxy_send_timeout 30s;
            proxy_read_timeout 30s;
            
            # Security headers
            proxy_set_header X-Request-ID $request_id;
        }
        
        # Authentication routes with stricter rate limiting
        location /api/auth/ {
            limit_req zone=login burst=5 nodelay;
            
            proxy_pass http://backend_servers/api/auth/;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
        
        # Static file serving with caching
        location / {
            limit_req zone=general burst=50 nodelay;
            
            proxy_pass http://frontend_servers/;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            # Caching for static assets
            location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
                expires 1y;
                add_header Cache-Control "public, immutable";
                add_header X-Frame-Options "SAMEORIGIN";
                add_header X-Content-Type-Options "nosniff";
            }
        }
        
        # Security: Deny access to hidden files
        location ~ /\. {
            deny all;
            access_log off;
            log_not_found off;
        }
        
        # Security: Deny access to backup files
        location ~ ~$ {
            deny all;
            access_log off;
            log_not_found off;
        }
    }
}
```

### Task 2: Container Security Hardening (60 minutes)

**2.1 Secure Dockerfile Practices**

Create hardened Dockerfiles:

```dockerfile
# File: backend/Dockerfile.secure
# Secure multi-stage build for Node.js backend

#################################
# Security-focused base image
#################################
FROM node:18-alpine AS base

# Install security updates
RUN apk upgrade --no-cache && \
    apk add --no-cache \
    dumb-init \
    curl \
    ca-certificates \
    && update-ca-certificates

# Create non-root user with minimal privileges
RUN addgroup -g 1001 -S appgroup && \
    adduser -S appuser -u 1001 -G appgroup -h /app -s /bin/sh

# Set secure directory permissions
RUN mkdir -p /app && \
    chown -R appuser:appgroup /app && \
    chmod 755 /app

WORKDIR /app

#################################
# Dependencies stage
#################################
FROM base AS deps

# Copy package files
COPY --chown=appuser:appgroup package*.json ./

# Install dependencies with security audit
USER appuser
RUN npm ci --only=production --audit && \
    npm cache clean --force

#################################
# Build stage
#################################
FROM base AS builder

# Copy all dependencies
COPY --from=deps --chown=appuser:appgroup /app/node_modules ./node_modules

# Copy source code
COPY --chown=appuser:appgroup . .

USER appuser

# Build application
RUN npm run build && \
    npm run test

#################################
# Security scanning stage
#################################
FROM aquasec/trivy:latest AS security-scan

# Copy built application for scanning
COPY --from=builder /app /scan-target

# Run security scan
RUN trivy fs --exit-code 0 --no-progress --format table /scan-target

#################################
# Production stage
#################################
FROM base AS production

# Copy production dependencies
COPY --from=deps --chown=appuser:appgroup /app/node_modules ./node_modules

# Copy built application
COPY --from=builder --chown=appuser:appgroup /app/dist ./dist
COPY --from=builder --chown=appuser:appgroup /app/package*.json ./

# Create necessary directories with secure permissions
RUN mkdir -p /app/logs /app/tmp && \
    chown -R appuser:appgroup /app/logs /app/tmp && \
    chmod 755 /app/logs /app/tmp

# Switch to non-root user
USER appuser

# Security: Remove unnecessary packages
RUN apk del --no-cache \
    && rm -rf /var/cache/apk/* \
    && rm -rf /tmp/*

# Set environment variables
ENV NODE_ENV=production
ENV PORT=3001
ENV USER=appuser

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:$PORT/api/health || exit 1

# Expose port
EXPOSE $PORT

# Use dumb-init for proper signal handling
ENTRYPOINT ["dumb-init", "--"]

# Start application
CMD ["node", "dist/server.js"]

# Security labels
LABEL security.scan="trivy" \
      security.non-root="true" \
      security.readonly-rootfs="true" \
      maintainer="devops-team" \
      version="1.0.0"
```

**2.2 Secret Management Implementation**

Create secure secret management:

```bash
#!/bin/bash
# File: scripts/setup-secrets.sh
# Script to set up Docker secrets securely

set -e

echo "=== Setting up Docker Secrets ==="

# Create secrets directory if it doesn't exist
mkdir -p secrets

# Generate strong random passwords
generate_password() {
    openssl rand -base64 32 | tr -d "=+/" | cut -c1-25
}

# Database password
if [ ! -f secrets/postgres_password ]; then
    generate_password > secrets/postgres_password
    echo "Generated PostgreSQL password"
fi

# Redis password
if [ ! -f secrets/redis_password ]; then
    generate_password > secrets/redis_password
    echo "Generated Redis password"
fi

# JWT secret
if [ ! -f secrets/jwt_secret ]; then
    openssl rand -hex 64 > secrets/jwt_secret
    echo "Generated JWT secret"
fi

# Database URL
if [ ! -f secrets/database_url ]; then
    POSTGRES_PASSWORD=$(cat secrets/postgres_password)
    echo "postgresql://appuser:${POSTGRES_PASSWORD}@postgres:5432/devops_app" > secrets/database_url
    echo "Generated Database URL"
fi

# API keys (example)
if [ ! -f secrets/api_keys ]; then
    cat > secrets/api_keys << EOF
STRIPE_API_KEY=$(generate_password)
SENDGRID_API_KEY=$(generate_password)
AWS_ACCESS_KEY_ID=$(generate_password)
AWS_SECRET_ACCESS_KEY=$(generate_password)
EOF
    echo "Generated API keys"
fi

# Create Docker secrets
echo "Creating Docker secrets..."

docker secret create database_url secrets/database_url 2>/dev/null || echo "database_url secret already exists"
docker secret create jwt_secret secrets/jwt_secret 2>/dev/null || echo "jwt_secret secret already exists"
docker secret create api_keys secrets/api_keys 2>/dev/null || echo "api_keys secret already exists"
docker secret create postgres_password secrets/postgres_password 2>/dev/null || echo "postgres_password secret already exists"
docker secret create redis_password secrets/redis_password 2>/dev/null || echo "redis_password secret already exists"

# Set proper permissions on secrets directory
chmod 600 secrets/*
chmod 700 secrets

echo "✅ Docker secrets setup completed!"
echo "🔒 Secrets files are stored in ./secrets/ with restricted permissions"
echo "⚠️  Remember to add secrets/ to your .gitignore file"

# Add to .gitignore if not already there
if ! grep -q "secrets/" .gitignore 2>/dev/null; then
    echo "secrets/" >> .gitignore
    echo "Added secrets/ to .gitignore"
fi
```

**2.3 Security Monitoring Configuration**

```yaml
# File: falco/falco.yaml
# Falco configuration for container security monitoring
global:
  # Enable syscall event drop alerting
  syscall_event_drops:
    actions:
      - log
      - alert
    rate: .03333
    max_burst: 1000

# Output channels configuration
file_output:
  enabled: true
  keep_alive: false
  filename: /var/log/falco/falco.log

json_output: true
json_include_output_property: true
json_include_tags_property: true

# Rules configuration
rules_file:
  - /etc/falco/falco_rules.yaml
  - /etc/falco/falco_rules.local.yaml
  - /etc/falco/k8s_audit_rules.yaml
  - /etc/falco/application_rules.yaml

# Load additional rules for Docker security
load_plugins: [k8saudit, json]

# Custom rules for our application
application_rules: |
  # Detect shell access to containers
  - rule: Shell in Container
    desc: A shell was spawned in a container
    condition: >
      spawned_process and container and
      (proc.name in (sh, bash, zsh, csh, ksh, fish))
    output: >
      Shell spawned in container (user=%user.name container_id=%container.id 
      container_name=%container.name shell=%proc.name)
    priority: WARNING
    tags: [container, shell, security]

  # Detect privilege escalation
  - rule: Privilege Escalation in Container
    desc: Detect privilege escalation in container
    condition: >
      spawned_process and container and
      (proc.args contains "sudo" or proc.args contains "su ")
    output: >
      Privilege escalation attempt in container (user=%user.name 
      container_id=%container.id container_name=%container.name command=%proc.cmdline)
    priority: HIGH
    tags: [container, privilege_escalation, security]

  # Detect network connections to suspicious ports
  - rule: Suspicious Network Connection
    desc: Detect connections to suspicious ports
    condition: >
      inbound_outbound and fd.type=ipv4 and
      (fd.rport in (22, 23, 1433, 3306, 5432, 6379) or fd.lport in (22, 23, 1433, 3306, 5432, 6379))
    output: >
      Suspicious network connection (user=%user.name container_id=%container.id 
      container_name=%container.name connection=%fd.name)
    priority: WARNING
    tags: [network, suspicious, security]

  # Detect file changes in sensitive directories
  - rule: Sensitive File Access
    desc: Detect access to sensitive files
    condition: >
      open_write and container and
      (fd.name startswith /etc/passwd or fd.name startswith /etc/shadow or
       fd.name startswith /etc/hosts or fd.name startswith /root/.ssh)
    output: >
      Sensitive file access (user=%user.name container_id=%container.id 
      container_name=%container.name file=%fd.name)
    priority: HIGH
    tags: [file, sensitive, security]
```

### Task 3: Production Security Testing and Compliance (30 minutes)

**2.4 Security Testing Automation**

Create comprehensive security testing:

```bash
#!/bin/bash
# File: scripts/security-test.sh
# Comprehensive security testing for Docker containers

set -e

echo "=== Docker Security Testing Suite ==="

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

print_header() {
    echo -e "\n${GREEN}=== $1 ===${NC}\n"
}

print_warning() {
    echo -e "${YELLOW}⚠️  $1${NC}"
}

print_error() {
    echo -e "${RED}❌ $1${NC}"
}

print_success() {
    echo -e "${GREEN}✅ $1${NC}"
}

# Test 1: Image vulnerability scanning
print_header "1. Container Image Vulnerability Scanning"

IMAGES=("secure_frontend" "secure_backend" "secure_postgres")

for image in "${IMAGES[@]}"; do
    echo "Scanning image: $image"
    
    if docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
        aquasec/trivy:latest image --exit-code 1 --severity HIGH,CRITICAL "$image"; then
        print_success "No high/critical vulnerabilities found in $image"
    else
        print_error "Vulnerabilities found in $image"
    fi
done

# Test 2: Container configuration security
print_header "2. Container Configuration Security"

check_container_security() {
    local container_name=$1
    echo "Checking container: $container_name"
    
    # Check if running as root
    if docker exec "$container_name" whoami | grep -q root; then
        print_error "$container_name is running as root user"
    else
        print_success "$container_name is running as non-root user"
    fi
    
    # Check for privileged mode
    if docker inspect "$container_name" | jq -r '.[0].HostConfig.Privileged' | grep -q true; then
        print_error "$container_name is running in privileged mode"
    else
        print_success "$container_name is not running in privileged mode"
    fi
    
    # Check for read-only root filesystem
    if docker inspect "$container_name" | jq -r '.[0].HostConfig.ReadonlyRootfs' | grep -q true; then
        print_success "$container_name has read-only root filesystem"
    else
        print_warning "$container_name does not have read-only root filesystem"
    fi
    
    # Check capabilities
    caps=$(docker inspect "$container_name" | jq -r '.[0].HostConfig.CapAdd[]?' 2>/dev/null)
    if [ -z "$caps" ]; then
        print_success "$container_name has no additional capabilities"
    else
        print_warning "$container_name has additional capabilities: $caps"
    fi
}

# Check running containers
CONTAINERS=$(docker ps --format "{{.Names}}" | grep secure_)
for container in $CONTAINERS; do
    check_container_security "$container"
done

# Test 3: Network security testing
print_header "3. Network Security Testing"

# Test network isolation
echo "Testing network isolation..."

# Test if frontend can directly access database
if docker exec secure_frontend curl -m 5 -f http://172.32.0.40:5432/ 2>/dev/null; then
    print_error "Frontend can directly access database network"
else
    print_success "Frontend cannot directly access database network"
fi

# Test if services are exposed only on necessary ports
echo "Checking exposed ports..."
exposed_ports=$(docker ps --format "table {{.Names}}\t{{.Ports}}" | grep -v PORTS)
echo "$exposed_ports"

# Test 4: Secret management validation
print_header "4. Secret Management Validation"

# Check if secrets are properly mounted
check_secrets() {
    local container_name=$1
    echo "Checking secrets in: $container_name"
    
    if docker exec "$container_name" ls /run/secrets/ 2>/dev/null; then
        print_success "$container_name has secrets mounted"
        
        # Check secret permissions
        secret_perms=$(docker exec "$container_name" stat -c %a /run/secrets/* 2>/dev/null | head -1)
        if [ "$secret_perms" = "444" ]; then
            print_success "Secrets have correct permissions (444)"
        else
            print_warning "Secrets permissions: $secret_perms"
        fi
    else
        print_warning "$container_name has no secrets mounted"
    fi
}

for container in $CONTAINERS; do
    check_secrets "$container"
done

# Test 5: Runtime security monitoring
print_header "5. Runtime Security Monitoring"

# Check if Falco is running and detecting events
if docker ps | grep -q security_monitor; then
    print_success "Falco security monitoring is active"
    
    # Check recent Falco alerts
    echo "Recent security alerts:"
    docker logs security_monitor --tail 10 2>/dev/null || print_warning "No recent alerts"
else
    print_warning "Falco security monitoring is not running"
fi

# Test 6: SSL/TLS configuration
print_header "6. SSL/TLS Security Testing"

# Test SSL configuration with sslyze (if available)
if command -v testssl.sh &> /dev/null; then
    echo "Testing SSL configuration..."
    testssl.sh --quiet --color 0 localhost:443
else
    print_warning "testssl.sh not available, skipping SSL tests"
fi

# Test 7: Application security
print_header "7. Application Security Testing"

# Basic security headers check
echo "Checking security headers..."
if curl -s -I http://localhost:80 | grep -q "X-Frame-Options"; then
    print_success "Security headers are present"
else
    print_error "Security headers are missing"
fi

# Test rate limiting
echo "Testing rate limiting..."
for i in {1..15}; do
    response=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:80/api/auth/login)
    if [ "$response" = "429" ]; then
        print_success "Rate limiting is working (got 429 after $i requests)"
        break
    fi
done

# Test 8: Compliance checks (CIS Docker Benchmark)
print_header "8. CIS Docker Benchmark Compliance"

# Run Docker Bench Security (if available)
if [ -d "/opt/docker-bench-security" ]; then
    echo "Running CIS Docker Benchmark..."
    cd /opt/docker-bench-security && ./docker-bench-security.sh
else
    print_warning "Docker Bench Security not installed"
    echo "Install with: git clone https://github.com/docker/docker-bench-security.git /opt/docker-bench-security"
fi

# Summary
print_header "Security Test Summary"
echo "Security testing completed. Review all warnings and errors above."
echo "For production deployment:"
echo "1. Fix all HIGH and CRITICAL vulnerabilities"
echo "2. Address security configuration warnings"
echo "3. Enable comprehensive monitoring and alerting"
echo "4. Regular security updates and rescanning"

print_success "Security testing suite completed!"
```

## Deliverables

1. **Secure Network Architecture:**
   - Multi-tier network isolation with custom bridge networks
   - Secure reverse proxy configuration with rate limiting
   - Network security policies and firewall rules
   - Service discovery and load balancing setup

2. **Container Security Hardening:**
   - Hardened Dockerfiles with non-root users and minimal attack surface
   - Read-only root filesystems and capability restrictions
   - Comprehensive vulnerability scanning integration
   - Security monitoring with Falco and alerting

3. **Secret Management System:**
   - Docker secrets integration for sensitive data
   - Secure secret generation and rotation procedures
   - Environment-specific secret management
   - Secret scanning and leak prevention

4. **Security Testing Automation:**
   - Comprehensive security testing suite
   - CIS benchmark compliance checking
   - Automated vulnerability scanning
   - Runtime security monitoring and alerting

## Success Criteria

- Successfully implemented network isolation with secure communication patterns
- Hardened all container images with security best practices
- Established secure secret management for all sensitive data
- Created comprehensive security testing and monitoring
- Achieved compliance with security benchmarks and standards
- Demonstrated secure deployment patterns for production environments

## Verification Commands

```bash
# Run security testing suite
./scripts/security-test.sh

# Set up secrets securely
./scripts/setup-secrets.sh

# Deploy secure environment
docker-compose -f docker-compose.secure-networks.yml up -d

# Test network isolation
docker exec secure_frontend ping -c 1 172.32.0.40  # Should fail
docker exec secure_backend ping -c 1 172.32.0.40   # Should succeed

# Check security monitoring
docker logs security_monitor --tail 20

# Scan for vulnerabilities
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy:latest image --severity HIGH,CRITICAL secure_backend
```

## Integration with Next Exercise

This security foundation prepares for:
- Kubernetes security policies and RBAC
- Production container orchestration security
- Cloud security and compliance requirements
- Advanced threat detection and incident response
- Security automation in CI/CD pipelines

## Real-World Applications

**Enterprise Security Requirements:**
- **Compliance**: Meeting SOC2, GDPR, HIPAA, and other regulatory requirements
- **Zero Trust Architecture**: Network segmentation and least-privilege access
- **Threat Detection**: Real-time security monitoring and incident response
- **Vulnerability Management**: Continuous scanning and remediation
- **Secret Management**: Enterprise-grade credential and key management

This comprehensive security exercise provides 150 minutes of intensive hands-on experience with production-grade container security and networking!
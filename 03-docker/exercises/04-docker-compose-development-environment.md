# Exercise 4: Docker Compose Development Environment Mastery

**Duration:** 180 minutes  
**Difficulty:** Intermediate to Advanced  
**Type:** Hands-on Implementation + Environment Configuration

## Objective
Master Docker Compose for creating sophisticated development environments, understanding service orchestration, networking, data persistence, and development workflow optimization. Build production-like local environments for your 3-tier application.

## Prerequisites
- Completed Exercise 3 (Multi-Stage Builds Advanced)
- Working Dockerfiles for frontend, backend, and database
- Understanding of networking and data persistence concepts
- Your application code from previous modules ready for integration

## Docker Compose Concepts to Master

### Core Features:
1. **Service Definition**: Multi-service application orchestration
2. **Networking**: Custom networks and service communication
3. **Volumes**: Data persistence and bind mounts
4. **Environment Variables**: Configuration management across services
5. **Dependencies**: Service startup order and health checks
6. **Scaling**: Running multiple instances of services
7. **Overrides**: Environment-specific configurations
8. **Profiles**: Selective service activation
9. **Extensions**: Reusable configuration blocks
10. **Development Workflows**: Hot reloading and debugging integration

## Tasks

### Task 1: Complete Development Environment Setup (60 minutes)

**1.1 Basic Multi-Service Docker Compose**

Create a comprehensive development environment:

```yaml
# File: docker-compose.dev.yml
version: '3.8'

# Define custom networks
networks:
  frontend-network:
    driver: bridge
  backend-network:
    driver: bridge
  database-network:
    driver: bridge

# Define named volumes for data persistence
volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local
  mongodb_data:
    driver: local
  elasticsearch_data:
    driver: local

# Services definition
services:
  # Frontend Service (React/Vue.js application)
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile.multistage
      target: development
      args:
        NODE_ENV: development
        REACT_APP_API_URL: http://localhost:3001
        REACT_APP_WS_URL: ws://localhost:3001
    
    container_name: devops_frontend_dev
    
    ports:
      - "3000:3000"
      - "3001:3001"  # For webpack dev server
    
    environment:
      - NODE_ENV=development
      - CHOKIDAR_USEPOLLING=true  # For file watching on some systems
      - REACT_APP_API_URL=http://backend:3001
      - REACT_APP_VERSION=dev-${VERSION:-latest}
      - REACT_APP_BUILD_DATE=${BUILD_DATE:-unknown}
    
    volumes:
      # Bind mount source code for hot reloading
      - ./frontend/src:/app/src:cached
      - ./frontend/public:/app/public:cached
      - ./frontend/package.json:/app/package.json:ro
      - ./frontend/package-lock.json:/app/package-lock.json:ro
      # Named volume for node_modules (performance optimization)
      - frontend_node_modules:/app/node_modules
    
    networks:
      - frontend-network
      - backend-network
    
    depends_on:
      backend:
        condition: service_healthy
    
    # Development-specific health check
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    
    # Restart policy
    restart: unless-stopped
    
    # Development labels
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.frontend.rule=Host(`app.localhost`)"
      - "traefik.http.services.frontend.loadbalancer.server.port=3000"
      - "com.docker.compose.service=frontend"
      - "environment=development"

  # Backend Service (Node.js/Python API)
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile.multistage
      target: development
      args:
        NODE_ENV: development
        PORT: 3001
    
    container_name: devops_backend_dev
    
    ports:
      - "3001:3001"
      - "9229:9229"  # Node.js debugging port
    
    environment:
      - NODE_ENV=development
      - PORT=3001
      - DATABASE_URL=postgresql://postgres:devpassword@postgres:5432/devops_app_dev
      - REDIS_URL=redis://redis:6379
      - MONGODB_URL=mongodb://mongo:27017/devops_app_dev
      - JWT_SECRET=dev-jwt-secret-change-in-production
      - LOG_LEVEL=debug
      - CORS_ORIGIN=http://localhost:3000
    
    volumes:
      # Bind mount source code for hot reloading
      - ./backend/src:/app/src:cached
      - ./backend/tests:/app/tests:cached
      - ./backend/package.json:/app/package.json:ro
      - ./backend/package-lock.json:/app/package-lock.json:ro
      # Named volume for node_modules
      - backend_node_modules:/app/node_modules
      # Mount for logs
      - ./logs/backend:/app/logs
    
    networks:
      - backend-network
      - database-network
    
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
      mongo:
        condition: service_healthy
    
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3001/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    
    restart: unless-stopped
    
    # Enable debugging
    command: ["npm", "run", "dev:debug"]
    
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.backend.rule=Host(`api.localhost`)"
      - "traefik.http.services.backend.loadbalancer.server.port=3001"
      - "environment=development"

  # PostgreSQL Database
  postgres:
    image: postgres:15-alpine
    container_name: devops_postgres_dev
    
    ports:
      - "5432:5432"
    
    environment:
      POSTGRES_DB: devops_app_dev
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: devpassword
      POSTGRES_INITDB_ARGS: "--auth-host=scram-sha-256"
    
    volumes:
      # Persistent data storage
      - postgres_data:/var/lib/postgresql/data
      # Initialization scripts
      - ./database/init-scripts:/docker-entrypoint-initdb.d:ro
      # Custom PostgreSQL configuration
      - ./database/postgresql.dev.conf:/etc/postgresql/postgresql.conf:ro
    
    networks:
      - database-network
    
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d devops_app_dev"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    
    restart: unless-stopped
    
    # Use custom PostgreSQL configuration
    command: ["postgres", "-c", "config_file=/etc/postgresql/postgresql.conf"]
    
    labels:
      - "environment=development"
      - "service=database"

  # Redis for Caching and Sessions
  redis:
    image: redis:7-alpine
    container_name: devops_redis_dev
    
    ports:
      - "6379:6379"
    
    volumes:
      - redis_data:/data
      - ./database/redis.dev.conf:/etc/redis/redis.conf:ro
    
    networks:
      - database-network
    
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 10s
    
    restart: unless-stopped
    
    command: ["redis-server", "/etc/redis/redis.conf"]
    
    labels:
      - "environment=development"
      - "service=cache"

  # MongoDB for Document Storage (if needed)
  mongo:
    image: mongo:6
    container_name: devops_mongo_dev
    
    ports:
      - "27017:27017"
    
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: devpassword
      MONGO_INITDB_DATABASE: devops_app_dev
    
    volumes:
      - mongodb_data:/data/db
      - ./database/mongo-init.js:/docker-entrypoint-initdb.d/mongo-init.js:ro
    
    networks:
      - database-network
    
    healthcheck:
      test: ["CMD", "mongo", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 20s
    
    restart: unless-stopped
    
    labels:
      - "environment=development"
      - "service=document-store"

  # Elasticsearch for Search and Analytics
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.8.0
    container_name: devops_elasticsearch_dev
    
    ports:
      - "9200:9200"
      - "9300:9300"
    
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - cluster.name=devops-cluster
      - node.name=devops-node-1
    
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
    
    networks:
      - database-network
    
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9200/_health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    
    restart: unless-stopped
    
    labels:
      - "environment=development"
      - "service=search"

  # Reverse Proxy with Traefik
  traefik:
    image: traefik:v3.0
    container_name: devops_traefik_dev
    
    ports:
      - "80:80"
      - "8080:8080"  # Traefik dashboard
    
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik/traefik.dev.yml:/etc/traefik/traefik.yml:ro
    
    networks:
      - frontend-network
      - backend-network
    
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.dashboard.rule=Host(`traefik.localhost`)"
      - "traefik.http.services.dashboard.loadbalancer.server.port=8080"
    
    restart: unless-stopped

# Additional volumes for node_modules
volumes:
  frontend_node_modules:
  backend_node_modules:
```

**1.2 Environment-Specific Configuration Files**

Create supporting configuration files:

```yaml
# File: traefik/traefik.dev.yml
api:
  dashboard: true
  insecure: true

entryPoints:
  web:
    address: ":80"

providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false

log:
  level: DEBUG
```

```conf
# File: database/postgresql.dev.conf
# PostgreSQL configuration for development
listen_addresses = '*'
port = 5432
max_connections = 200
shared_buffers = 256MB
dynamic_shared_memory_type = posix
max_wal_size = 1GB
min_wal_size = 80MB

# Logging configuration
log_statement = 'all'
log_duration = on
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '

# Development-friendly settings
fsync = off
synchronous_commit = off
full_page_writes = off
```

```conf
# File: database/redis.dev.conf
# Redis configuration for development
bind 0.0.0.0
port 6379
timeout 0
save 900 1
save 300 10
save 60 10000
rdbcompression yes
dbfilename dump.rdb
dir /data
maxmemory 256mb
maxmemory-policy allkeys-lru
```

### Task 2: Advanced Docker Compose Features (60 minutes)

**2.1 Environment Overrides and Profiles**

```yaml
# File: docker-compose.override.yml
# Local development overrides (automatically applied)
version: '3.8'

services:
  backend:
    # Override for local development with file watching
    volumes:
      - ./backend:/app:cached
    environment:
      - DEBUG=*
      - NODE_ENV=development
    
    # Add development tools
    depends_on:
      - mailhog  # Local email testing
      - pgadmin  # Database management

  # Add development-only services
  mailhog:
    image: mailhog/mailhog:v1.0.1
    container_name: devops_mailhog_dev
    ports:
      - "1025:1025"  # SMTP
      - "8025:8025"  # Web UI
    networks:
      - backend-network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.mailhog.rule=Host(`mail.localhost`)"

  pgadmin:
    image: dpage/pgadmin4:latest
    container_name: devops_pgadmin_dev
    ports:
      - "5050:80"
    environment:
      PGADMIN_DEFAULT_EMAIL: dev@example.com
      PGADMIN_DEFAULT_PASSWORD: devpassword
    volumes:
      - pgadmin_data:/var/lib/pgadmin
    networks:
      - database-network
    depends_on:
      - postgres
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.pgadmin.rule=Host(`pgadmin.localhost`)"

volumes:
  pgadmin_data:
```

**2.2 Profile-Based Service Management**

```yaml
# File: docker-compose.profiles.yml
# Profile-based service activation
version: '3.8'

services:
  # Monitoring stack (profile: monitoring)
  prometheus:
    image: prom/prometheus:v2.40.0
    container_name: devops_prometheus_dev
    profiles: ["monitoring", "full"]
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--web.enable-lifecycle'
    networks:
      - backend-network

  grafana:
    image: grafana/grafana:9.3.0
    container_name: devops_grafana_dev
    profiles: ["monitoring", "full"]
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=devpassword
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana/provisioning:/etc/grafana/provisioning:ro
    networks:
      - backend-network
    depends_on:
      - prometheus

  # Testing stack (profile: testing)
  selenium-hub:
    image: selenium/hub:4.8.0
    container_name: devops_selenium_hub
    profiles: ["testing", "e2e"]
    ports:
      - "4444:4444"
    networks:
      - frontend-network

  selenium-chrome:
    image: selenium/node-chrome:4.8.0
    container_name: devops_selenium_chrome
    profiles: ["testing", "e2e"]
    environment:
      - HUB_HOST=selenium-hub
      - HUB_PORT=4444
    depends_on:
      - selenium-hub
    networks:
      - frontend-network

  # Load testing (profile: performance)
  k6:
    image: grafana/k6:latest
    container_name: devops_k6_performance
    profiles: ["performance", "load-test"]
    volumes:
      - ./tests/performance:/scripts:ro
    networks:
      - frontend-network
    command: run /scripts/load-test.js

volumes:
  prometheus_data:
  grafana_data:
```

**2.3 Advanced Networking Configuration**

```yaml
# File: docker-compose.networking.yml
# Advanced networking scenarios
version: '3.8'

networks:
  # External network (connects to other projects)
  external-network:
    external: true
    name: shared-dev-network
  
  # Custom bridge network with specific configuration
  app-network:
    driver: bridge
    driver_opts:
      com.docker.network.bridge.name: br-devops-app
    ipam:
      config:
        - subnet: 172.20.0.0/16
          gateway: 172.20.0.1

services:
  # Service with static IP
  backend:
    networks:
      app-network:
        ipv4_address: 172.20.0.10
      external-network:
    # Network aliases for service discovery
    aliases:
      - api
      - backend-service

  # Service accessible from host network
  debug-tools:
    image: nicolaka/netshoot:latest
    container_name: devops_debug_tools
    network_mode: host
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    command: sleep infinity
    profiles: ["debug"]
```

### Task 3: Development Workflow Optimization (60 minutes)

**2.4 Hot Reloading and Development Tools Integration**

```yaml
# File: docker-compose.development.yml
# Optimized for development workflow
version: '3.8'

services:
  frontend:
    build:
      target: development
      args:
        INSTALL_DEV_TOOLS: "true"
    
    volumes:
      # Optimized bind mounts for different platforms
      - type: bind
        source: ./frontend/src
        target: /app/src
        consistency: cached
      
      - type: bind
        source: ./frontend/public
        target: /app/public
        consistency: cached
      
      # Named volume for node_modules (faster on Windows/macOS)
      - type: volume
        source: frontend_node_modules
        target: /app/node_modules
    
    environment:
      # Development environment variables
      - FAST_REFRESH=true
      - CHOKIDAR_USEPOLLING=true
      - WATCHPACK_POLLING=true
      - WDS_SOCKET_HOST=localhost
      - WDS_SOCKET_PORT=3000
    
    # Development-specific command
    command: >
      bash -c "
        echo 'Starting development server with hot reloading...' &&
        npm run start:dev
      "
    
    # Development debugging
    stdin_open: true
    tty: true

  backend:
    build:
      target: development
      args:
        INSTALL_DEV_TOOLS: "true"
    
    volumes:
      - type: bind
        source: ./backend
        target: /app
        consistency: cached
      
      - type: volume
        source: backend_node_modules
        target: /app/node_modules
      
      # Development tools volumes
      - type: bind
        source: ./backend/.vscode
        target: /app/.vscode
        consistency: cached
    
    environment:
      - NODE_ENV=development
      - DEBUG=app:*
      - NODE_OPTIONS=--inspect=0.0.0.0:9229
    
    # Enable debugging and hot reloading
    command: >
      bash -c "
        echo 'Installing development dependencies...' &&
        npm install &&
        echo 'Starting backend with debugging enabled...' &&
        npm run dev:debug
      "
    
    # Expose debugging port
    ports:
      - "9229:9229"
    
    stdin_open: true
    tty: true

  # Development database with sample data
  postgres:
    volumes:
      # Add development seed data
      - ./database/dev-seed-data:/docker-entrypoint-initdb.d/seed:ro
    
    environment:
      # Development-friendly settings
      - POSTGRES_LOG_STATEMENT=all
      - POSTGRES_LOG_DURATION=on

  # File watcher for automated tasks
  file-watcher:
    image: node:18-alpine
    container_name: devops_file_watcher
    working_dir: /workspace
    volumes:
      - ./:/workspace:cached
    command: >
      sh -c "
        npm install -g nodemon chokidar-cli &&
        echo 'Starting file watcher for automated tasks...' &&
        chokidar '**/*.js' '**/*.json' -c 'echo File changed: {path}' --initial
      "
    profiles: ["dev-tools"]

  # Code quality tools
  linter:
    image: node:18-alpine
    container_name: devops_linter
    working_dir: /workspace
    volumes:
      - ./:/workspace:ro
    command: >
      sh -c "
        npm install -g eslint prettier &&
        echo 'Running code quality checks...' &&
        eslint . --ext .js,.jsx,.ts,.tsx &&
        prettier --check '**/*.{js,jsx,ts,tsx,json,css,md}'
      "
    profiles: ["quality", "ci"]

  # Testing service
  test-runner:
    build:
      context: .
      dockerfile: Dockerfile.test
    container_name: devops_test_runner
    volumes:
      - ./:/workspace:cached
      - test_coverage:/workspace/coverage
    environment:
      - CI=true
      - NODE_ENV=test
    command: npm run test:watch
    profiles: ["testing"]

volumes:
  test_coverage:
```

**2.5 Production-Like Environment**

```yaml
# File: docker-compose.production-like.yml
# Production-like environment for testing
version: '3.8'

services:
  frontend:
    build:
      target: production
    
    # Use Nginx for serving static files
    image: nginx:alpine
    volumes:
      - ./frontend/build:/usr/share/nginx/html:ro
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    ports:
      - "80:80"
    
    # Production-like health checks
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  backend:
    build:
      target: production
    
    # Production environment settings
    environment:
      - NODE_ENV=production
      - PORT=3001
      - DATABASE_URL=postgresql://postgres:prodpassword@postgres:5432/devops_app_prod
      - REDIS_URL=redis://redis:6379/0
      - JWT_SECRET=${JWT_SECRET}
      - LOG_LEVEL=info
    
    # No bind mounts in production-like environment
    volumes:
      - app_logs:/app/logs
    
    # Resource limits (like production)
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '0.5'
        reservations:
          memory: 256M
          cpus: '0.25'

  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: devops_app_prod
      POSTGRES_USER: postgres  
      POSTGRES_PASSWORD: prodpassword
    
    # Production-like configuration
    volumes:
      - postgres_prod_data:/var/lib/postgresql/data
      - ./database/postgresql.prod.conf:/etc/postgresql/postgresql.conf:ro
    
    command: ["postgres", "-c", "config_file=/etc/postgresql/postgresql.conf"]
    
    # Resource limits
    deploy:
      resources:
        limits:
          memory: 1G
          cpus: '1.0'

  # Load balancer (simulating production setup)
  nginx-lb:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./nginx/nginx-lb.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - frontend
      - backend
    
    # Load balancer configuration for multiple backend instances
    command: >
      sh -c "
        echo 'Starting load balancer...' &&
        nginx -g 'daemon off;'
      "

volumes:
  app_logs:
  postgres_prod_data:
```

## Deliverables

1. **Comprehensive Development Environment:**
   - Multi-service Docker Compose setup with all application tiers
   - Advanced networking with custom networks and service discovery
   - Data persistence with named volumes and bind mounts
   - Development tools integration (debugging, hot reloading, etc.)

2. **Environment Management:**
   - Environment-specific override files
   - Profile-based service activation
   - Production-like configuration for testing
   - Development workflow optimization

3. **Supporting Infrastructure:**
   - Reverse proxy configuration with Traefik
   - Database management tools (pgAdmin, etc.)
   - Monitoring stack integration (Prometheus, Grafana)
   - Testing infrastructure (Selenium, K6)

4. **Development Scripts and Automation:**
   - Startup scripts for different environments
   - Database seeding and migration scripts
   - Development workflow automation
   - Health check and monitoring integration

## Success Criteria

- Successfully orchestrated multi-service application with Docker Compose
- Implemented hot reloading and debugging capabilities for development
- Created environment-specific configurations with proper overrides
- Demonstrated advanced networking and service communication
- Integrated monitoring, logging, and development tools
- Built production-like environment for realistic testing
- Optimized development workflow with file watching and automation

## Verification Commands

```bash
# Start complete development environment
docker-compose -f docker-compose.dev.yml up -d

# Start with specific profiles
docker-compose --profile monitoring --profile testing up -d

# Test service communication
docker-compose exec backend curl http://frontend:3000/health
docker-compose exec frontend curl http://backend:3001/api/status

# Check logs across services
docker-compose logs -f backend frontend

# Scale services
docker-compose up -d --scale backend=3

# Production-like environment testing
docker-compose -f docker-compose.production-like.yml up -d

# Environment cleanup
docker-compose down -v
```

## Integration with Next Exercise

This Docker Compose foundation prepares for:
- Container orchestration with Kubernetes
- Production deployment strategies
- Service mesh and networking concepts
- Monitoring and observability integration
- CI/CD pipeline integration with containerized applications

## Real-World Applications

**Enterprise Development Scenarios:**
- **Microservices Development**: Complex multi-service application development
- **Team Onboarding**: Consistent development environment across team members
- **Integration Testing**: Realistic multi-service testing environments
- **Performance Testing**: Load testing with complete application stack
- **Production Simulation**: Testing production-like configurations locally

This comprehensive Docker Compose exercise provides 180 minutes of intensive hands-on experience with sophisticated container orchestration and development environment management!
# Exercise 7: Docker Compose Microservices Architecture
**Duration: 240 minutes (4 hours)**  
**Difficulty: Advanced**  
**Prerequisites: Exercises 1-6**

## Learning Objectives
By completing this exercise, you will:
- Design and implement a complete microservices architecture with Docker Compose
- Master advanced Docker Compose patterns for production-like deployments
- Implement service discovery and inter-service communication
- Configure advanced networking, load balancing, and health checks
- Implement comprehensive logging, monitoring, and observability
- Handle database migrations and data consistency in distributed systems

## Scenario
You're architecting a complete e-commerce microservices platform using Docker Compose. The system needs to handle multiple services, ensure data consistency, and provide comprehensive observability across all components.

## Part 1: Microservices Architecture Design (60 minutes)

### Task 1.1: Design Service Architecture
Create a comprehensive microservices architecture with proper service boundaries.

**Instructions:**
1. **Create the overall architecture documentation**
   ```bash
   mkdir -p microservices-platform/{services,infrastructure,monitoring,docs}
   cd microservices-platform
   
   cat > docs/architecture.md << 'EOF'
   # E-Commerce Microservices Architecture

   ## Service Overview
   ```
   ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
   │   API Gateway   │    │  Load Balancer  │    │   Web Frontend  │
   │   (nginx-proxy) │    │     (nginx)     │    │     (React)     │
   └─────────────────┘    └─────────────────┘    └─────────────────┘
            │                       │                       │
            └───────────────────────┼───────────────────────┘
                                   │
         ┌─────────────────────────┼─────────────────────────┐
         │                         │                         │
   ┌──────────┐  ┌──────────────┐  │  ┌─────────────┐  ┌──────────────┐
   │   User   │  │   Product    │  │  │   Order     │  │   Payment    │
   │ Service  │  │   Service    │  │  │  Service    │  │   Service    │
   └──────────┘  └──────────────┘  │  └─────────────┘  └──────────────┘
         │              │          │         │               │
         │              │          │         │               │
   ┌──────────┐  ┌──────────────┐  │  ┌─────────────┐  ┌──────────────┐
   │  Users   │  │   Products   │  │  │   Orders    │  │   Payments   │
   │Database  │  │  Database    │  │  │  Database   │  │   Database   │
   │(postgres)│  │ (postgres)   │  │  │(postgres)   │  │ (postgres)   │
   └──────────┘  └──────────────┘  │  └─────────────┘  └──────────────┘
                                   │
         ┌─────────────────────────┼─────────────────────────┐
         │                         │                         │
   ┌──────────┐  ┌──────────────┐  │  ┌─────────────┐  ┌──────────────┐
   │   Redis  │  │  Elasticsearch │ │  │  Prometheus │  │    Grafana   │
   │  Cache   │  │    Search      │  │  │ Monitoring  │  │ Dashboards   │
   └──────────┘  └──────────────┘  │  └─────────────┘  └──────────────┘
                                   │
                           ┌───────────────┐
                           │   Jaeger      │
                           │   Tracing     │
                           └───────────────┘
   ```

   ## Service Responsibilities

   ### API Gateway (nginx-proxy)
   - Request routing and load balancing
   - Authentication and authorization
   - Rate limiting and throttling
   - SSL termination

   ### User Service
   - User registration and authentication
   - User profile management  
   - JWT token generation and validation

   ### Product Service
   - Product catalog management
   - Product search and filtering
   - Inventory management
   - Product recommendations

   ### Order Service
   - Order creation and management
   - Order status tracking
   - Order history and reporting
   - Saga pattern for distributed transactions

   ### Payment Service
   - Payment processing
   - Payment method management
   - Transaction history
   - Fraud detection

   ## Data Consistency Strategy
   - **Eventual Consistency**: Between services using event-driven architecture
   - **Strong Consistency**: Within service boundaries using database transactions
   - **Saga Pattern**: For distributed transactions across services
   EOF
   ```

2. **Create service communication contracts**
   ```bash
   mkdir -p services/contracts
   
   cat > services/contracts/user-service.json << 'EOF'
   {
     "service": "user-service",
     "version": "1.0.0",
     "endpoints": {
       "POST /api/users/register": {
         "description": "Register new user",
         "request": {
           "email": "string",
           "password": "string", 
           "firstName": "string",
           "lastName": "string"
         },
         "response": {
           "userId": "string",
           "email": "string",
           "token": "string"
         }
       },
       "POST /api/users/login": {
         "description": "User login",
         "request": {
           "email": "string",
           "password": "string"
         },
         "response": {
           "userId": "string",
           "token": "string",
           "expiresIn": "number"
         }
       },
       "GET /api/users/{userId}": {
         "description": "Get user profile",
         "headers": {
           "Authorization": "Bearer {token}"
         },
         "response": {
           "userId": "string",
           "email": "string",
           "firstName": "string",
           "lastName": "string",
           "createdAt": "string"
         }
       }
     },
     "events": {
       "user.registered": {
         "userId": "string",
         "email": "string",
         "timestamp": "string"
       },
       "user.updated": {
         "userId": "string",
         "changes": "object",
         "timestamp": "string"
       }
     }
   }
   EOF
   
   cat > services/contracts/product-service.json << 'EOF'
   {
     "service": "product-service",
     "version": "1.0.0",
     "endpoints": {
       "GET /api/products": {
         "description": "Get products with pagination and filtering",
         "query": {
           "page": "number",
           "limit": "number",
           "category": "string",
           "search": "string"
         },
         "response": {
           "products": "array",
           "totalCount": "number",
           "page": "number",
           "totalPages": "number"
         }
       },
       "GET /api/products/{productId}": {
         "description": "Get product details",
         "response": {
           "productId": "string",
           "name": "string",
           "description": "string",
           "price": "number",
           "inventory": "number",
           "category": "string"
         }
       },
       "PUT /api/products/{productId}/inventory": {
         "description": "Update product inventory",
         "headers": {
           "Authorization": "Bearer {token}"
         },
         "request": {
           "quantity": "number",
           "operation": "add|subtract|set"
         }
       }
     },
     "events": {
       "product.inventory.updated": {
         "productId": "string",
         "previousQuantity": "number",
         "newQuantity": "number",
         "timestamp": "string"
       }
     }
   }
   EOF
   ```

### Task 1.2: Implement Service Base Structure
Create the foundation for all microservices with shared patterns.

**Instructions:**
1. **Create shared Docker base image**
   ```bash
   cat > services/Dockerfile.base << 'EOF'
   FROM node:18-alpine AS base

   # Install dumb-init for proper signal handling
   RUN apk add --no-cache dumb-init

   # Create app directory
   WORKDIR /app

   # Create non-root user
   RUN addgroup -g 1001 -S nodejs
   RUN adduser -S nodeapp -u 1001

   # Copy package files
   COPY package*.json ./

   # Install dependencies
   FROM base AS dependencies
   RUN npm ci --only=production && npm cache clean --force

   # Development dependencies
   FROM base AS dev-dependencies  
   RUN npm ci

   # Production build
   FROM dependencies AS production
   COPY --chown=nodeapp:nodejs . .

   # Health check
   HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
     CMD curl -f http://localhost:${PORT:-3000}/health || exit 1

   USER nodeapp

   ENTRYPOINT ["dumb-init", "--"]
   CMD ["node", "server.js"]
   EOF
   ```

2. **Create shared service utilities**
   ```bash
   mkdir -p services/shared
   
   cat > services/shared/middleware.js << 'EOF'
   const jwt = require('jsonwebtoken');
   const rateLimit = require('express-rate-limit');

   // Authentication middleware
   const authenticate = (req, res, next) => {
     const token = req.header('Authorization')?.replace('Bearer ', '');
     
     if (!token) {
       return res.status(401).json({ error: 'Access denied. No token provided.' });
     }

     try {
       const decoded = jwt.verify(token, process.env.JWT_SECRET);
       req.user = decoded;
       next();
     } catch (error) {
       res.status(400).json({ error: 'Invalid token.' });
     }
   };

   // Rate limiting middleware
   const createRateLimit = (windowMs = 15 * 60 * 1000, max = 100) => {
     return rateLimit({
       windowMs,
       max,
       message: 'Too many requests from this IP, please try again later.',
       standardHeaders: true,
       legacyHeaders: false,
     });
   };

   // Request logging middleware
   const requestLogger = (req, res, next) => {
     const start = Date.now();
     
     res.on('finish', () => {
       const duration = Date.now() - start;
       console.log({
         method: req.method,
         url: req.url,
         status: res.statusCode,
         duration: `${duration}ms`,
         userAgent: req.get('User-Agent'),
         ip: req.ip,
         timestamp: new Date().toISOString()
       });
     });
     
     next();
   };

   // Error handling middleware
   const errorHandler = (err, req, res, next) => {
     console.error({
       error: err.message,
       stack: err.stack,
       url: req.url,
       method: req.method,
       timestamp: new Date().toISOString()
     });

     // Don't leak error details in production
     const message = process.env.NODE_ENV === 'production' 
       ? 'Internal server error' 
       : err.message;

     res.status(err.status || 500).json({
       error: message,
       requestId: req.headers['x-request-id']
     });
   };

   module.exports = {
     authenticate,
     createRateLimit,
     requestLogger,
     errorHandler
   };
   EOF
   
   cat > services/shared/database.js << 'EOF'
   const { Pool } = require('pg');

   class Database {
     constructor() {
       this.pool = new Pool({
         host: process.env.DB_HOST,
         port: process.env.DB_PORT || 5432,
         database: process.env.DB_NAME,
         user: process.env.DB_USER,
         password: process.env.DB_PASSWORD,
         max: 20,
         idleTimeoutMillis: 30000,
         connectionTimeoutMillis: 2000,
       });

       // Handle pool errors
       this.pool.on('error', (err) => {
         console.error('Unexpected error on idle client', err);
       });
     }

     async query(text, params) {
       const start = Date.now();
       const client = await this.pool.connect();
       
       try {
         const result = await client.query(text, params);
         const duration = Date.now() - start;
         
         console.log({
           query: text,
           duration: `${duration}ms`,
           rows: result.rowCount
         });
         
         return result;
       } finally {
         client.release();
       }
     }

     async transaction(queries) {
       const client = await this.pool.connect();
       
       try {
         await client.query('BEGIN');
         
         const results = [];
         for (const { text, params } of queries) {
           const result = await client.query(text, params);
           results.push(result);
         }
         
         await client.query('COMMIT');
         return results;
       } catch (error) {
         await client.query('ROLLBACK');
         throw error;
       } finally {
         client.release();
       }
     }

     async close() {
       await this.pool.end();
     }
   }

   module.exports = Database;
   EOF
   ```

## Part 2: Docker Compose Advanced Configuration (60 minutes)

### Task 2.1: Advanced Docker Compose Setup
Set up advanced Docker Compose configuration with proper networking and service discovery.

**Instructions:**
1. **Create environment configuration**
   ```bash
   mkdir -p infrastructure/compose
   
   cat > infrastructure/compose/setup.sh << 'EOF'
   #!/bin/bash
   set -euo pipefail

   echo "🚀 Setting up Docker Compose environment..."

   # Create custom networks
   echo "📡 Creating custom networks..."
   docker network create frontend-network --driver bridge || echo "frontend-network already exists"
   docker network create backend-network --driver bridge || echo "backend-network already exists"
   docker network create database-network --driver bridge || echo "database-network already exists"
   docker network create monitoring-network --driver bridge || echo "monitoring-network already exists"

   # Create environment files for secrets
   echo "🔐 Creating environment files..."
   mkdir -p secrets
   echo "POSTGRES_PASSWORD=postgres_password_123" > secrets/.env.postgres
   echo "JWT_SECRET=jwt_secret_key_456" > secrets/.env.jwt
   echo "REDIS_PASSWORD=redis_password_789" > secrets/.env.redis

   # Create nginx configuration for API Gateway
   echo "⚙️ Creating nginx configuration..."
   mkdir -p configs/nginx
   cat > configs/nginx/nginx.conf << 'NGINX_EOF'
   upstream user_service {
       server user-service:3001 max_fails=3 fail_timeout=30s;
   }
   
   upstream product_service {
       server product-service:3002 max_fails=3 fail_timeout=30s;
   }
   
   upstream order_service {
       server order-service:3003 max_fails=3 fail_timeout=30s;
   }
   
   upstream payment_service {
       server payment-service:3004 max_fails=3 fail_timeout=30s;
   }

   # Rate limiting
   limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

   server {
       listen 80;
       server_name api.localhost;
       
       # Rate limiting
       limit_req zone=api burst=20 nodelay;
       
       # Health check endpoint
       location /health {
           access_log off;
           return 200 "healthy\n";
           add_header Content-Type text/plain;
       }
       
       location /api/users {
           proxy_pass http://user_service;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
           proxy_connect_timeout 30s;
           proxy_send_timeout 30s;
           proxy_read_timeout 30s;
       }
       
       location /api/products {
           proxy_pass http://product_service;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
       }
       
       location /api/orders {
           proxy_pass http://order_service;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
       }
       
       location /api/payments {
           proxy_pass http://payment_service;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
       }
   }
   NGINX_EOF

   echo "✅ Docker Compose environment setup completed"
   EOF
   
   chmod +x infrastructure/compose/setup.sh
   ```

2. **Create comprehensive Docker Compose file**
   ```yaml
   # docker-compose.yml
   version: '3.8'

   services:
     # API Gateway
     api-gateway:
       image: nginx:alpine
       volumes:
         - ./configs/nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
       ports:
         - "80:80"
         - "443:443"
       networks:
         - frontend-network
         - backend-network
       restart: unless-stopped
       mem_limit: 128m
       mem_reservation: 64m
       healthcheck:
         test: ["CMD", "curl", "-f", "http://localhost/health"]
         interval: 30s
         timeout: 10s
         retries: 3
         start_period: 10s
       depends_on:
         - user-service
         - product-service
         - order-service
         - payment-service

     # User Service (scaled with docker-compose up --scale user-service=3)
     user-service:
       image: microservices-platform/user-service:latest
       environment:
         - NODE_ENV=production
         - PORT=3001
         - DB_HOST=user-db
         - DB_NAME=users
         - DB_USER=postgres
         - JWT_SECRET=jwt_secret_key_456
         - POSTGRES_PASSWORD=postgres_password_123
       env_file:
         - ./secrets/.env.postgres
         - ./secrets/.env.jwt
       networks:
         - backend-network
         - database-network
       restart: unless-stopped
       mem_limit: 512m
       mem_reservation: 256m
       healthcheck:
         test: ["CMD", "curl", "-f", "http://localhost:3001/health"]
         interval: 30s
         timeout: 10s
         retries: 3
         start_period: 40s
       depends_on:
         user-db:
           condition: service_healthy

     # Product Service (scaled with docker-compose up --scale product-service=4)
     product-service:
       image: microservices-platform/product-service:latest
       environment:
         - NODE_ENV=production
         - PORT=3002
         - DB_HOST=product-db
         - DB_NAME=products
         - DB_USER=postgres
         - REDIS_HOST=redis-cache
       env_file:
         - ./secrets/.env.postgres
         - ./secrets/.env.redis
       networks:
         - backend-network
         - database-network
       restart: unless-stopped
       mem_limit: 512m
       mem_reservation: 256m
       healthcheck:
         test: ["CMD", "curl", "-f", "http://localhost:3002/health"]
         interval: 30s
         timeout: 10s
         retries: 3
         start_period: 40s
       depends_on:
         product-db:
           condition: service_healthy
         redis-cache:
           condition: service_healthy

     # Order Service (scaled with docker-compose up --scale order-service=3)
     order-service:
       image: microservices-platform/order-service:latest
       environment:
         - NODE_ENV=production
         - PORT=3003
         - DB_HOST=order-db
         - DB_NAME=orders
         - DB_USER=postgres
         - USER_SERVICE_URL=http://user-service:3001
         - PRODUCT_SERVICE_URL=http://product-service:3002
         - PAYMENT_SERVICE_URL=http://payment-service:3004
       env_file:
         - ./secrets/.env.postgres
       networks:
         - backend-network
         - database-network
       restart: unless-stopped
       mem_limit: 512m
       mem_reservation: 256m
       healthcheck:
         test: ["CMD", "curl", "-f", "http://localhost:3003/health"]
         interval: 30s
         timeout: 10s
         retries: 3
         start_period: 40s
       depends_on:
         order-db:
           condition: service_healthy
         user-service:
           condition: service_healthy
         product-service:
           condition: service_healthy

     # Payment Service (scaled with docker-compose up --scale payment-service=2)
     payment-service:
       image: microservices-platform/payment-service:latest
       environment:
         - NODE_ENV=production
         - PORT=3004
         - DB_HOST=payment-db
         - DB_NAME=payments
         - DB_USER=postgres
       env_file:
         - ./secrets/.env.postgres
       networks:
         - backend-network
         - database-network
       restart: unless-stopped
       mem_limit: 512m
       mem_reservation: 256m
       healthcheck:
         test: ["CMD", "curl", "-f", "http://localhost:3004/health"]
         interval: 30s
         timeout: 10s
         retries: 3
         start_period: 40s
       depends_on:
         payment-db:
           condition: service_healthy

     # Databases
     user-db:
       image: postgres:15-alpine
       environment:
         - POSTGRES_DB=users
         - POSTGRES_USER=postgres
         - POSTGRES_PASSWORD=postgres_password_123
       env_file:
         - ./secrets/.env.postgres
       volumes:
         - user-db-data:/var/lib/postgresql/data
         - ./configs/postgres/init-user-db.sql:/docker-entrypoint-initdb.d/init.sql:ro
       networks:
         - database-network
       restart: unless-stopped
       mem_limit: 512m
       mem_reservation: 256m
       healthcheck:
         test: ["CMD-SHELL", "pg_isready -U postgres -d users"]
         interval: 10s
         timeout: 5s
         retries: 5
         start_period: 30s

     product-db:
       image: postgres:15-alpine
       environment:
         - POSTGRES_DB=products
         - POSTGRES_USER=postgres
         - POSTGRES_PASSWORD=postgres_password_123
       env_file:
         - ./secrets/.env.postgres
       volumes:
         - product-db-data:/var/lib/postgresql/data
         - ./configs/postgres/init-product-db.sql:/docker-entrypoint-initdb.d/init.sql:ro
       networks:
         - database-network
       restart: unless-stopped
       mem_limit: 512m
       mem_reservation: 256m
       healthcheck:
         test: ["CMD-SHELL", "pg_isready -U postgres -d products"]
         interval: 10s
         timeout: 5s
         retries: 5
         start_period: 30s

     order-db:
       image: postgres:15-alpine
       environment:
         - POSTGRES_DB=orders
         - POSTGRES_USER=postgres
         - POSTGRES_PASSWORD=postgres_password_123
       env_file:
         - ./secrets/.env.postgres
       volumes:
         - order-db-data:/var/lib/postgresql/data
         - ./configs/postgres/init-order-db.sql:/docker-entrypoint-initdb.d/init.sql:ro
       networks:
         - database-network
       restart: unless-stopped
       mem_limit: 512m
       mem_reservation: 256m
       healthcheck:
         test: ["CMD-SHELL", "pg_isready -U postgres -d orders"]
         interval: 10s
         timeout: 5s
         retries: 5
         start_period: 30s

     payment-db:
       image: postgres:15-alpine
       environment:
         - POSTGRES_DB=payments
         - POSTGRES_USER=postgres
         - POSTGRES_PASSWORD=postgres_password_123
       env_file:
         - ./secrets/.env.postgres
       volumes:
         - payment-db-data:/var/lib/postgresql/data
         - ./configs/postgres/init-payment-db.sql:/docker-entrypoint-initdb.d/init.sql:ro
       networks:
         - database-network
       restart: unless-stopped
       mem_limit: 512m
       mem_reservation: 256m
       healthcheck:
         test: ["CMD-SHELL", "pg_isready -U postgres -d payments"]
         interval: 10s
         timeout: 5s
         retries: 5
         start_period: 30s

     # Cache
     redis-cache:
       image: redis:7-alpine
       command: redis-server --requirepass redis_password_789
       env_file:
         - ./secrets/.env.redis
       networks:
         - backend-network
       restart: unless-stopped
       mem_limit: 256m
       mem_reservation: 128m
       healthcheck:
         test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
         interval: 10s
         timeout: 3s
         retries: 5
         start_period: 20s

     # Monitoring
     prometheus:
       image: prom/prometheus:latest
       command:
         - '--config.file=/etc/prometheus/prometheus.yml'
         - '--storage.tsdb.path=/prometheus'
         - '--web.console.libraries=/etc/prometheus/console_libraries'
         - '--web.console.templates=/etc/prometheus/consoles'
         - '--web.enable-lifecycle'
       volumes:
         - ./monitoring/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
         - ./monitoring/prometheus/alert_rules.yml:/etc/prometheus/alert_rules.yml:ro
         - prometheus-data:/prometheus
       ports:
         - "9090:9090"
       networks:
         - monitoring-network
         - backend-network
       restart: unless-stopped
       healthcheck:
         test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:9090/-/healthy"]
         interval: 30s
         timeout: 10s
         retries: 3
         start_period: 30s

     grafana:
       image: grafana/grafana:latest
       environment:
         - GF_SECURITY_ADMIN_PASSWORD=admin123
         - GF_USERS_ALLOW_SIGN_UP=false
       volumes:
         - grafana-data:/var/lib/grafana
         - ./monitoring/grafana/dashboards:/etc/grafana/provisioning/dashboards:ro
       ports:
         - "3000:3000"
       networks:
         - monitoring-network
       restart: unless-stopped
       healthcheck:
         test: ["CMD-SHELL", "curl -f http://localhost:3000/api/health || exit 1"]
         interval: 30s
         timeout: 10s
         retries: 3
         start_period: 30s
       depends_on:
         - prometheus

   volumes:
     user-db-data:
       driver: local
     product-db-data:
       driver: local
     order-db-data:
       driver: local
     payment-db-data:
       driver: local
     prometheus-data:
       driver: local
     grafana-data:
       driver: local

   networks:
     frontend-network:
       driver: bridge
       external: true
     backend-network:
       driver: bridge
       external: true
     database-network:
       driver: bridge
       external: true
     monitoring-network:
       driver: bridge
       external: true
   ```

### Task 2.2: Docker Compose Deployment and Scaling
Deploy services using Docker Compose with proper scaling and resource management.

**Instructions:**
1. **Create deployment script**
   ```bash
   cat > infrastructure/compose/deploy.sh << 'EOF'
   #!/bin/bash
   set -euo pipefail

   echo "🚀 Deploying microservices with Docker Compose..."

   # Setup environment
   echo "⚙️ Setting up environment..."
   ./infrastructure/compose/setup.sh

   # Build services (if needed)
   echo "🔧 Building services..."
   docker-compose build

   # Deploy the services
   echo "📦 Starting services..."
   docker-compose up -d

   # Scale services for high availability
   echo "📈 Scaling services..."
   docker-compose up -d --scale user-service=3 --scale product-service=4 --scale order-service=3 --scale payment-service=2

   # Wait for services to be ready
   echo "⏳ Waiting for services to be ready..."
   sleep 30

   # Check service status
   echo "📊 Service status:"
   docker-compose ps

   echo "🔍 Checking service health..."
   docker-compose ps --services | while read service; do
     echo "Checking $service..."
     docker-compose ps $service
   done

   echo "✅ Deployment completed!"
   echo "🌐 API Gateway available at: http://localhost"
   echo "📊 Prometheus available at: http://localhost:9090"
   echo "📈 Grafana available at: http://localhost:3000 (admin/admin123)"
   EOF
   
   chmod +x infrastructure/compose/deploy.sh
   ```

2. **Create scaling script**
   ```bash
   cat > infrastructure/compose/scale.sh << 'EOF'
   #!/bin/bash
   set -euo pipefail

   SERVICE=$1
   REPLICAS=$2

   if [[ -z "$SERVICE" || -z "$REPLICAS" ]]; then
     echo "Usage: $0 <service-name> <replica-count>"
     echo "Available services:"
     docker-compose config --services
     echo ""
     echo "Current scaling:"
     docker-compose ps
     exit 1
   fi

   echo "🔄 Scaling $SERVICE to $REPLICAS replicas..."

   # Scale the service
   docker-compose up -d --scale ${SERVICE}=${REPLICAS}

   # Wait and show status
   sleep 10
   echo "📊 Current status:"
   docker-compose ps $SERVICE

   echo "✅ Scaling completed!"
   EOF
   
   chmod +x infrastructure/compose/scale.sh
   ```

## Part 3: Service Communication and Integration (60 minutes)

### Task 3.1: Implement Service-to-Service Communication
Create robust inter-service communication with circuit breakers and retries.

**Instructions:**
1. **Create service communication library**
   ```bash
   cat > services/shared/service-client.js << 'EOF'
   const axios = require('axios');

   class ServiceClient {
     constructor(baseURL, options = {}) {
       this.baseURL = baseURL;
       this.timeout = options.timeout || 5000;
       this.retries = options.retries || 3;
       this.retryDelay = options.retryDelay || 1000;
       
       // Circuit breaker state
       this.circuitBreaker = {
         failures: 0,
         threshold: options.circuitBreakerThreshold || 5,
         timeout: options.circuitBreakerTimeout || 60000,
         state: 'CLOSED', // CLOSED, OPEN, HALF_OPEN
         nextAttempt: 0
       };
       
       this.client = axios.create({
         baseURL: this.baseURL,
         timeout: this.timeout,
         headers: {
           'Content-Type': 'application/json',
           'X-Service-Name': process.env.SERVICE_NAME || 'unknown'
         }
       });

       // Request interceptor for correlation ID
       this.client.interceptors.request.use((config) => {
         config.headers['X-Correlation-ID'] = this.generateCorrelationId();
         config.headers['X-Request-Start'] = Date.now();
         return config;
       });

       // Response interceptor for logging and metrics
       this.client.interceptors.response.use(
         (response) => {
           const duration = Date.now() - response.config.headers['X-Request-Start'];
           this.logRequest(response.config, response, duration);
           this.recordSuccess();
           return response;
         },
         (error) => {
           const duration = error.config ? Date.now() - error.config.headers['X-Request-Start'] : 0;
           this.logRequest(error.config, error.response, duration, error);
           this.recordFailure();
           throw error;
         }
       );
     }

     async request(config) {
       // Check circuit breaker
       if (this.circuitBreaker.state === 'OPEN') {
         if (Date.now() < this.circuitBreaker.nextAttempt) {
           throw new Error('Circuit breaker is OPEN');
         }
         this.circuitBreaker.state = 'HALF_OPEN';
       }

       let lastError;
       
       for (let attempt = 1; attempt <= this.retries; attempt++) {
         try {
           const response = await this.client.request(config);
           
           // Reset circuit breaker on success
           if (this.circuitBreaker.state === 'HALF_OPEN') {
             this.circuitBreaker.state = 'CLOSED';
             this.circuitBreaker.failures = 0;
           }
           
           return response;
         } catch (error) {
           lastError = error;
           
           // Don't retry on 4xx errors (except 429)
           if (error.response && error.response.status >= 400 && error.response.status < 500 && error.response.status !== 429) {
             break;
           }
           
           // Wait before retry (exponential backoff)
           if (attempt < this.retries) {
             const delay = this.retryDelay * Math.pow(2, attempt - 1);
             await this.sleep(delay);
           }
         }
       }
       
       throw lastError;
     }

     recordSuccess() {
       this.circuitBreaker.failures = Math.max(0, this.circuitBreaker.failures - 1);
     }

     recordFailure() {
       this.circuitBreaker.failures++;
       
       if (this.circuitBreaker.failures >= this.circuitBreaker.threshold) {
         this.circuitBreaker.state = 'OPEN';
         this.circuitBreaker.nextAttempt = Date.now() + this.circuitBreaker.timeout;
       }
     }

     generateCorrelationId() {
       return `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
     }

     logRequest(config, response, duration, error) {
       const logEntry = {
         method: config?.method?.toUpperCase(),
         url: config?.url,
         status: response?.status || error?.code,
         duration: `${duration}ms`,
         correlationId: config?.headers['X-Correlation-ID'],
         timestamp: new Date().toISOString()
       };

       if (error) {
         logEntry.error = error.message;
         console.error('Service request failed:', logEntry);
       } else {
         console.log('Service request completed:', logEntry);
       }
     }

     sleep(ms) {
       return new Promise(resolve => setTimeout(resolve, ms));
     }

     // Convenience methods
     get(url, config = {}) {
       return this.request({ method: 'GET', url, ...config });
     }

     post(url, data, config = {}) {
       return this.request({ method: 'POST', url, data, ...config });
     }

     put(url, data, config = {}) {
       return this.request({ method: 'PUT', url, data, ...config });
     }

     delete(url, config = {}) {
       return this.request({ method: 'DELETE', url, ...config });
     }
   }

   module.exports = ServiceClient;
   EOF
   ```

2. **Create event-driven communication**
   ```bash
   cat > services/shared/event-bus.js << 'EOF'
   const Redis = require('redis');

   class EventBus {
     constructor() {
       this.subscriber = Redis.createClient({
         host: process.env.REDIS_HOST || 'redis-cache',
         port: process.env.REDIS_PORT || 6379,
         password: process.env.REDIS_PASSWORD
       });

       this.publisher = Redis.createClient({
         host: process.env.REDIS_HOST || 'redis-cache',
         port: process.env.REDIS_PORT || 6379,
         password: process.env.REDIS_PASSWORD
       });

       this.handlers = new Map();
       
       this.subscriber.on('message', (channel, message) => {
         this.handleMessage(channel, message);
       });
     }

     async connect() {
       await Promise.all([
         this.subscriber.connect(),
         this.publisher.connect()
       ]);
       console.log('Event bus connected');
     }

     async disconnect() {
       await Promise.all([
         this.subscriber.disconnect(),
         this.publisher.disconnect()
       ]);
       console.log('Event bus disconnected');
     }

     async publish(eventType, data) {
       const event = {
         type: eventType,
         data,
         timestamp: new Date().toISOString(),
         service: process.env.SERVICE_NAME,
         correlationId: this.generateCorrelationId()
       };

       const message = JSON.stringify(event);
       
       try {
         await this.publisher.publish(eventType, message);
         console.log('Event published:', { type: eventType, correlationId: event.correlationId });
       } catch (error) {
         console.error('Failed to publish event:', error);
         throw error;
       }
     }

     async subscribe(eventType, handler) {
       if (!this.handlers.has(eventType)) {
         this.handlers.set(eventType, []);
         await this.subscriber.subscribe(eventType);
       }
       
       this.handlers.get(eventType).push(handler);
       console.log(`Subscribed to event: ${eventType}`);
     }

     async handleMessage(channel, message) {
       try {
         const event = JSON.parse(message);
         const handlers = this.handlers.get(channel) || [];
         
         console.log('Event received:', { 
           type: event.type, 
           correlationId: event.correlationId,
           handlersCount: handlers.length 
         });

         // Process handlers concurrently
         const promises = handlers.map(async (handler) => {
           try {
             await handler(event);
           } catch (error) {
             console.error('Event handler failed:', {
               eventType: event.type,
               correlationId: event.correlationId,
               error: error.message
             });
           }
         });

         await Promise.allSettled(promises);
       } catch (error) {
         console.error('Failed to process event message:', error);
       }
     }

     generateCorrelationId() {
       return `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
     }
   }

   module.exports = EventBus;
   EOF
   ```

### Task 3.2: Implement Distributed Tracing
Add comprehensive distributed tracing across all services.

**Instructions:**
1. **Create tracing infrastructure**
   ```bash
   mkdir -p monitoring/jaeger
   
   cat > monitoring/jaeger/docker-compose.jaeger.yml << 'EOF'
   version: '3.8'

   services:
     jaeger:
       image: jaegertracing/all-in-one:latest
       ports:
         - "16686:16686"
         - "14268:14268"
         - "6831:6831/udp"
         - "6832:6832/udp"
       environment:
         - COLLECTOR_ZIPKIN_HOST_PORT=:9411
       networks:
         - monitoring-network
       deploy:
         replicas: 1
         placement:
           constraints:
             - node.role == manager

   networks:
     monitoring-network:
       external: true
   EOF
   ```

2. **Create tracing middleware**
   ```bash
   cat > services/shared/tracing.js << 'EOF'
   const opentelemetry = require('@opentelemetry/api');
   const { NodeTracerProvider } = require('@opentelemetry/sdk-node');
   const { JaegerExporter } = require('@opentelemetry/exporter-jaeger');
   const { Resource } = require('@opentelemetry/resources');
   const { SemanticResourceAttributes } = require('@opentelemetry/semantic-conventions');
   const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');

   class TracingService {
     constructor(serviceName) {
       this.serviceName = serviceName;
       this.tracer = null;
       this.init();
     }

     init() {
       const provider = new NodeTracerProvider({
         resource: new Resource({
           [SemanticResourceAttributes.SERVICE_NAME]: this.serviceName,
           [SemanticResourceAttributes.SERVICE_VERSION]: process.env.SERVICE_VERSION || '1.0.0',
         }),
         instrumentations: [getNodeAutoInstrumentations()]
       });

       const exporter = new JaegerExporter({
         endpoint: process.env.JAEGER_ENDPOINT || 'http://jaeger:14268/api/traces',
       });

       provider.addSpanProcessor(
         new opentelemetry.BatchSpanProcessor(exporter)
       );

       provider.register();

       this.tracer = opentelemetry.trace.getTracer(this.serviceName);
       
       console.log(`Tracing initialized for ${this.serviceName}`);
     }

     // Express middleware
     middleware() {
       return (req, res, next) => {
         const span = this.tracer.startSpan(`${req.method} ${req.path}`);
         
         // Add request attributes
         span.setAttributes({
           'http.method': req.method,
           'http.url': req.url,
           'http.path': req.path,
           'user.id': req.user?.id,
           'correlation.id': req.headers['x-correlation-id']
         });

         // Add span to request context
         req.span = span;
         req.traceId = span.spanContext().traceId;

         // End span when response finishes
         res.on('finish', () => {
           span.setAttributes({
             'http.status_code': res.statusCode,
             'http.response.size': res.get('content-length')
           });
           
           if (res.statusCode >= 400) {
             span.setStatus({
               code: opentelemetry.SpanStatusCode.ERROR,
               message: `HTTP ${res.statusCode}`
             });
           }
           
           span.end();
         });

         next();
       };
     }

     // Create child span
     createSpan(name, parentSpan) {
       const context = parentSpan ? opentelemetry.trace.setSpan(opentelemetry.context.active(), parentSpan) : undefined;
       return this.tracer.startSpan(name, {}, context);
     }

     // Trace database operations
     traceDatabase(operation, query, params = []) {
       return async (span) => {
         span.setAttributes({
           'db.operation': operation,
           'db.statement': query,
           'db.params.count': params.length
         });

         try {
           const result = await operation();
           span.setAttributes({
             'db.rows.affected': result.rowCount || 0
           });
           return result;
         } catch (error) {
           span.setStatus({
             code: opentelemetry.SpanStatusCode.ERROR,
             message: error.message
           });
           throw error;
         }
       };
     }

     // Trace service calls
     traceServiceCall(serviceName, method, url) {
       return (span) => {
         span.setAttributes({
           'service.name': serviceName,
           'http.method': method,
           'http.url': url
         });

         return async (serviceCall) => {
           try {
             const result = await serviceCall();
             span.setAttributes({
               'http.status_code': result.status
             });
             return result;
           } catch (error) {
             span.setStatus({
               code: opentelemetry.SpanStatusCode.ERROR,
               message: error.message
             });
             throw error;
           }
         };
       };
     }
   }

   module.exports = TracingService;
   EOF
   ```

## Part 4: Comprehensive Monitoring and Observability (60 minutes)

### Task 4.1: Implement Application Metrics
Create comprehensive metrics collection for all services.

**Instructions:**
1. **Create metrics collection**
   ```bash
   cat > services/shared/metrics.js << 'EOF'
   const promClient = require('prom-client');

   class MetricsService {
     constructor(serviceName) {
       this.serviceName = serviceName;
       this.register = new promClient.Registry();
       
       // Add default metrics
       promClient.collectDefaultMetrics({
         register: this.register,
         prefix: `${serviceName}_`
       });

       this.initCustomMetrics();
     }

     initCustomMetrics() {
       // HTTP request metrics
       this.httpRequestDuration = new promClient.Histogram({
         name: `${this.serviceName}_http_request_duration_seconds`,
         help: 'Duration of HTTP requests in seconds',
         labelNames: ['method', 'route', 'status_code'],
         buckets: [0.1, 0.3, 0.5, 0.7, 1, 3, 5, 7, 10]
       });

       this.httpRequestTotal = new promClient.Counter({
         name: `${this.serviceName}_http_requests_total`,
         help: 'Total number of HTTP requests',
         labelNames: ['method', 'route', 'status_code']
       });

       // Database metrics
       this.databaseQueryDuration = new promClient.Histogram({
         name: `${this.serviceName}_database_query_duration_seconds`,
         help: 'Duration of database queries in seconds',
         labelNames: ['operation', 'table'],
         buckets: [0.01, 0.05, 0.1, 0.3, 0.5, 1, 2, 5]
       });

       this.databaseQueryTotal = new promClient.Counter({
         name: `${this.serviceName}_database_queries_total`,
         help: 'Total number of database queries',
         labelNames: ['operation', 'table', 'status']
       });

       // Service call metrics
       this.serviceCallDuration = new promClient.Histogram({
         name: `${this.serviceName}_service_call_duration_seconds`,
         help: 'Duration of service calls in seconds',
         labelNames: ['target_service', 'method', 'status_code'],
         buckets: [0.1, 0.3, 0.5, 0.7, 1, 3, 5, 7, 10]
       });

       this.serviceCallTotal = new promClient.Counter({
         name: `${this.serviceName}_service_calls_total`,
         help: 'Total number of service calls',
         labelNames: ['target_service', 'method', 'status_code']
       });

       // Business metrics
       this.businessEventTotal = new promClient.Counter({
         name: `${this.serviceName}_business_events_total`,
         help: 'Total number of business events',
         labelNames: ['event_type', 'status']
       });

       this.activeConnections = new promClient.Gauge({
         name: `${this.serviceName}_active_connections`,
         help: 'Number of active connections'
       });

       // Register all metrics
       this.register.registerMetric(this.httpRequestDuration);
       this.register.registerMetric(this.httpRequestTotal);
       this.register.registerMetric(this.databaseQueryDuration);
       this.register.registerMetric(this.databaseQueryTotal);
       this.register.registerMetric(this.serviceCallDuration);
       this.register.registerMetric(this.serviceCallTotal);
       this.register.registerMetric(this.businessEventTotal);
       this.register.registerMetric(this.activeConnections);
     }

     // Express middleware for HTTP metrics
     httpMiddleware() {
       return (req, res, next) => {
         const start = Date.now();

         res.on('finish', () => {
           const duration = (Date.now() - start) / 1000;
           const labels = {
             method: req.method,
             route: req.route?.path || req.path,
             status_code: res.statusCode
           };

           this.httpRequestDuration.observe(labels, duration);
           this.httpRequestTotal.inc(labels);
         });

         next();
       };
     }

     // Track database operations
     trackDatabaseQuery(operation, table, duration, success = true) {
       const labels = { operation, table };
       this.databaseQueryDuration.observe(labels, duration / 1000);
       this.databaseQueryTotal.inc({ 
         ...labels, 
         status: success ? 'success' : 'error' 
       });
     }

     // Track service calls
     trackServiceCall(targetService, method, statusCode, duration) {
       const labels = {
         target_service: targetService,
         method,
         status_code: statusCode
       };

       this.serviceCallDuration.observe(labels, duration / 1000);
       this.serviceCallTotal.inc(labels);
     }

     // Track business events
     trackBusinessEvent(eventType, status = 'success') {
       this.businessEventTotal.inc({ event_type: eventType, status });
     }

     // Set active connections
     setActiveConnections(count) {
       this.activeConnections.set(count);
     }

     // Get metrics endpoint handler
     getMetricsHandler() {
       return async (req, res) => {
         res.set('Content-Type', this.register.contentType);
         res.send(await this.register.metrics());
       };
     }
   }

   module.exports = MetricsService;
   EOF
   ```

2. **Create Prometheus configuration**
   ```bash
   mkdir -p monitoring/prometheus
   
   cat > monitoring/prometheus/prometheus.yml << 'EOF'
   global:
     scrape_interval: 15s
     evaluation_interval: 15s

   rule_files:
     - "alert_rules.yml"

   alerting:
     alertmanagers:
       - static_configs:
           - targets:
               - alertmanager:9093

   scrape_configs:
     # Prometheus itself
     - job_name: 'prometheus'
       static_configs:
         - targets: ['localhost:9090']

     # Microservices
     - job_name: 'user-service'
       static_configs:
         - targets: ['user-service:3001']
       metrics_path: '/metrics'
       scrape_interval: 10s

     - job_name: 'product-service'
       static_configs:
         - targets: ['product-service:3002']
       metrics_path: '/metrics'
       scrape_interval: 10s

     - job_name: 'order-service'
       static_configs:
         - targets: ['order-service:3003']
       metrics_path: '/metrics'
       scrape_interval: 10s

     - job_name: 'payment-service'
       static_configs:
         - targets: ['payment-service:3004']
       metrics_path: '/metrics'
       scrape_interval: 10s

     # Infrastructure
     - job_name: 'node-exporter'
       static_configs:
         - targets: ['node-exporter:9100']

     - job_name: 'postgres-exporter'
       static_configs:
         - targets: ['postgres-exporter:9187']

     - job_name: 'redis-exporter'
       static_configs:
         - targets: ['redis-exporter:9121']
   EOF
   
   cat > monitoring/prometheus/alert_rules.yml << 'EOF'
   groups:
     - name: microservices-alerts
       rules:
         # High error rate alert
         - alert: HighErrorRate
           expr: |
             (
               sum(rate(http_requests_total{status_code=~"5.."}[5m])) by (service)
               /
               sum(rate(http_requests_total[5m])) by (service)
             ) * 100 > 5
           for: 2m
           labels:
             severity: warning
           annotations:
             summary: "High error rate detected"
             description: "Service {{ $labels.service }} has error rate of {{ $value }}% for the last 5 minutes"

         # High response time alert
         - alert: HighResponseTime
           expr: |
             histogram_quantile(0.95, 
               sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
             ) > 1
           for: 3m
           labels:
             severity: warning
           annotations:
             summary: "High response time detected"
             description: "Service {{ $labels.service }} 95th percentile response time is {{ $value }}s"

         # Service down alert
         - alert: ServiceDown
           expr: up == 0
           for: 1m
           labels:
             severity: critical
           annotations:
             summary: "Service is down"
             description: "Service {{ $labels.job }} has been down for more than 1 minute"

         # High CPU usage
         - alert: HighCpuUsage
           expr: |
             100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
           for: 5m
           labels:
             severity: warning
           annotations:
             summary: "High CPU usage detected"
             description: "Instance {{ $labels.instance }} CPU usage is {{ $value }}%"

         # High memory usage
         - alert: HighMemoryUsage
           expr: |
             (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
           for: 5m
           labels:
             severity: warning
           annotations:
             summary: "High memory usage detected"
             description: "Instance {{ $labels.instance }} memory usage is {{ $value }}%"

         # Database connection issues
         - alert: DatabaseConnectionIssues
           expr: |
             increase(database_queries_total{status="error"}[5m]) > 10
           for: 2m
           labels:
             severity: critical
           annotations:
             summary: "Database connection issues"
             description: "Service {{ $labels.service }} has {{ $value }} database errors in the last 5 minutes"
   EOF
   ```

### Task 4.2: Create Comprehensive Dashboards
Build Grafana dashboards for monitoring the entire microservices platform.

**Instructions:**
1. **Create Grafana dashboard configurations**
   ```bash
   mkdir -p monitoring/grafana/dashboards
   
   cat > monitoring/grafana/dashboards/microservices-overview.json << 'EOF'
   {
     "dashboard": {
       "id": null,
       "title": "Microservices Overview",
       "tags": ["microservices", "overview"],
       "timezone": "browser",
       "refresh": "30s",
       "time": {
         "from": "now-1h",
         "to": "now"
       },
       "panels": [
         {
           "id": 1,
           "title": "Request Rate",
           "type": "stat",
           "targets": [
             {
               "expr": "sum(rate(http_requests_total[5m]))",
               "legendFormat": "Total RPS"
             }
           ],
           "fieldConfig": {
             "defaults": {
               "unit": "reqps",
               "color": {
                 "mode": "thresholds"
               },
               "thresholds": {
                 "steps": [
                   {"color": "green", "value": null},
                   {"color": "yellow", "value": 100},
                   {"color": "red", "value": 500}
                 ]
               }
             }
           }
         },
         {
           "id": 2,
           "title": "Error Rate",
           "type": "stat",
           "targets": [
             {
               "expr": "sum(rate(http_requests_total{status_code=~\"5..\"}[5m])) / sum(rate(http_requests_total[5m])) * 100",
               "legendFormat": "Error Rate"
             }
           ],
           "fieldConfig": {
             "defaults": {
               "unit": "percent",
               "color": {
                 "mode": "thresholds"
               },
               "thresholds": {
                 "steps": [
                   {"color": "green", "value": null},
                   {"color": "yellow", "value": 1},
                   {"color": "red", "value": 5}
                 ]
               }
             }
           }
         },
         {
           "id": 3,
           "title": "Response Time (95th percentile)",
           "type": "timeseries",
           "targets": [
             {
               "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service))",
               "legendFormat": "{{ service }}"
             }
           ],
           "fieldConfig": {
             "defaults": {
               "unit": "s",
               "custom": {
                 "drawStyle": "line",
                 "lineInterpolation": "linear",
                 "spanNulls": false
               }
             }
           }
         },
         {
           "id": 4,
           "title": "Service Health",
           "type": "table",
           "targets": [
             {
               "expr": "up",
               "legendFormat": "{{ job }}"
             }
           ],
           "transformations": [
             {
               "id": "organize",
               "options": {
                 "excludeByName": {},
                 "indexByName": {},
                 "renameByName": {
                   "job": "Service",
                   "Value": "Status"
                 }
               }
             }
           ]
         }
       ]
     }
   }
   EOF
   
   cat > monitoring/grafana/dashboards/service-details.json << 'EOF'
   {
     "dashboard": {
       "id": null,
       "title": "Service Details",
       "tags": ["microservices", "details"],
       "timezone": "browser",
       "refresh": "30s",
       "templating": {
         "list": [
           {
             "name": "service",
             "type": "query",
             "query": "label_values(http_requests_total, service)",
             "refresh": 1,
             "includeAll": false,
             "multi": false
           }
         ]
       },
       "panels": [
         {
           "id": 1,
           "title": "Request Rate by Endpoint",
           "type": "timeseries",
           "targets": [
             {
               "expr": "sum(rate(http_requests_total{service=\"$service\"}[5m])) by (route)",
               "legendFormat": "{{ route }}"
             }
           ]
         },
         {
           "id": 2,
           "title": "Response Time Distribution",
           "type": "heatmap",
           "targets": [
             {
               "expr": "sum(rate(http_request_duration_seconds_bucket{service=\"$service\"}[5m])) by (le)",
               "legendFormat": "{{ le }}"
             }
           ]
         },
         {
           "id": 3,
           "title": "Database Query Performance",
           "type": "timeseries",
           "targets": [
             {
               "expr": "histogram_quantile(0.95, sum(rate(database_query_duration_seconds_bucket{service=\"$service\"}[5m])) by (le, operation))",
               "legendFormat": "{{ operation }} (95th)"
             },
             {
               "expr": "histogram_quantile(0.50, sum(rate(database_query_duration_seconds_bucket{service=\"$service\"}[5m])) by (le, operation))",
               "legendFormat": "{{ operation }} (50th)"
             }
           ]
         },
         {
           "id": 4,
           "title": "Service Dependencies",
           "type": "timeseries",
           "targets": [
             {
               "expr": "sum(rate(service_calls_total{service=\"$service\"}[5m])) by (target_service)",
               "legendFormat": "{{ target_service }}"
             }
           ]
         }
       ]
     }
   }
   EOF
   ```

2. **Create deployment health monitoring**
   ```bash
   cat > monitoring/health-check.sh << 'EOF'
   #!/bin/bash
   set -euo pipefail

   echo "🏥 Starting comprehensive health check..."

   # Colors for output
   RED='\033[0;31m'
   GREEN='\033[0;32m'
   YELLOW='\033[1;33m'
   NC='\033[0m' # No Color

   # Function to check service health
   check_service_health() {
     local service_name=$1
     local health_url=$2
     local max_attempts=5
     local attempt=1

     while [[ $attempt -le $max_attempts ]]; do
       if curl -f -s "$health_url" > /dev/null; then
         echo -e "${GREEN}✅ $service_name is healthy${NC}"
         return 0
       else
         echo -e "${YELLOW}⏳ $service_name health check failed (attempt $attempt/$max_attempts)${NC}"
         ((attempt++))
         sleep 5
       fi
     done

     echo -e "${RED}❌ $service_name is unhealthy after $max_attempts attempts${NC}"
     return 1
   }

   # Function to check Docker service status
   check_docker_service() {
     local service_name=$1
     local replicas=$(docker service ps $service_name --filter desired-state=running --format "table {{.Name}}" | wc -l)
     local desired=$(docker service ls --filter name=$service_name --format "{{.Replicas}}" | cut -d'/' -f2)
     
     if [[ $replicas -eq $desired ]]; then
       echo -e "${GREEN}✅ $service_name: $replicas/$desired replicas running${NC}"
       return 0
     else
       echo -e "${RED}❌ $service_name: $replicas/$desired replicas running${NC}"
       return 1
     fi
   }

   # Health check results
   failed_checks=0

   echo "🔍 Checking Docker Swarm services..."
   services=("microservices_user-service" "microservices_product-service" "microservices_order-service" "microservices_payment-service" "microservices_api-gateway")

   for service in "${services[@]}"; do
     if ! check_docker_service $service; then
       ((failed_checks++))
     fi
   done

   echo ""
   echo "🌐 Checking service endpoints..."

   # Check API Gateway
   if ! check_service_health "API Gateway" "http://localhost/health"; then
     ((failed_checks++))
   fi

   # Check individual services (if accessible directly)
   endpoints=(
     "User Service:http://localhost:3001/health"
     "Product Service:http://localhost:3002/health"
     "Order Service:http://localhost:3003/health"
     "Payment Service:http://localhost:3004/health"
   )

   for endpoint in "${endpoints[@]}"; do
     IFS=':' read -r name url <<< "$endpoint"
     if ! check_service_health "$name" "$url"; then
       ((failed_checks++))
     fi
   done

   echo ""
   echo "📊 Checking monitoring services..."

   # Check Prometheus
   if ! check_service_health "Prometheus" "http://localhost:9090/-/healthy"; then
     ((failed_checks++))
   fi

   # Check Grafana
   if ! check_service_health "Grafana" "http://localhost:3000/api/health"; then
     ((failed_checks++))
   fi

   echo ""
   echo "🗄️ Checking databases..."

   # Check database connectivity through services
   db_services=("user-db" "product-db" "order-db" "payment-db")
   for db in "${db_services[@]}"; do
     if docker service ps microservices_$db --filter desired-state=running | grep -q "Running"; then
       echo -e "${GREEN}✅ $db is running${NC}"
     else
       echo -e "${RED}❌ $db is not running${NC}"
       ((failed_checks++))
     fi
   done

   echo ""
   echo "📈 Health check summary:"
   if [[ $failed_checks -eq 0 ]]; then
     echo -e "${GREEN}🎉 All systems are healthy!${NC}"
     exit 0
   else
     echo -e "${RED}⚠️  $failed_checks checks failed${NC}"
     echo ""
     echo "🔧 Troubleshooting steps:"
     echo "1. Check service logs: docker service logs <service-name>"
     echo "2. Check service status: docker service ps <service-name>"
     echo "3. Check network connectivity: docker network ls"
     echo "4. Check resource usage: docker stats"
     exit 1
   fi
   EOF
   
   chmod +x monitoring/health-check.sh
   ```

## Exercise Summary

**Congratulations!** You have successfully implemented a comprehensive microservices architecture with Docker Compose including:

✅ **Microservices Architecture Design**
- Complete service boundaries and communication contracts
- Shared base infrastructure and utilities
- Event-driven architecture patterns

✅ **Advanced Docker Compose Configuration**
- Multi-network service deployment
- Environment-based configuration management
- Service scaling and resource management
- High availability database configuration with health checks

✅ **Service Communication and Integration**
- Circuit breaker patterns with retry logic
- Event-driven inter-service communication
- Distributed tracing with OpenTelemetry and Jaeger
- Correlation ID tracking across services

✅ **Comprehensive Observability**
- Application metrics with Prometheus
- Custom business metrics and SLA monitoring
- Grafana dashboards for visualization
- Automated health checking and alerting

### Key Achievements
- **Scalability**: Horizontal scaling with Docker Compose
- **Resilience**: Circuit breakers, retries, and graceful degradation
- **Observability**: Full-stack monitoring with metrics, logs, and traces
- **Security**: Environment-based secrets management and network isolation
- **Performance**: Load balancing and resource optimization

### Architecture Benefits
- **High Availability**: Multi-replica deployment with health checks
- **Fault Tolerance**: Service isolation and automatic recovery
- **Performance**: Distributed caching and optimized queries
- **Maintainability**: Clear service boundaries and contracts
- **Monitoring**: Complete observability across the platform

**Total Time Investment: 240 minutes (4 hours)**
**Skills Developed: Microservices architecture, Advanced Docker Compose, Service communication, Distributed tracing, Production monitoring**
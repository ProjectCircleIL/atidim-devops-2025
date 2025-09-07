# Exercise 1: Application Containerization

**Duration:** 90 minutes  
**Difficulty:** Intermediate  
**Type:** Hands-on Implementation

## Objective
Containerize your 3-tier application from Module 2 by creating optimized Dockerfiles for frontend, backend, and database components.

## Prerequisites
- Completed Module 2 exercises (working Git repository with application code)
- Docker Desktop installed and running
- Understanding of your application's technology stack
- Basic knowledge of Dockerfile syntax

## Scenario
Your team wants to move from local development to containerized deployment. You need to containerize each tier of your application while maintaining development workflows and preparing for production deployment.

## Application Structure Review
From Module 2, you should have:
```
your-project/
├── frontend/          # React, Vue, or similar
├── backend/           # Node.js, Python, or similar  
├── database/          # SQL scripts and migrations
├── .github/workflows/ # CI/CD pipelines
└── docs/             # Project documentation
```

## Tasks

### Task 1: Backend Containerization (30 minutes)

**1.1 Create Backend Dockerfile**

Create `backend/Dockerfile`:

```dockerfile
# Multi-stage build for Node.js backend example
# Adjust for your technology stack (Python, Java, etc.)

# Build stage
FROM node:18-alpine AS builder

# Set working directory
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies (including dev dependencies for building)
RUN npm ci --only=production && npm cache clean --force

# Development stage
FROM node:18-alpine AS development

WORKDIR /app

# Install dumb-init for proper signal handling
RUN apk add --no-cache dumb-init

# Create non-root user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodejs -u 1001

# Copy package files
COPY package*.json ./

# Install all dependencies (including dev)
RUN npm ci && npm cache clean --force

# Copy source code
COPY --chown=nodejs:nodejs . .

# Switch to non-root user
USER nodejs

# Expose port
EXPOSE 3001

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:3001/api/health || exit 1

# Use dumb-init for proper signal handling
ENTRYPOINT ["dumb-init", "--"]

# Start development server
CMD ["npm", "run", "dev"]

# Production stage
FROM node:18-alpine AS production

WORKDIR /app

# Install dumb-init
RUN apk add --no-cache dumb-init curl

# Create non-root user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodejs -u 1001

# Copy built dependencies from builder stage
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --chown=nodejs:nodejs package*.json ./

# Copy source code
COPY --chown=nodejs:nodejs . .

# Switch to non-root user
USER nodejs

# Expose port
EXPOSE 3001

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:3001/api/health || exit 1

# Use dumb-init for proper signal handling
ENTRYPOINT ["dumb-init", "--"]

# Start production server
CMD ["npm", "start"]
```

**1.2 Create .dockerignore for Backend**

Create `backend/.dockerignore`:
```
node_modules
npm-debug.log
.env
.env.local
.env.production
.git
.gitignore
README.md
Dockerfile
.dockerignore
coverage
.coverage
.nyc_output
tests
__tests__
test
spec
*.test.js
*.spec.js
.cache
dist
build
```

**1.3 Update Backend Package.json**

Ensure your `backend/package.json` has proper scripts:
```json
{
  "scripts": {
    "start": "node src/server.js",
    "dev": "nodemon src/server.js",
    "test": "jest",
    "build": "echo 'No build step required for Node.js'"
  }
}
```

### Task 2: Frontend Containerization (30 minutes)

**2.1 Create Frontend Dockerfile**

Create `frontend/Dockerfile`:

```dockerfile
# Multi-stage build for React frontend
# Adjust build tools for Vue, Angular, etc.

# Build stage
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --silent

# Copy source code
COPY . .

# Build the application
RUN npm run build

# Production stage with Nginx
FROM nginx:alpine AS production

# Install curl for health checks
RUN apk add --no-cache curl

# Copy built application
COPY --from=builder /app/build /usr/share/nginx/html

# Copy custom Nginx configuration
COPY nginx.conf /etc/nginx/nginx.conf

# Create non-root user for Nginx
RUN addgroup -g 1001 -S nginx && \
    adduser -S nginx -u 1001 -G nginx

# Change ownership of nginx directories
RUN chown -R nginx:nginx /var/cache/nginx && \
    chown -R nginx:nginx /var/log/nginx && \
    chown -R nginx:nginx /etc/nginx/conf.d

RUN touch /var/run/nginx.pid && \
    chown -R nginx:nginx /var/run/nginx.pid

# Switch to non-root user
USER nginx

# Expose port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8080 || exit 1

# Start Nginx
CMD ["nginx", "-g", "daemon off;"]

# Development stage
FROM node:18-alpine AS development

WORKDIR /app

# Create non-root user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodejs -u 1001

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci

# Copy source code
COPY --chown=nodejs:nodejs . .

# Switch to non-root user  
USER nodejs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:3000 || exit 1

# Start development server
CMD ["npm", "start"]
```

**2.2 Create Nginx Configuration**

Create `frontend/nginx.conf`:
```nginx
events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    
    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;
    add_header Content-Security-Policy "default-src 'self' http: https: data: blob: 'unsafe-inline'" always;
    
    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css text/xml text/javascript application/javascript application/xml+rss application/json;
    
    server {
        listen 8080;
        server_name localhost;
        root /usr/share/nginx/html;
        index index.html index.htm;
        
        # Handle client-side routing
        location / {
            try_files $uri $uri/ /index.html;
        }
        
        # API proxy (adjust URL for your backend)
        location /api {
            proxy_pass http://backend:3001;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_cache_bypass $http_upgrade;
        }
        
        # Static assets caching
        location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
        }
        
        # Health check endpoint
        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
    }
}
```

**2.3 Create Frontend .dockerignore**

Create `frontend/.dockerignore`:
```
node_modules
build
.git
.gitignore
README.md
Dockerfile
.dockerignore
.env.local
.env.development.local
.env.test.local
.env.production.local
npm-debug.log*
yarn-debug.log*
yarn-error.log*
coverage
.DS_Store
```

### Task 3: Database Containerization (30 minutes)

**3.1 Create Database Setup**

Create `database/Dockerfile`:
```dockerfile
FROM postgres:13-alpine

# Install additional tools
RUN apk add --no-cache curl

# Create database initialization directory
COPY init-scripts/ /docker-entrypoint-initdb.d/

# Copy custom configuration
COPY postgresql.conf /usr/local/share/postgresql/postgresql.conf.sample

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD pg_isready -U ${POSTGRES_USER:-postgres} -d ${POSTGRES_DB:-postgres} || exit 1

# Expose port
EXPOSE 5432

# Use official PostgreSQL entrypoint
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["postgres", "-c", "config_file=/usr/local/share/postgresql/postgresql.conf.sample"]
```

**3.2 Create Database Initialization Scripts**

Create `database/init-scripts/01-create-database.sql`:
```sql
-- Create application database
CREATE DATABASE devops_app;

-- Create user for application
CREATE USER app_user WITH ENCRYPTED PASSWORD 'app_password';
GRANT ALL PRIVILEGES ON DATABASE devops_app TO app_user;

-- Connect to the application database
\c devops_app;

-- Create application tables
CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Create indexes
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username ON users(username);

-- Grant permissions to app user
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO app_user;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO app_user;
```

Create `database/init-scripts/02-seed-data.sql`:
```sql
-- Connect to application database
\c devops_app;

-- Insert sample data
INSERT INTO users (username, email, password_hash) VALUES
('admin', 'admin@example.com', '$2b$10$example_hash_here_admin'),
('testuser', 'test@example.com', '$2b$10$example_hash_here_test'),
('demo', 'demo@example.com', '$2b$10$example_hash_here_demo')
ON CONFLICT (username) DO NOTHING;

-- Verify data
SELECT COUNT(*) as user_count FROM users;
```

**3.3 Create PostgreSQL Configuration**

Create `database/postgresql.conf`:
```
# PostgreSQL configuration for containerized deployment
listen_addresses = '*'
port = 5432
max_connections = 100
shared_buffers = 128MB
dynamic_shared_memory_type = posix
max_wal_size = 1GB
min_wal_size = 80MB
log_timezone = 'UTC'
datestyle = 'iso, mdy'
timezone = 'UTC'
lc_messages = 'en_US.utf8'
lc_monetary = 'en_US.utf8'
lc_numeric = 'en_US.utf8'
lc_time = 'en_US.utf8'
default_text_search_config = 'pg_catalog.english'

# Logging
log_statement = 'error'
log_min_duration_statement = 1000
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
```

## Deliverables

1. **Dockerfiles**: Working Dockerfiles for all three tiers:
   - `backend/Dockerfile` with multi-stage build
   - `frontend/Dockerfile` with Nginx production setup
   - `database/Dockerfile` with PostgreSQL and initialization

2. **Configuration Files**:
   - `.dockerignore` files for each component
   - `nginx.conf` for frontend proxy configuration
   - Database initialization and configuration scripts

3. **Built Images**: Successfully built Docker images:
   ```bash
   docker build -t my-app-backend:dev backend/
   docker build -t my-app-frontend:dev frontend/
   docker build -t my-app-database:dev database/
   ```

4. **Testing Documentation**: Instructions for testing each container individually

## Success Criteria

- All three Docker images build successfully without errors
- Containers start and run without immediate crashes
- Health checks pass for all containers
- Images follow security best practices (non-root users, minimal attack surface)
- Multi-stage builds optimize for both development and production
- Proper signal handling and graceful shutdowns

## Verification Steps

```bash
# Build all images
docker build -t my-app-backend:dev --target development backend/
docker build -t my-app-frontend:dev --target development frontend/
docker build -t my-app-database:dev database/

# Test backend container
docker run --rm -p 3001:3001 my-app-backend:dev &
curl http://localhost:3001/api/health
docker stop $(docker ps -q --filter ancestor=my-app-backend:dev)

# Test database container
docker run --rm -p 5432:5432 -e POSTGRES_PASSWORD=testpass my-app-database:dev &
docker exec -it $(docker ps -q --filter ancestor=my-app-database:dev) pg_isready
docker stop $(docker ps -q --filter ancestor=my-app-database:dev)

# Test frontend container (production build)
docker build -t my-app-frontend:prod --target production frontend/
docker run --rm -p 8080:8080 my-app-frontend:prod &
curl http://localhost:8080/health
docker stop $(docker ps -q --filter ancestor=my-app-frontend:prod)

# Check image sizes
docker images | grep my-app
```

## Integration with Next Exercise

These Dockerfiles will be used in Exercise 2 for:
- Multi-stage build optimization
- Environment-specific configurations
- Docker Compose integration
- Production deployment preparation

## Troubleshooting

**Common Issues:**
- **Build failures**: Check syntax and file paths in Dockerfiles
- **Permission errors**: Ensure proper user/group setup and file ownership
- **Network issues**: Verify port configurations and firewall settings
- **Health check failures**: Test endpoints manually before containerizing

**Debugging Commands:**
```bash
# Build with detailed output
docker build --progress=plain -t my-app:debug .

# Run container with shell access
docker run -it --entrypoint /bin/sh my-app:debug

# Check container logs
docker logs container_id

# Inspect container configuration
docker inspect container_id
```

## Advanced Challenges

1. **Multi-Architecture Builds**: Create images for both AMD64 and ARM64
2. **Distroless Images**: Use distroless base images for production
3. **Build Optimization**: Minimize image layers and optimize caching
4. **Security Scanning**: Integrate Trivy or similar tools for vulnerability scanning
5. **Custom Base Images**: Create and maintain organization-specific base images

This containerization foundation prepares your application for orchestration with Docker Compose and eventual Kubernetes deployment!
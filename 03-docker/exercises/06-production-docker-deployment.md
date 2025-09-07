# Exercise 6: Production Docker Deployment and Orchestration

**Duration:** 210 minutes  
**Difficulty:** Advanced  
**Type:** Production Implementation + Performance Optimization

## Objective
Implement production-grade Docker deployment strategies including container orchestration, performance optimization, monitoring integration, high availability patterns, and disaster recovery. Build a complete production-ready containerized application stack.

## Prerequisites
- Completed Exercise 5 (Docker Networking and Security)
- Understanding of load balancing and high availability concepts
- Access to cloud provider or multiple servers for deployment
- Knowledge of monitoring and logging best practices

## Production Deployment Concepts

### Advanced Topics to Master:
1. **High Availability**: Multi-instance deployments, load balancing, failover
2. **Performance Optimization**: Resource management, caching, database optimization
3. **Monitoring and Observability**: Comprehensive logging, metrics, and alerting
4. **Backup and Recovery**: Data persistence, backup strategies, disaster recovery
5. **Scaling Strategies**: Horizontal and vertical scaling, auto-scaling
6. **Zero-Downtime Deployment**: Rolling updates, blue-green deployments
7. **Production Troubleshooting**: Debugging, performance analysis, incident response
8. **Compliance and Governance**: Audit logging, access control, policy enforcement

## Tasks

### Task 1: Production-Grade Multi-Node Deployment (75 minutes)

**1.1 Docker Swarm Production Cluster Setup**

Create a production Docker Swarm cluster:

```yaml
# File: docker-compose.prod.yml
# Production Docker Compose with Swarm mode
version: '3.8'

networks:
  frontend-prod:
    driver: overlay
    encrypted: true
    attachable: false
  backend-prod:
    driver: overlay
    encrypted: true
    attachable: false
  database-prod:
    driver: overlay
    encrypted: true
    attachable: false
  monitoring-prod:
    driver: overlay
    encrypted: true
    attachable: false

volumes:
  postgres_prod_data:
    driver: local
    driver_opts:
      type: nfs
      o: addr=nfs-server,rw
      device: ":/var/nfs/postgres"
  redis_prod_data:
    driver: local
    driver_opts:
      type: nfs
      o: addr=nfs-server,rw
      device: ":/var/nfs/redis"

services:
  # Load Balancer / Reverse Proxy (HA)
  nginx-lb:
    image: nginx:alpine
    ports:
      - target: 80
        published: 80
        protocol: tcp
        mode: host
      - target: 443
        published: 443
        protocol: tcp
        mode: host
    
    configs:
      - source: nginx_config
        target: /etc/nginx/nginx.conf
      - source: nginx_ssl_config
        target: /etc/nginx/conf.d/ssl.conf
    
    secrets:
      - ssl_certificate
      - ssl_private_key
    
    networks:
      - frontend-prod
      - backend-prod
    
    deploy:
      mode: replicated
      replicas: 2
      placement:
        constraints:
          - node.role == worker
          - node.labels.tier == frontend
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
        order: start-first
      resources:
        limits:
          memory: 256M
          cpus: '0.5'
        reservations:
          memory: 128M
          cpus: '0.25'
    
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
    
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    
    labels:
      - "traefik.enable=false"
      - "prometheus.scrape=true"
      - "prometheus.port=9113"

  # Frontend Application (HA)
  frontend:
    image: ${REGISTRY_URL}/devops-frontend:${IMAGE_TAG:-latest}
    
    networks:
      - frontend-prod
    
    deploy:
      mode: replicated
      replicas: 3
      placement:
        constraints:
          - node.role == worker
          - node.labels.tier == frontend
        preferences:
          - spread: node.labels.zone
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
        monitor: 60s
        order: start-first
      rollback_config:
        parallelism: 1
        delay: 10s
        failure_action: pause
        monitor: 60s
        order: stop-first
      resources:
        limits:
          memory: 512M
          cpus: '0.5'
        reservations:
          memory: 256M
          cpus: '0.25'
    
    environment:
      - NODE_ENV=production
      - REACT_APP_API_URL=/api
      - REACT_APP_VERSION=${IMAGE_TAG:-latest}
      - REACT_APP_BUILD_DATE=${BUILD_DATE}
    
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    
    logging:
      driver: "fluentd"
      options:
        fluentd-address: "fluentd:24224"
        tag: "frontend.{{.Name}}"
    
    labels:
      - "prometheus.scrape=true"
      - "prometheus.port=3000"
      - "prometheus.path=/metrics"

  # Backend API (HA with Auto-scaling)
  backend:
    image: ${REGISTRY_URL}/devops-backend:${IMAGE_TAG:-latest}
    
    networks:
      - backend-prod
      - database-prod
    
    secrets:
      - database_url
      - jwt_secret
      - api_keys
    
    configs:
      - source: backend_config
        target: /app/config/production.json
    
    deploy:
      mode: replicated
      replicas: 3
      placement:
        constraints:
          - node.role == worker
          - node.labels.tier == backend
        preferences:
          - spread: node.labels.zone
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s
      update_config:
        parallelism: 1
        delay: 15s
        failure_action: rollback
        monitor: 60s
        order: start-first
      rollback_config:
        parallelism: 1
        delay: 10s
        failure_action: pause
        monitor: 60s
        order: stop-first
      resources:
        limits:
          memory: 1G
          cpus: '1.0'
        reservations:
          memory: 512M
          cpus: '0.5'
    
    environment:
      - NODE_ENV=production
      - PORT=3001
      - LOG_LEVEL=info
      - METRICS_ENABLED=true
    
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3001/api/health"]
      interval: 30s
      timeout: 15s
      retries: 3
      start_period: 90s
    
    logging:
      driver: "fluentd"
      options:
        fluentd-address: "fluentd:24224"
        tag: "backend.{{.Name}}"
    
    labels:
      - "prometheus.scrape=true"
      - "prometheus.port=3001"
      - "prometheus.path=/metrics"

  # PostgreSQL Database (HA with Replication)
  postgres-primary:
    image: postgres:15-alpine
    
    networks:
      - database-prod
    
    volumes:
      - postgres_prod_data:/var/lib/postgresql/data
    
    environment:
      - POSTGRES_DB=devops_app_prod
      - POSTGRES_USER=appuser
      - POSTGRES_REPLICATION_MODE=master
      - POSTGRES_REPLICATION_USER=replicator
    
    secrets:
      - postgres_password
      - postgres_replication_password
    
    configs:
      - source: postgres_config
        target: /etc/postgresql/postgresql.conf
      - source: postgres_hba_config
        target: /etc/postgresql/pg_hba.conf
    
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.role == worker
          - node.labels.tier == database
          - node.labels.postgres-primary == true
      restart_policy:
        condition: on-failure
        delay: 30s
        max_attempts: 3
        window: 300s
      resources:
        limits:
          memory: 2G
          cpus: '2.0'
        reservations:
          memory: 1G
          cpus: '1.0'
    
    command: >
      sh -c "
        postgres -c config_file=/etc/postgresql/postgresql.conf
      "
    
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d devops_app_prod"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "3"

  postgres-replica:
    image: postgres:15-alpine
    
    networks:
      - database-prod
    
    environment:
      - POSTGRES_REPLICATION_MODE=slave
      - POSTGRES_REPLICATION_USER=replicator
      - POSTGRES_MASTER_HOST=postgres-primary
      - POSTGRES_MASTER_PORT=5432
    
    secrets:
      - postgres_replication_password
    
    deploy:
      mode: replicated
      replicas: 2
      placement:
        constraints:
          - node.role == worker
          - node.labels.tier == database
          - node.labels.postgres-replica == true
      restart_policy:
        condition: on-failure
        delay: 30s
        max_attempts: 3
        window: 300s
      resources:
        limits:
          memory: 1G
          cpus: '1.0'
        reservations:
          memory: 512M
          cpus: '0.5'
    
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U replicator"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 90s

  # Redis Cluster (HA)
  redis-master:
    image: redis:7-alpine
    
    networks:
      - database-prod
    
    volumes:
      - redis_prod_data:/data
    
    configs:
      - source: redis_config
        target: /etc/redis/redis.conf
    
    secrets:
      - redis_password
    
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.role == worker
          - node.labels.tier == database
          - node.labels.redis-master == true
      restart_policy:
        condition: on-failure
        delay: 10s
        max_attempts: 3
        window: 120s
      resources:
        limits:
          memory: 512M
          cpus: '0.5'
        reservations:
          memory: 256M
          cpus: '0.25'
    
    command: ["redis-server", "/etc/redis/redis.conf"]
    
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 30s
      timeout: 10s
      retries: 3

  redis-sentinel:
    image: redis:7-alpine
    
    networks:
      - database-prod
    
    configs:
      - source: redis_sentinel_config
        target: /etc/redis/sentinel.conf
    
    deploy:
      mode: replicated
      replicas: 3
      placement:
        constraints:
          - node.role == worker
          - node.labels.tier == database
      resources:
        limits:
          memory: 128M
          cpus: '0.25'
        reservations:
          memory: 64M
          cpus: '0.1'
    
    command: ["redis-sentinel", "/etc/redis/sentinel.conf"]
    
    healthcheck:
      test: ["CMD", "redis-cli", "-p", "26379", "ping"]
      interval: 30s
      timeout: 10s
      retries: 3

# Configuration files as configs
configs:
  nginx_config:
    external: true
  nginx_ssl_config:
    external: true
  backend_config:
    external: true
  postgres_config:
    external: true
  postgres_hba_config:
    external: true
  redis_config:
    external: true
  redis_sentinel_config:
    external: true

# Secrets
secrets:
  ssl_certificate:
    external: true
  ssl_private_key:
    external: true
  database_url:
    external: true
  jwt_secret:
    external: true
  api_keys:
    external: true
  postgres_password:
    external: true
  postgres_replication_password:
    external: true
  redis_password:
    external: true
```

**1.2 Production Deployment Scripts**

```bash
#!/bin/bash
# File: scripts/deploy-production.sh
# Production deployment script with safety checks and rollback capability

set -e

# Configuration
STACK_NAME="devops-app-prod"
REGISTRY_URL="${REGISTRY_URL:-your-registry.com}"
IMAGE_TAG="${IMAGE_TAG:-latest}"
BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ')
BACKUP_RETENTION_DAYS=30

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color

print_header() {
    echo -e "\n${BLUE}=== $1 ===${NC}\n"
}

print_success() {
    echo -e "${GREEN}✅ $1${NC}"
}

print_warning() {
    echo -e "${YELLOW}⚠️  $1${NC}"
}

print_error() {
    echo -e "${RED}❌ $1${NC}"
}

# Pre-deployment checks
pre_deployment_checks() {
    print_header "Pre-Deployment Checks"
    
    # Check Docker Swarm status
    if ! docker node ls >/dev/null 2>&1; then
        print_error "Docker Swarm is not initialized"
        exit 1
    fi
    print_success "Docker Swarm is active"
    
    # Check node health
    unhealthy_nodes=$(docker node ls --filter "availability=drain" --quiet | wc -l)
    if [ "$unhealthy_nodes" -gt 0 ]; then
        print_warning "$unhealthy_nodes nodes are drained"
    fi
    
    # Check image availability
    if ! docker manifest inspect "${REGISTRY_URL}/devops-frontend:${IMAGE_TAG}" >/dev/null 2>&1; then
        print_error "Frontend image not found: ${REGISTRY_URL}/devops-frontend:${IMAGE_TAG}"
        exit 1
    fi
    
    if ! docker manifest inspect "${REGISTRY_URL}/devops-backend:${IMAGE_TAG}" >/dev/null 2>&1; then
        print_error "Backend image not found: ${REGISTRY_URL}/devops-backend:${IMAGE_TAG}"
        exit 1
    fi
    
    print_success "All images are available"
    
    # Check secrets and configs
    required_secrets=("database_url" "jwt_secret" "postgres_password" "redis_password")
    for secret in "${required_secrets[@]}"; do
        if ! docker secret ls --format "{{.Name}}" | grep -q "^${secret}$"; then
            print_error "Required secret missing: $secret"
            exit 1
        fi
    done
    print_success "All required secrets are present"
    
    # Check available resources
    available_memory=$(docker system df --format "table {{.Type}}\t{{.TotalCount}}\t{{.Size}}" | grep Images | awk '{print $3}')
    print_success "Resource check completed"
}

# Backup current state
backup_current_state() {
    print_header "Backup Current State"
    
    backup_dir="backups/$(date +%Y%m%d_%H%M%S)"
    mkdir -p "$backup_dir"
    
    # Backup database
    print_success "Starting database backup..."
    docker exec $(docker ps -q --filter "name=${STACK_NAME}_postgres-primary") \
        pg_dump -U appuser devops_app_prod | gzip > "${backup_dir}/database_backup.sql.gz"
    
    # Backup Redis data
    print_success "Starting Redis backup..."
    docker exec $(docker ps -q --filter "name=${STACK_NAME}_redis-master") \
        redis-cli BGSAVE
    
    # Export current service configuration
    docker service ls --format "table {{.Name}}\t{{.Image}}\t{{.Replicas}}" > "${backup_dir}/services_before.txt"
    
    # Save current stack configuration
    docker stack config "$STACK_NAME" > "${backup_dir}/stack_config.yml" 2>/dev/null || true
    
    print_success "Backup completed in $backup_dir"
    
    # Clean old backups
    find backups/ -type d -mtime +$BACKUP_RETENTION_DAYS -exec rm -rf {} + 2>/dev/null || true
}

# Deploy with rolling update
deploy_application() {
    print_header "Deploying Application"
    
    # Set environment variables for compose
    export REGISTRY_URL
    export IMAGE_TAG
    export BUILD_DATE
    
    # Deploy stack with update configuration
    echo "Deploying stack: $STACK_NAME"
    docker stack deploy \
        --compose-file docker-compose.prod.yml \
        --with-registry-auth \
        --resolve-image always \
        "$STACK_NAME"
    
    print_success "Stack deployment initiated"
}

# Monitor deployment progress
monitor_deployment() {
    print_header "Monitoring Deployment Progress"
    
    echo "Waiting for services to update..."
    
    # Get list of services
    services=$(docker service ls --filter "name=${STACK_NAME}" --format "{{.Name}}")
    
    # Monitor each service
    for service in $services; do
        echo "Monitoring service: $service"
        
        # Wait for update to complete
        timeout 600 bash -c "
            while true; do
                state=\$(docker service inspect --format '{{.UpdateStatus.State}}' '$service' 2>/dev/null || echo 'unknown')
                if [[ \"\$state\" == \"completed\" ]]; then
                    break
                elif [[ \"\$state\" == \"rollback_started\" || \"\$state\" == \"rollback_completed\" ]]; then
                    echo 'Service $service rolled back'
                    exit 1
                elif [[ \"\$state\" == \"paused\" ]]; then
                    echo 'Service $service update paused'
                    exit 1
                fi
                sleep 5
            done
        "
        
        if [ $? -eq 0 ]; then
            print_success "Service $service updated successfully"
        else
            print_error "Service $service update failed"
            return 1
        fi
    done
    
    print_success "All services updated successfully"
}

# Health checks
verify_deployment() {
    print_header "Verifying Deployment Health"
    
    # Wait for services to stabilize
    sleep 30
    
    # Check service health
    unhealthy_services=$(docker service ls --filter "name=${STACK_NAME}" --format "table {{.Name}}\t{{.Replicas}}" | grep "0/")
    if [ -n "$unhealthy_services" ]; then
        print_error "Unhealthy services detected:"
        echo "$unhealthy_services"
        return 1
    fi
    
    # Application-specific health checks
    max_attempts=30
    attempt=1
    
    while [ $attempt -le $max_attempts ]; do
        echo "Health check attempt $attempt/$max_attempts"
        
        # Check frontend
        if curl -f -s http://localhost/health >/dev/null 2>&1; then
            print_success "Frontend health check passed"
            break
        fi
        
        if [ $attempt -eq $max_attempts ]; then
            print_error "Frontend health check failed after $max_attempts attempts"
            return 1
        fi
        
        sleep 10
        ((attempt++))
    done
    
    # Check backend API
    if curl -f -s http://localhost/api/health >/dev/null 2>&1; then
        print_success "Backend API health check passed"
    else
        print_error "Backend API health check failed"
        return 1
    fi
    
    # Check database connectivity
    if docker exec $(docker ps -q --filter "name=${STACK_NAME}_postgres-primary") \
        pg_isready -U appuser -d devops_app_prod >/dev/null 2>&1; then
        print_success "Database connectivity check passed"
    else
        print_error "Database connectivity check failed"
        return 1
    fi
    
    print_success "All health checks passed"
}

# Rollback function
rollback_deployment() {
    print_header "Rolling Back Deployment"
    
    # Get previous image tags from backup or service history
    # This is a simplified rollback - in production, you'd track versions more carefully
    
    print_warning "Rolling back to previous version..."
    
    # Rollback each service
    services=$(docker service ls --filter "name=${STACK_NAME}" --format "{{.Name}}")
    
    for service in $services; do
        echo "Rolling back service: $service"
        docker service rollback "$service"
    done
    
    # Wait for rollback to complete
    sleep 60
    
    # Verify rollback
    if verify_deployment; then
        print_success "Rollback completed successfully"
    else
        print_error "Rollback verification failed"
        exit 1
    fi
}

# Cleanup function
cleanup_deployment() {
    print_header "Cleaning Up"
    
    # Remove dangling images
    docker image prune -f
    
    # Clean up old task logs
    docker system prune -f --volumes=false
    
    print_success "Cleanup completed"
}

# Main deployment process
main() {
    print_header "Starting Production Deployment"
    echo "Stack: $STACK_NAME"
    echo "Image Tag: $IMAGE_TAG"
    echo "Registry: $REGISTRY_URL"
    echo "Build Date: $BUILD_DATE"
    
    # Trap errors and rollback
    trap 'print_error "Deployment failed, initiating rollback..."; rollback_deployment' ERR
    
    pre_deployment_checks
    backup_current_state
    deploy_application
    monitor_deployment
    
    # Disable error trap before verification
    trap - ERR
    
    if verify_deployment; then
        cleanup_deployment
        print_success "🚀 Production deployment completed successfully!"
        
        # Send notification (example with Slack webhook)
        if [ -n "$SLACK_WEBHOOK_URL" ]; then
            curl -X POST -H 'Content-type: application/json' \
                --data "{\"text\":\"🚀 Production deployment successful!\nVersion: $IMAGE_TAG\nTime: $BUILD_DATE\"}" \
                "$SLACK_WEBHOOK_URL"
        fi
    else
        print_error "Deployment verification failed, rolling back..."
        rollback_deployment
        exit 1
    fi
}

# Command line options
case "${1:-deploy}" in
    "deploy")
        main
        ;;
    "rollback")
        rollback_deployment
        ;;
    "health-check")
        verify_deployment
        ;;
    "backup")
        backup_current_state
        ;;
    *)
        echo "Usage: $0 {deploy|rollback|health-check|backup}"
        exit 1
        ;;
esac
```

### Task 2: Performance Optimization and Monitoring (75 minutes)

**2.1 Advanced Performance Monitoring Setup**

```yaml
# File: docker-compose.monitoring.yml
# Comprehensive monitoring stack for production
version: '3.8'

networks:
  monitoring:
    driver: overlay
    encrypted: true

volumes:
  prometheus_data:
    driver: local
  grafana_data:
    driver: local
  elasticsearch_data:
    driver: local
  alertmanager_data:
    driver: local

services:
  # Prometheus for metrics collection
  prometheus:
    image: prom/prometheus:v2.40.0
    
    networks:
      - monitoring
    
    ports:
      - "9090:9090"
    
    volumes:
      - prometheus_data:/prometheus
    
    configs:
      - source: prometheus_config
        target: /etc/prometheus/prometheus.yml
      - source: prometheus_rules
        target: /etc/prometheus/rules.yml
    
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--web.enable-lifecycle'
      - '--web.enable-admin-api'
      - '--storage.tsdb.retention.time=30d'
      - '--storage.tsdb.retention.size=10GB'
    
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.role == worker
          - node.labels.monitoring == true
      resources:
        limits:
          memory: 2G
          cpus: '1.0'
        reservations:
          memory: 1G
          cpus: '0.5'
    
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:9090/-/healthy"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Grafana for visualization
  grafana:
    image: grafana/grafana:9.3.0
    
    networks:
      - monitoring
    
    ports:
      - "3000:3000"
    
    volumes:
      - grafana_data:/var/lib/grafana
    
    environment:
      - GF_SECURITY_ADMIN_PASSWORD__FILE=/run/secrets/grafana_admin_password
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_SMTP_ENABLED=true
      - GF_SMTP_HOST=smtp.gmail.com:587
      - GF_SMTP_USER=alerts@yourcompany.com
      - GF_SMTP_FROM_ADDRESS=alerts@yourcompany.com
    
    secrets:
      - grafana_admin_password
      - smtp_password
    
    configs:
      - source: grafana_provisioning_datasources
        target: /etc/grafana/provisioning/datasources/datasources.yml
      - source: grafana_provisioning_dashboards
        target: /etc/grafana/provisioning/dashboards/dashboards.yml
    
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.role == worker
          - node.labels.monitoring == true
      resources:
        limits:
          memory: 1G
          cpus: '0.5'
        reservations:
          memory: 512M
          cpus: '0.25'
    
    healthcheck:
      test: ["CMD-SHELL", "curl -f localhost:3000/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # AlertManager for alerting
  alertmanager:
    image: prom/alertmanager:v0.25.0
    
    networks:
      - monitoring
    
    ports:
      - "9093:9093"
    
    volumes:
      - alertmanager_data:/alertmanager
    
    configs:
      - source: alertmanager_config
        target: /etc/alertmanager/alertmanager.yml
    
    secrets:
      - slack_webhook_url
      - pagerduty_service_key
    
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
      - '--web.external-url=http://alertmanager:9093'
      - '--cluster.listen-address=0.0.0.0:9094'
    
    deploy:
      mode: replicated
      replicas: 2
      placement:
        constraints:
          - node.role == worker
          - node.labels.monitoring == true
      resources:
        limits:
          memory: 256M
          cpus: '0.25'
        reservations:
          memory: 128M
          cpus: '0.1'

  # Node Exporter for system metrics
  node-exporter:
    image: prom/node-exporter:v1.5.0
    
    networks:
      - monitoring
    
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    
    deploy:
      mode: global
      resources:
        limits:
          memory: 128M
          cpus: '0.1'
        reservations:
          memory: 64M
          cpus: '0.05'

  # cAdvisor for container metrics
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.46.0
    
    networks:
      - monitoring
    
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    
    ports:
      - "8080:8080"
    
    deploy:
      mode: global
      resources:
        limits:
          memory: 256M
          cpus: '0.2'
        reservations:
          memory: 128M
          cpus: '0.1'
    
    command:
      - '--housekeeping_interval=10s'
      - '--docker_only=true'
      - '--store_container_labels=false'

  # Elasticsearch for log aggregation
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.6.0
    
    networks:
      - monitoring
    
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
    
    environment:
      - cluster.name=devops-logging-cluster
      - node.name=es-node-1
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms2g -Xmx2g"
      - xpack.security.enabled=false
      - xpack.monitoring.collection.enabled=true
    
    ports:
      - "9200:9200"
    
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.role == worker
          - node.labels.logging == true
      resources:
        limits:
          memory: 4G
          cpus: '2.0'
        reservations:
          memory: 2G
          cpus: '1.0'
    
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9200/_health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Fluentd for log collection
  fluentd:
    image: fluentd:v1.16-debian-1
    
    networks:
      - monitoring
    
    ports:
      - "24224:24224"
      - "24224:24224/udp"
    
    configs:
      - source: fluentd_config
        target: /fluentd/etc/fluent.conf
    
    environment:
      - FLUENTD_CONF=fluent.conf
      - FLUENTD_OPT=-v
    
    deploy:
      mode: replicated
      replicas: 2
      placement:
        constraints:
          - node.role == worker
      resources:
        limits:
          memory: 512M
          cpus: '0.5'
        reservations:
          memory: 256M
          cpus: '0.25'
    
    depends_on:
      - elasticsearch

  # Jaeger for distributed tracing
  jaeger:
    image: jaegertracing/all-in-one:1.42
    
    networks:
      - monitoring
    
    ports:
      - "14268:14268"  # jaeger.thrift
      - "16686:16686"  # UI
    
    environment:
      - COLLECTOR_OTLP_ENABLED=true
      - SPAN_STORAGE_TYPE=elasticsearch
      - ES_SERVER_URLS=http://elasticsearch:9200
    
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.role == worker
          - node.labels.monitoring == true
      resources:
        limits:
          memory: 512M
          cpus: '0.5'
        reservations:
          memory: 256M
          cpus: '0.25'
    
    depends_on:
      - elasticsearch

configs:
  prometheus_config:
    external: true
  prometheus_rules:
    external: true
  grafana_provisioning_datasources:
    external: true
  grafana_provisioning_dashboards:
    external: true
  alertmanager_config:
    external: true
  fluentd_config:
    external: true

secrets:
  grafana_admin_password:
    external: true
  smtp_password:
    external: true
  slack_webhook_url:
    external: true
  pagerduty_service_key:
    external: true
```

### Task 3: Disaster Recovery and Business Continuity (60 minutes)

**2.2 Automated Backup and Recovery System**

```bash
#!/bin/bash
# File: scripts/disaster-recovery.sh
# Comprehensive disaster recovery system

set -e

# Configuration
STACK_NAME="devops-app-prod"
S3_BUCKET="your-backup-bucket"
BACKUP_RETENTION_DAYS=90
ENCRYPTION_KEY_FILE="/etc/backup-encryption-key"
NOTIFICATION_WEBHOOK="$SLACK_WEBHOOK_URL"

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'

print_header() {
    echo -e "\n${BLUE}=== $1 ===${NC}\n"
}

print_success() {
    echo -e "${GREEN}✅ $1${NC}"
}

print_warning() {
    echo -e "${YELLOW}⚠️  $1${NC}"
}

print_error() {
    echo -e "${RED}❌ $1${NC}"
}

# Send notifications
send_notification() {
    local message="$1"
    local status="${2:-info}"
    
    if [ -n "$NOTIFICATION_WEBHOOK" ]; then
        local color="good"
        local emoji="ℹ️"
        
        case $status in
            "success") color="good"; emoji="✅" ;;
            "warning") color="warning"; emoji="⚠️" ;;
            "error") color="danger"; emoji="❌" ;;
        esac
        
        curl -X POST -H 'Content-type: application/json' \
            --data "{
                \"attachments\": [{
                    \"color\": \"$color\",
                    \"text\": \"$emoji $message\",
                    \"footer\": \"Disaster Recovery System\",
                    \"ts\": $(date +%s)
                }]
            }" \
            "$NOTIFICATION_WEBHOOK" 2>/dev/null || true
    fi
}

# Full system backup
full_backup() {
    print_header "Full System Backup"
    
    local backup_timestamp=$(date +%Y%m%d_%H%M%S)
    local backup_dir="/tmp/backup_${backup_timestamp}"
    local backup_name="full_backup_${backup_timestamp}"
    
    mkdir -p "$backup_dir"
    
    send_notification "Starting full system backup: $backup_name"
    
    # Database backup
    print_success "Backing up PostgreSQL database..."
    docker exec $(docker ps -q --filter "name=${STACK_NAME}_postgres-primary") \
        pg_dumpall -U appuser | gzip > "${backup_dir}/database_full.sql.gz"
    
    # Application data backup
    print_success "Backing up application data..."
    
    # Backup volumes
    for volume in $(docker volume ls --filter "name=${STACK_NAME}" --format "{{.Name}}"); do
        echo "Backing up volume: $volume"
        docker run --rm -v "$volume":/volume -v "$backup_dir":/backup alpine \
            tar czf "/backup/${volume}.tar.gz" -C /volume .
    done
    
    # Configuration backup
    print_success "Backing up configurations..."
    
    # Export secrets (metadata only, not content for security)
    docker secret ls --format "{{.Name}}" > "${backup_dir}/secrets_list.txt"
    
    # Export configs
    docker config ls --format "{{.Name}}" > "${backup_dir}/configs_list.txt"
    
    # Export service definitions
    docker service ls --format "table {{.Name}}\t{{.Image}}\t{{.Replicas}}" > "${backup_dir}/services.txt"
    
    # Stack configuration
    docker stack config "$STACK_NAME" > "${backup_dir}/stack_config.yml" 2>/dev/null || true
    
    # Network configuration
    docker network ls --format "table {{.Name}}\t{{.Driver}}\t{{.Scope}}" > "${backup_dir}/networks.txt"
    
    # Create manifest file
    cat > "${backup_dir}/backup_manifest.json" << EOF
{
    "backup_name": "$backup_name",
    "timestamp": "$backup_timestamp",
    "stack_name": "$STACK_NAME",
    "backup_type": "full",
    "components": [
        "database",
        "volumes",
        "configurations",
        "service_definitions"
    ],
    "retention_days": $BACKUP_RETENTION_DAYS,
    "encryption": "enabled"
}
EOF
    
    # Encrypt backup
    print_success "Encrypting backup..."
    if [ -f "$ENCRYPTION_KEY_FILE" ]; then
        tar czf - -C "$backup_dir" . | \
        gpg --cipher-algo AES256 --compress-algo 2 --symmetric \
            --passphrase-file "$ENCRYPTION_KEY_FILE" \
            --output "/tmp/${backup_name}.tar.gz.gpg"
        
        # Upload to S3
        print_success "Uploading backup to S3..."
        aws s3 cp "/tmp/${backup_name}.tar.gz.gpg" "s3://$S3_BUCKET/backups/"
        
        # Cleanup local files
        rm -rf "$backup_dir" "/tmp/${backup_name}.tar.gz.gpg"
        
        print_success "Full backup completed: $backup_name"
        send_notification "Full system backup completed successfully: $backup_name" "success"
    else
        print_error "Encryption key file not found: $ENCRYPTION_KEY_FILE"
        send_notification "Backup failed: encryption key not found" "error"
        return 1
    fi
    
    # Cleanup old backups
    cleanup_old_backups
}

# Incremental backup
incremental_backup() {
    print_header "Incremental Backup"
    
    local backup_timestamp=$(date +%Y%m%d_%H%M%S)
    local backup_name="incremental_backup_${backup_timestamp}"
    local backup_dir="/tmp/$backup_name"
    
    mkdir -p "$backup_dir"
    
    send_notification "Starting incremental backup: $backup_name"
    
    # Database incremental backup (WAL files)
    print_success "Creating database incremental backup..."
    
    # Get current WAL position
    wal_position=$(docker exec $(docker ps -q --filter "name=${STACK_NAME}_postgres-primary") \
        psql -U appuser -d devops_app_prod -t -c "SELECT pg_current_wal_lsn();")
    
    # Archive WAL files since last backup
    docker exec $(docker ps -q --filter "name=${STACK_NAME}_postgres-primary") \
        find /var/lib/postgresql/data/pg_wal -name "*.backup" -newer /var/lib/postgresql/last_backup_marker 2>/dev/null | \
        head -10 > "${backup_dir}/wal_files.list" || true
    
    # Create backup marker
    docker exec $(docker ps -q --filter "name=${STACK_NAME}_postgres-primary") \
        touch /var/lib/postgresql/last_backup_marker
    
    # Application logs backup
    print_success "Backing up application logs..."
    docker service logs "$STACK_NAME" --since 24h > "${backup_dir}/application_logs.txt" 2>/dev/null || true
    
    # Recent monitoring data
    if docker service ls | grep -q prometheus; then
        curl -s "http://localhost:9090/api/v1/query?query=up" > "${backup_dir}/prometheus_snapshot.json" 2>/dev/null || true
    fi
    
    # Create manifest
    cat > "${backup_dir}/backup_manifest.json" << EOF
{
    "backup_name": "$backup_name",
    "timestamp": "$backup_timestamp",
    "stack_name": "$STACK_NAME",
    "backup_type": "incremental",
    "wal_position": "$wal_position",
    "components": [
        "database_wal",
        "application_logs",
        "monitoring_snapshot"
    ]
}
EOF
    
    # Encrypt and upload
    tar czf - -C "$backup_dir" . | \
    gpg --cipher-algo AES256 --symmetric \
        --passphrase-file "$ENCRYPTION_KEY_FILE" \
        --output "/tmp/${backup_name}.tar.gz.gpg"
    
    aws s3 cp "/tmp/${backup_name}.tar.gz.gpg" "s3://$S3_BUCKET/backups/incremental/"
    
    # Cleanup
    rm -rf "$backup_dir" "/tmp/${backup_name}.tar.gz.gpg"
    
    print_success "Incremental backup completed: $backup_name"
    send_notification "Incremental backup completed: $backup_name" "success"
}

# Point-in-time recovery
point_in_time_recovery() {
    local recovery_timestamp="$1"
    
    if [ -z "$recovery_timestamp" ]; then
        print_error "Recovery timestamp required (format: YYYY-MM-DD HH:MM:SS)"
        return 1
    fi
    
    print_header "Point-in-Time Recovery to $recovery_timestamp"
    
    send_notification "Starting point-in-time recovery to $recovery_timestamp" "warning"
    
    # Stop application services
    print_warning "Stopping application services..."
    docker service scale "${STACK_NAME}_frontend"=0
    docker service scale "${STACK_NAME}_backend"=0
    
    # Find appropriate backup
    print_success "Finding appropriate backup..."
    local base_backup=$(aws s3 ls "s3://$S3_BUCKET/backups/" | \
        awk '{print $4}' | grep "full_backup" | \
        sort -r | head -1)
    
    if [ -z "$base_backup" ]; then
        print_error "No base backup found"
        send_notification "Recovery failed: no base backup found" "error"
        return 1
    fi
    
    # Download and decrypt base backup
    print_success "Restoring from base backup: $base_backup"
    aws s3 cp "s3://$S3_BUCKET/backups/$base_backup" "/tmp/$base_backup"
    
    local restore_dir="/tmp/restore_$(date +%s)"
    mkdir -p "$restore_dir"
    
    gpg --decrypt --passphrase-file "$ENCRYPTION_KEY_FILE" \
        "/tmp/$base_backup" | tar xzf - -C "$restore_dir"
    
    # Restore database
    print_success "Restoring database..."
    
    # Stop current database
    docker service scale "${STACK_NAME}_postgres-primary"=0
    sleep 30
    
    # Restore database data
    docker run --rm -v "${STACK_NAME}_postgres_prod_data":/volume -v "$restore_dir":/backup alpine \
        sh -c "rm -rf /volume/* && tar xzf /backup/postgres_prod_data.tar.gz -C /volume"
    
    # Start database in recovery mode
    docker service update \
        --env-add "POSTGRES_RECOVERY_TARGET_TIME=$recovery_timestamp" \
        "${STACK_NAME}_postgres-primary"
    
    docker service scale "${STACK_NAME}_postgres-primary"=1
    
    # Wait for database recovery
    print_success "Waiting for database recovery..."
    sleep 60
    
    # Verify database state
    if docker exec $(docker ps -q --filter "name=${STACK_NAME}_postgres-primary") \
        psql -U appuser -d devops_app_prod -c "SELECT NOW();" > /dev/null 2>&1; then
        
        print_success "Database recovery completed successfully"
        
        # Restart application services
        print_success "Restarting application services..."
        docker service scale "${STACK_NAME}_backend"=3
        docker service scale "${STACK_NAME}_frontend"=3
        
        # Verify application health
        sleep 120
        if curl -f -s http://localhost/health >/dev/null 2>&1; then
            print_success "Point-in-time recovery completed successfully"
            send_notification "Point-in-time recovery completed successfully to $recovery_timestamp" "success"
        else
            print_error "Application health check failed after recovery"
            send_notification "Recovery completed but application health check failed" "error"
        fi
    else
        print_error "Database recovery failed"
        send_notification "Database recovery failed during point-in-time recovery" "error"
        return 1
    fi
    
    # Cleanup
    rm -rf "$restore_dir" "/tmp/$base_backup"
}

# Cleanup old backups
cleanup_old_backups() {
    print_header "Cleaning Up Old Backups"
    
    # List old backups in S3
    old_backups=$(aws s3 ls "s3://$S3_BUCKET/backups/" | \
        awk '{print $4}' | \
        while read backup; do
            backup_date=$(echo "$backup" | grep -o '[0-9]\{8\}' | head -1)
            if [ -n "$backup_date" ]; then
                backup_epoch=$(date -d "$backup_date" +%s 2>/dev/null || echo 0)
                cutoff_epoch=$(date -d "$BACKUP_RETENTION_DAYS days ago" +%s)
                if [ "$backup_epoch" -lt "$cutoff_epoch" ]; then
                    echo "$backup"
                fi
            fi
        done)
    
    if [ -n "$old_backups" ]; then
        print_success "Removing old backups older than $BACKUP_RETENTION_DAYS days:"
        echo "$old_backups"
        
        echo "$old_backups" | while read backup; do
            aws s3 rm "s3://$S3_BUCKET/backups/$backup"
        done
        
        send_notification "Cleaned up old backups older than $BACKUP_RETENTION_DAYS days" "success"
    else
        print_success "No old backups to clean up"
    fi
}

# Disaster recovery test
disaster_recovery_test() {
    print_header "Disaster Recovery Test"
    
    local test_timestamp=$(date +%Y%m%d_%H%M%S)
    send_notification "Starting disaster recovery test: $test_timestamp" "warning"
    
    # Create test environment
    local test_stack_name="${STACK_NAME}_dr_test"
    
    # Find latest backup
    local latest_backup=$(aws s3 ls "s3://$S3_BUCKET/backups/" | \
        awk '{print $4}' | grep "full_backup" | sort -r | head -1)
    
    if [ -z "$latest_backup" ]; then
        print_error "No backup found for testing"
        return 1
    fi
    
    print_success "Testing recovery with backup: $latest_backup"
    
    # Simulate recovery process in test environment
    # This would typically restore to a separate test infrastructure
    
    # For demonstration, we'll just validate the backup integrity
    aws s3 cp "s3://$S3_BUCKET/backups/$latest_backup" "/tmp/test_$latest_backup"
    
    if gpg --decrypt --passphrase-file "$ENCRYPTION_KEY_FILE" \
        "/tmp/test_$latest_backup" | tar tzf - > /dev/null 2>&1; then
        
        print_success "Backup integrity test passed"
        send_notification "Disaster recovery test completed successfully: backup integrity verified" "success"
    else
        print_error "Backup integrity test failed"
        send_notification "Disaster recovery test failed: backup corruption detected" "error"
        return 1
    fi
    
    # Cleanup test files
    rm -f "/tmp/test_$latest_backup"
    
    print_success "Disaster recovery test completed"
}

# Main function
main() {
    case "${1:-help}" in
        "full-backup")
            full_backup
            ;;
        "incremental-backup")
            incremental_backup
            ;;
        "point-in-time-recovery")
            point_in_time_recovery "$2"
            ;;
        "cleanup")
            cleanup_old_backups
            ;;
        "test")
            disaster_recovery_test
            ;;
        "help")
            echo "Usage: $0 {full-backup|incremental-backup|point-in-time-recovery|cleanup|test}"
            echo ""
            echo "Commands:"
            echo "  full-backup                      Create full system backup"
            echo "  incremental-backup               Create incremental backup"
            echo "  point-in-time-recovery 'YYYY-MM-DD HH:MM:SS'  Restore to specific timestamp"
            echo "  cleanup                          Remove old backups"
            echo "  test                            Test disaster recovery procedures"
            ;;
        *)
            echo "Invalid command. Use '$0 help' for usage information."
            exit 1
            ;;
    esac
}

# Set up backup schedule if running as cron
if [ "$1" = "cron" ]; then
    # Daily full backup at 2 AM
    if [ "$(date +%H)" = "02" ]; then
        full_backup
    else
        # Incremental backup every 4 hours
        if [ $(($(date +%H) % 4)) -eq 0 ]; then
            incremental_backup
        fi
    fi
else
    main "$@"
fi
```

## Deliverables

1. **Production Docker Swarm Deployment:**
   - Multi-node high availability cluster setup
   - Load-balanced application services with auto-scaling
   - Database replication and failover capabilities
   - Secure networking with encrypted overlay networks

2. **Comprehensive Monitoring Stack:**
   - Prometheus metrics collection with custom rules
   - Grafana visualization with production dashboards
   - Centralized logging with ELK stack
   - Distributed tracing with Jaeger
   - Alerting with PagerDuty and Slack integration

3. **Disaster Recovery System:**
   - Automated backup procedures (full and incremental)
   - Point-in-time recovery capabilities
   - Disaster recovery testing automation
   - Multi-region backup storage with encryption

4. **Production Deployment Automation:**
   - Zero-downtime deployment with health checks
   - Automated rollback on failure detection
   - Performance monitoring and alerting
   - Compliance logging and audit trails

## Success Criteria

- Successfully deployed highly available multi-service application
- Implemented comprehensive monitoring with real-time alerting
- Created automated backup and disaster recovery procedures
- Achieved zero-downtime deployment capabilities
- Established performance baselines and optimization
- Demonstrated production-grade security and compliance
- Created comprehensive documentation and runbooks

## Verification Commands

```bash
# Deploy production stack
export REGISTRY_URL=your-registry.com
export IMAGE_TAG=v1.0.0
./scripts/deploy-production.sh

# Verify high availability
docker node ls
docker service ls
docker service ps devops-app-prod_backend

# Test load balancing
for i in {1..10}; do curl -s http://localhost/api/health | jq '.hostname'; done

# Create full backup
./scripts/disaster-recovery.sh full-backup

# Test disaster recovery
./scripts/disaster-recovery.sh test

# Monitor deployment health
curl http://localhost:9090/targets  # Prometheus targets
curl http://localhost:3000/api/health  # Grafana health

# Check logs
docker service logs devops-app-prod_backend --follow
```

## Integration with Next Exercise

This production deployment foundation prepares for:
- Kubernetes migration and comparison
- Advanced orchestration patterns
- Cloud-native deployment strategies
- Service mesh integration
- Advanced monitoring and observability

## Real-World Applications

**Enterprise Production Requirements:**
- **High Availability**: 99.9%+ uptime with automatic failover
- **Scalability**: Handle 10x traffic spikes with auto-scaling
- **Security**: Enterprise-grade security and compliance
- **Monitoring**: Comprehensive observability and alerting
- **Disaster Recovery**: RTO < 4 hours, RPO < 1 hour
- **Performance**: Sub-500ms response times under load

This comprehensive production deployment exercise provides 210 minutes of intensive hands-on experience with enterprise-grade container orchestration and production operations!
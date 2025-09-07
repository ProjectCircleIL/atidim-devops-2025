# Module 3: Docker & Docker Compose (Days 6-8)

## Learning Objectives
By the end of this module, students will be able to:
- Containerize their 3-tier application from previous modules
- Create optimized, production-ready Docker images
- Implement multi-stage builds for different environments
- Use Docker Compose for local development environments
- Integrate Docker builds with GitHub Actions CI/CD
- Apply Docker security best practices
- Deploy containers to registries (ECR, DockerHub)

## Topics Covered

### Day 6: Docker Fundamentals & Application Containerization
**Morning (4 hours):**
- Docker Architecture and Core Concepts
- Container vs VM comparison
- Docker Images, Containers, and Registries
- Dockerfile basics and best practices
- Multi-stage builds for optimization

**Afternoon (4 hours):**
- Containerizing your application stack
- Frontend containerization (React, Vue, etc.)
- Backend API containerization (Node.js, Python, etc.)
- Database containerization and data persistence
- Container networking basics

### Day 7: Docker Compose & Development Workflows
**Morning (4 hours):**
- Docker Compose fundamentals
- YAML configuration and service definitions
- Inter-service communication
- Environment variable management
- Volume mounting and data persistence

**Afternoon (4 hours):**
- Complete development environment setup
- Hot reloading and development workflows
- Database initialization and seeding
- Service dependencies and health checks
- Docker networking deep dive

### Day 8: Production Readiness & CI/CD Integration
**Morning (4 hours):**
- Production Docker optimizations
- Security scanning and best practices
- Container registries (ECR, DockerHub, Harbor)
- Image tagging strategies
- Container monitoring and logging

**Afternoon (4 hours):**
- CI/CD integration with GitHub Actions
- Automated container builds and pushes
- Multi-environment container strategies
- Container orchestration preparation (for Kubernetes)
- Performance optimization and troubleshooting

## Hands-On Exercises

### Progressive Exercise Structure
Each exercise builds upon your Git repository and application from Module 2:

1. **Application Containerization** (90 min) - Create Dockerfiles for all 3 tiers
2. **Multi-Stage Docker Builds** (75 min) - Optimize for production vs development
3. **Docker Compose Development Environment** (90 min) - Complete local stack
4. **Container Registry Integration** (60 min) - Push to ECR/DockerHub via GitHub Actions
5. **Docker Security and Best Practices** (90 min) - Security scanning and optimization
6. **Production Docker Setup** (120 min) - Production-ready multi-environment containers

## Project Integration
This module containerizes the application built in Modules 1-2:
- **Uses**: Application code, Git workflows, and CI/CD foundation from previous modules
- **Creates**: Production-ready container images for all application components
- **Prepares for**: Kubernetes deployment (Module 4), advanced CI/CD (Module 5-6), and infrastructure (Module 7)

## Prerequisites
- Completed Module 2 (working application in Git repository)
- Docker Desktop installed locally
- AWS account (for ECR integration)
- Basic understanding of YAML syntax

## Success Metrics
By module completion, students will have:
- Dockerized 3-tier application with optimized images
- Complete Docker Compose development environment
- Automated container builds via GitHub Actions
- Images pushed to container registry
- Security scanned and optimized containers
- Production-ready container deployment strategy

## Tools and Technologies
- **Docker**: Container runtime and image building
- **Docker Compose**: Multi-container development environments
- **DockerHub/ECR**: Container registries  
- **GitHub Actions**: CI/CD integration for container builds
- **Trivy/Snyk**: Container security scanning
- **Dive**: Container image analysis

## Integration with Future Modules
- **Module 4 (Kubernetes)**: Deploys these containers to K8s clusters
- **Module 5 (Jenkins)**: Alternative CI/CD for container builds
- **Module 6 (GitOps)**: GitOps deployment of containerized applications
- **Module 7 (Terraform)**: Provisions container registry and infrastructure
- **Module 8 (Monitoring)**: Monitors containerized applications
- **Module 9 (Final Project)**: Complete containerized application deployment
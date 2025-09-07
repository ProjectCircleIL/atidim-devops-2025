# Exercise 3: CI/CD Pipeline Design

**Duration:** 60 minutes  
**Difficulty:** Intermediate  
**Type:** Design + Documentation

## Objective
Design a comprehensive CI/CD pipeline for your 3-tier application that will be implemented throughout the course using real DevOps tools.

## Scenario
You need to design a CI/CD pipeline that supports:
- **Multiple environments:** Development, Staging, Production  
- **Multiple services:** Frontend, Backend, Database
- **Quality gates:** Testing, security scans, code quality
- **Deployment strategies:** Automated rollouts and rollbacks
- **Monitoring:** Health checks and alerting

## Prerequisites
- Completed Exercise 2 (technology stack selection)
- Basic understanding of CI/CD concepts from today's lecture

## Tasks

### Task 1: Pipeline Architecture Design (25 minutes)

Create a visual diagram of your CI/CD pipeline. Include these components:

**Source Control Trigger:**
- What events trigger the pipeline? (commits, PRs, tags)
- Which branches trigger which environments?

**Build Stage:**
- How will you build your frontend?
- How will you build your backend?
- What about dependencies and caching?

**Test Stages:**
- Unit tests
- Integration tests  
- End-to-end tests
- Security scans
- Code quality checks

**Packaging Stage:**
- How will you create deployment artifacts?
- Container image creation and tagging
- Artifact storage

**Deployment Stages:**
- Development environment deployment
- Staging environment deployment  
- Production deployment (manual approval?)
- Rollback procedures

Draw your pipeline as a flowchart showing:
```
[Trigger] → [Build] → [Test] → [Package] → [Deploy Dev] → [Deploy Staging] → [Approval] → [Deploy Prod]
```

### Task 2: Environment Strategy (20 minutes)

Define your environment strategy:

| Environment | Purpose | Trigger | Auto-Deploy | Approval Required |
|-------------|---------|---------|-------------|------------------|
| Development | Feature testing | Every commit to feature branch | Yes | No |
| Staging | Integration testing | Merge to main/develop | Yes | No |
| Production | Live application | Git tag (v1.0.0) | No | Yes |

**Environment Configuration:**
- How will you manage environment-specific configurations?
- What secrets/credentials are needed for each environment?
- How will you handle database migrations across environments?

### Task 3: Quality Gates Definition (15 minutes)

Define quality gates that must pass before deployment:

**Build Quality Gates:**
- [ ] Code compiles successfully
- [ ] Dependencies resolve
- [ ] Docker images build successfully
- [ ] Other: ________________

**Test Quality Gates:**
- [ ] Unit test coverage > 80%
- [ ] All integration tests pass
- [ ] Performance tests within acceptable limits
- [ ] Security vulnerability scan passes
- [ ] Other: ________________

**Deployment Quality Gates:**
- [ ] Health checks pass
- [ ] Database migrations complete
- [ ] Services respond to requests
- [ ] Monitoring alerts configured
- [ ] Other: ________________

## Deliverables

1. **Pipeline Architecture Diagram** - Visual representation of your CI/CD flow
2. **Environment Strategy Table** - Completed with your specific requirements
3. **Quality Gates Checklist** - Comprehensive list of gates for each stage
4. **Technology Mapping** - Which tools will you use for each stage:

| Pipeline Stage | Tool Choice | Rationale |
|----------------|-------------|-----------|
| Source Control | Git + GitHub | (example) |
| CI/CD Platform | GitHub Actions / Jenkins | |
| Container Registry | Docker Hub / AWS ECR | |
| Testing Framework | (depends on your tech stack) | |
| Deployment Target | Kubernetes | |
| Monitoring | Prometheus + Grafana | |

## Success Criteria
- Created a realistic, implementable CI/CD design
- Addressed all three tiers of your application
- Defined clear quality gates and approval processes
- Selected appropriate tools that integrate well together
- Considered security and compliance requirements

## Real-World Integration
This design will be implemented progressively:

- **Module 2 (Git/GitHub):** Set up repositories and basic GitHub Actions
- **Module 3 (Docker):** Implement container building stages  
- **Module 4 (Kubernetes):** Add deployment stages
- **Module 5 (Jenkins):** Alternative CI/CD implementation
- **Module 6 (GitOps):** ArgoCD deployment automation
- **Module 7 (Terraform):** Infrastructure provisioning
- **Module 8 (Monitoring):** Add observability to your pipeline

## Advanced Challenges

1. **Multi-Service Coordination:** How will you handle dependencies between frontend and backend deployments?

2. **Database Migrations:** How will you safely deploy database schema changes?

3. **Rollback Strategy:** Design a quick rollback procedure if production deployment fails.

4. **Monitoring Integration:** How will you automatically create monitoring dashboards for new services?

## Template for Documentation

Use this template to document your pipeline:

```markdown
# CI/CD Pipeline Design - [Your Application Name]

## Overview
Brief description of your application and pipeline goals.

## Architecture
[Insert your pipeline diagram here]

## Environments
[Your environment strategy table]

## Quality Gates  
[Your quality gates checklist]

## Tool Selection
[Your technology mapping table]

## Implementation Plan
Phase 1: Basic build and test (Module 2-3)
Phase 2: Container deployment (Module 4)  
Phase 3: Advanced automation (Module 5-6)
Phase 4: Infrastructure integration (Module 7-8)
```

Save this design - you'll implement it piece by piece throughout the course!
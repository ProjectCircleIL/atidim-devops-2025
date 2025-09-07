# Module 2: Git, GitHub, and GitHub Actions (Days 3-5)

## Learning Objectives
By the end of this module, students will be able to:
- Master Git workflow for collaborative development
- Implement branching strategies suitable for DevOps
- Create and manage GitHub repositories with proper collaboration features
- Build automated CI/CD workflows using GitHub Actions
- Apply Git and GitHub skills to their final project application

## Topics Covered

### Day 3: Git Fundamentals & Advanced Workflows
**Morning (4 hours):**
- Git Fundamentals Review
  - Repository initialization and cloning
  - Basic commands: add, commit, push, pull
  - Understanding the staging area
  - Git configuration and best practices

- Advanced Git Operations
  - Branching strategies (GitFlow, GitHub Flow, GitLab Flow)
  - Merging vs. rebasing strategies
  - Cherry-picking and advanced commit manipulation
  - Git hooks for automation

**Afternoon (4 hours):**
- Collaboration Workflows
  - Fork-based workflows
  - Pull request/merge request workflows
  - Code review processes and best practices
  - Conflict resolution strategies
  - Branch protection and quality gates

### Day 4: GitHub Platform & Collaboration
**Morning (4 hours):**
- GitHub Repository Management
  - Repository setup and configuration
  - Issue tracking and project management
  - Wiki and documentation management
  - Security features (secrets, dependabot)

- Team Collaboration Features
  - Teams and permissions management
  - Branch protection rules
  - Required status checks
  - Code review assignments and policies

**Afternoon (4 hours):**
- GitHub Advanced Features
  - GitHub Templates (issue, PR, repository)
  - GitHub Packages integration
  - GitHub Security features (dependency scanning, code scanning)
  - Integration with external tools

### Day 5: GitHub Actions & CI/CD
**Morning (4 hours):**
- GitHub Actions Fundamentals
  - Workflow syntax and structure
  - Triggers and event-driven automation
  - Jobs, steps, and actions marketplace
  - Secrets and environment management

**Afternoon (4 hours):**
- Advanced GitHub Actions
  - Matrix builds and parallel execution
  - Custom actions development
  - Workflow dependencies and artifacts
  - Integration with external services (Docker, cloud providers)

## Hands-On Exercises

### Exercise Progression
Each exercise builds upon your project from Module 1 and prepares for Docker containerization in Module 3.

1. **Git Workflow Setup** (45 minutes) - Initialize project repositories with proper structure
2. **Branching Strategy Implementation** (60 minutes) - Implement GitFlow for your 3-tier application  
3. **Collaborative Development Simulation** (90 minutes) - Multi-person workflow with conflicts
4. **GitHub Repository Configuration** (75 minutes) - Professional repository setup with templates and protection
5. **Basic CI Pipeline with GitHub Actions** (120 minutes) - Automated build and test pipeline
6. **Advanced CI/CD Pipeline** (150 minutes) - Multi-environment deployment preparation

## Project Integration
This module directly implements the version control and CI/CD design from Module 1:

- **Repository Structure:** Separate repos for application code, infrastructure, and configuration
- **Branching Strategy:** Feature branches ’ Development ’ Staging ’ Production
- **CI/CD Foundation:** Automated builds, tests, and artifact creation (containers in Module 3)
- **Collaboration:** Code review processes and team workflows

## Prerequisites
- Completed Module 1 project planning
- Git installed locally
- GitHub account with appropriate permissions
- Basic command line familiarity

## Success Metrics
By module completion, students will have:
- Functional Git workflow supporting their project architecture
- GitHub repositories with professional configuration
- Working CI/CD pipeline ready for containerization
- Demonstrated collaborative development skills
- Documentation and templates for team onboarding

## Tools and Technologies
- **Git:** Version control system
- **GitHub:** Cloud-based Git hosting and collaboration
- **GitHub Actions:** CI/CD and workflow automation
- **Git CLI:** Command-line Git operations
- **GitHub CLI (gh):** Command-line GitHub operations

## Integration with Future Modules
- **Module 3 (Docker):** GitHub Actions will build and push container images
- **Module 4 (Kubernetes):** Deployment manifests will be version controlled and automatically deployed
- **Module 5 (Jenkins):** Alternative CI/CD implementation using Git webhooks
- **Module 6 (GitOps):** Repository structure supporting ArgoCD GitOps workflows
- **Module 7 (Terraform):** Infrastructure code version control and automation
- **Module 8 (Monitoring):** Automated monitoring configuration deployment
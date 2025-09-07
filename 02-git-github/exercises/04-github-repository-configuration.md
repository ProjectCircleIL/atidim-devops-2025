# Exercise 4: GitHub Repository Configuration

**Duration:** 75 minutes  
**Difficulty:** Intermediate  
**Type:** Configuration + Documentation

## Objective
Configure your GitHub repository with professional-grade settings, templates, and protection rules that enforce quality standards and support team collaboration.

## Prerequisites
- Completed Exercise 3 (Collaborative Development Simulation)
- Repository with multiple merged features
- GitHub account with appropriate permissions
- Understanding of branch protection concepts

## Scenario
Your project has grown beyond individual development. You need to establish professional repository governance that:
- Prevents direct pushes to important branches
- Enforces code review processes
- Provides consistent issue and PR templates
- Automates quality checks
- Maintains professional project documentation

## Tasks

### Task 1: Branch Protection Rules (20 minutes)

**1.1 Configure Main Branch Protection**

Go to your GitHub repository → Settings → Branches → Add rule

Configure these settings for `main` branch:

```yaml
Branch Protection Rules for 'main':
✅ Restrict pushes that create files larger than 100MB
✅ Require a pull request before merging
  ✅ Require approvals (1)
  ✅ Dismiss stale PR approvals when new commits are pushed
  ✅ Require review from code owners
✅ Require status checks to pass before merging
  ✅ Require branches to be up to date before merging
  - Status checks (will add after GitHub Actions setup):
    □ continuous-integration
    □ code-quality-check
✅ Require conversation resolution before merging
✅ Require signed commits (optional, for advanced security)
✅ Include administrators (enforce rules for admins too)
✅ Allow force pushes (❌ disabled for safety)
✅ Allow deletions (❌ disabled for safety)
```

**1.2 Configure Develop Branch Protection**

Add similar rules for `develop` branch:

```yaml
Branch Protection Rules for 'develop':
✅ Require a pull request before merging
  ✅ Require approvals (1) 
✅ Require status checks to pass before merging
  ✅ Require branches to be up to date before merging
✅ Require conversation resolution before merging
✅ Include administrators
```

**1.3 Test Branch Protection**

Try to push directly to main (this should fail):
```bash
# This should be blocked
git checkout main
echo "Direct commit test" >> test-protection.txt
git add test-protection.txt
git commit -m "test: direct commit to main (should fail)"
git push origin main
```

You should see an error like: `remote: error: GH006: Protected branch update failed`

### Task 2: Repository Templates and Documentation (25 minutes)

**2.1 Create Issue Templates**

Create `.github/ISSUE_TEMPLATE/` directory and templates:

**Bug Report Template** (`.github/ISSUE_TEMPLATE/bug_report.md`):
```markdown
---
name: Bug report
about: Create a report to help us improve
title: '[BUG] '
labels: 'bug'
assignees: ''
---

## Bug Description
A clear and concise description of what the bug is.

## Steps to Reproduce
1. Go to '...'
2. Click on '....'
3. Scroll down to '....'
4. See error

## Expected Behavior
A clear and concise description of what you expected to happen.

## Actual Behavior
A clear and concise description of what actually happened.

## Screenshots
If applicable, add screenshots to help explain your problem.

## Environment
- OS: [e.g. Ubuntu 20.04, macOS 12.0, Windows 10]
- Browser: [e.g. chrome 91, safari 14]
- Version: [e.g. v1.2.3]

## Additional Context
Add any other context about the problem here.

## Acceptance Criteria
- [ ] Bug is reproduced and confirmed
- [ ] Root cause is identified
- [ ] Fix is implemented and tested
- [ ] Documentation is updated if needed
```

**Feature Request Template** (`.github/ISSUE_TEMPLATE/feature_request.md`):
```markdown
---
name: Feature request
about: Suggest an idea for this project
title: '[FEATURE] '
labels: 'enhancement'
assignees: ''
---

## Feature Description
A clear and concise description of the feature you'd like to see implemented.

## Problem Statement
What problem does this feature solve? What pain point does it address?

## Proposed Solution
Describe the solution you'd like to see implemented.

## Alternative Solutions
Describe any alternative solutions or features you've considered.

## User Stories
- As a [user type], I want [functionality] so that [benefit]
- As a [user type], I want [functionality] so that [benefit]

## Acceptance Criteria
- [ ] Feature requirements are clearly defined
- [ ] Technical design is approved
- [ ] Implementation is complete
- [ ] Tests are written and passing
- [ ] Documentation is updated
- [ ] Feature is deployed and verified

## Additional Context
Add any other context, mockups, or examples about the feature request here.

## Priority
- [ ] Low
- [ ] Medium  
- [ ] High
- [ ] Critical

## Effort Estimate
- [ ] Small (1-2 days)
- [ ] Medium (3-5 days)
- [ ] Large (1-2 weeks)
- [ ] Extra Large (>2 weeks)
```

**2.2 Create Pull Request Template**

Create `.github/pull_request_template.md`:
```markdown
## Pull Request Summary
Brief description of what this PR accomplishes.

Closes #[issue_number]

## Type of Change
- [ ] Bug fix (non-breaking change which fixes an issue)
- [ ] New feature (non-breaking change which adds functionality)
- [ ] Breaking change (fix or feature that would cause existing functionality to not work as expected)
- [ ] Documentation update
- [ ] Performance improvement
- [ ] Code refactoring
- [ ] Security fix

## Changes Made
- List the specific changes made in this PR
- Use bullet points for clarity
- Include any new files added or removed

## Testing Performed
- [ ] Unit tests added/updated and passing
- [ ] Integration tests added/updated and passing
- [ ] Manual testing performed
- [ ] Performance testing (if applicable)
- [ ] Security testing (if applicable)

Describe the testing you performed:

## Deployment Impact
- [ ] No deployment impact
- [ ] Requires database migration
- [ ] Requires configuration changes
- [ ] Requires infrastructure changes
- [ ] Requires coordinated deployment

## Screenshots (if applicable)
Add screenshots to help explain your changes.

## Checklist
- [ ] Code follows project style guidelines
- [ ] Self-review of code completed
- [ ] Code is commented, particularly hard-to-understand areas
- [ ] Corresponding changes to documentation made
- [ ] No new warnings introduced
- [ ] Tests added that prove fix is effective or feature works
- [ ] New and existing unit tests pass locally
- [ ] Any dependent changes have been merged and published

## Additional Notes
Add any additional notes for reviewers or deployment teams.

## Reviewer Notes
@mention specific people you want to review this PR and any specific areas you want them to focus on.
```

**2.3 Create CODEOWNERS File**

Create `.github/CODEOWNERS`:
```
# Global owners (will be requested for review on all PRs)
* @your-username

# Frontend specific
/frontend/ @your-username

# Backend specific  
/backend/ @your-username

# Infrastructure and DevOps
/infrastructure/ @your-username
/.github/ @your-username
/docker-compose*.yml @your-username
/Dockerfile* @your-username

# Documentation
/docs/ @your-username
*.md @your-username

# Configuration files
/.env* @your-username
/config/ @your-username
```

**2.4 Update Main README**

Enhance your project README:
```markdown
# [Your Project Name] - DevOps Learning Project

[![Build Status](https://github.com/[username]/[repo]/workflows/CI/badge.svg)](https://github.com/[username]/[repo]/actions)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

> A comprehensive 3-tier web application demonstrating modern DevOps practices and tools.

## 📋 Table of Contents
- [Overview](#overview)
- [Architecture](#architecture)
- [Quick Start](#quick-start)
- [Development](#development)
- [Deployment](#deployment)
- [Contributing](#contributing)
- [License](#license)

## 🎯 Overview
This project demonstrates a complete DevOps pipeline including:
- **Frontend**: [Your technology choice]
- **Backend**: [Your technology choice]  
- **Database**: [Your technology choice]
- **Infrastructure**: Docker, Kubernetes, Terraform
- **CI/CD**: GitHub Actions, Jenkins
- **Monitoring**: Prometheus, Grafana
- **GitOps**: ArgoCD

## 🏗️ Architecture
```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Frontend  │───▶│   Backend   │───▶│  Database   │
│   (Port:    │    │   (Port:    │    │   (Port:    │
│    3000)    │    │    3001)    │    │    5432)    │
└─────────────┘    └─────────────┘    └─────────────┘
```

## 🚀 Quick Start
```bash
# Clone the repository
git clone https://github.com/[username]/[your-project].git
cd [your-project]

# Run setup script
./scripts/setup-dev-env.sh

# Start development environment
docker-compose -f docker-compose.dev.yml up
```

## 💻 Development

### Prerequisites
- Node.js 16+ (for frontend/backend)
- Docker & Docker Compose
- Git

### Environment Setup
1. Fork and clone the repository
2. Run the development setup script
3. Copy `.env.example` to `.env` and configure
4. Start the development environment

### Branch Strategy
We use GitHub Flow:
- `main`: Production-ready code
- `develop`: Staging/integration branch
- `feature/*`: Feature development
- `hotfix/*`: Critical production fixes

### Commit Convention
Follow [Conventional Commits](https://conventionalcommits.org/):
- `feat:` New features
- `fix:` Bug fixes  
- `docs:` Documentation changes
- `chore:` Maintenance tasks

## 🚢 Deployment
- **Development**: Local Docker Compose
- **Staging**: Kubernetes cluster (develop branch)
- **Production**: Kubernetes cluster (main branch)

See [deployment guide](docs/deployment.md) for details.

## 🤝 Contributing
1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Add tests and documentation
5. Submit a pull request

See [Contributing Guide](CONTRIBUTING.md) for detailed guidelines.

## 📄 License
This project is licensed under the MIT License - see [LICENSE](LICENSE) file.

## 🔗 Links
- [Documentation](docs/)
- [API Documentation](docs/api.md)
- [Deployment Guide](docs/deployment.md)
- [Troubleshooting](docs/troubleshooting.md)
```

### Task 3: Repository Settings and Security (15 minutes)

**3.1 Configure Repository Settings**

Go to Settings → General and configure:

**Repository Settings:**
- [ ] Repository name is descriptive
- [ ] Description clearly explains the project
- [ ] Website URL (if applicable)
- [ ] Topics/tags added for discoverability

**Features:**
- [x] Wikis (for additional documentation)
- [x] Issues (for bug tracking and features)
- [x] Sponsorships (if accepting donations)
- [x] Projects (for project management)
- [x] Preserve this repository (for important projects)
- [x] Discussions (for community)

**Pull Requests:**
- [x] Allow merge commits
- [x] Allow squash merging
- [x] Allow rebase merging
- [x] Always suggest updating pull request branches
- [x] Allow auto-merge
- [x] Automatically delete head branches

**3.2 Security Settings**

Go to Settings → Security & analysis:

**Security Features:**
- [x] Dependency graph
- [x] Dependabot alerts  
- [x] Dependabot security updates
- [x] Dependabot version updates
- [x] Code scanning alerts
- [x] Secret scanning alerts

**3.3 Create Security Policy**

Create `.github/SECURITY.md`:
```markdown
# Security Policy

## Supported Versions
| Version | Supported          |
| ------- | ------------------ |
| 1.x.x   | :white_check_mark: |
| 0.x.x   | :x:                |

## Reporting a Vulnerability
We take security vulnerabilities seriously. If you discover a security vulnerability, please follow these steps:

1. **DO NOT** create a public GitHub issue
2. Email us at [security@yourproject.com] with:
   - Description of the vulnerability
   - Steps to reproduce
   - Potential impact
   - Suggested fix (if known)

3. We will respond within 48 hours
4. We will work with you to understand and resolve the issue
5. We will coordinate disclosure timing with you

## Security Best Practices
- Keep dependencies up to date
- Use environment variables for secrets
- Never commit credentials to git
- Enable 2FA on all accounts
- Regularly audit access permissions

## Security Updates
Security updates will be released as patch versions and communicated through:
- GitHub Security Advisories
- Release notes
- Email notifications (if subscribed)

Thank you for helping keep our project secure!
```

### Task 4: Labels and Project Management (15 minutes)

**4.1 Create Issue Labels**

Add these labels to your repository (Settings → Labels):

**Type Labels:**
- `bug` - Something isn't working (color: #d73a4a)
- `enhancement` - New feature or request (color: #a2eeef)
- `documentation` - Improvements to documentation (color: #0075ca)
- `question` - Further information is requested (color: #d876e3)

**Priority Labels:**
- `priority: critical` - Critical priority (color: #B60205)
- `priority: high` - High priority (color: #D93F0B)
- `priority: medium` - Medium priority (color: #FBCA04)
- `priority: low` - Low priority (color: #0E8A16)

**Status Labels:**
- `status: needs-triage` - Needs initial review (color: #ededed)
- `status: in-progress` - Currently being worked on (color: #fbca04)
- `status: needs-review` - Waiting for review (color: #d4c5f9)
- `status: blocked` - Blocked by external dependency (color: #d73a4a)

**Area Labels:**
- `area: frontend` - Frontend related (color: #1f77b4)
- `area: backend` - Backend related (color: #ff7f0e)
- `area: database` - Database related (color: #2ca02c)
- `area: infrastructure` - Infrastructure related (color: #d62728)
- `area: ci-cd` - CI/CD related (color: #9467bd)

**4.2 Create Project Board**

1. Go to Projects → New project → Board
2. Name: "DevOps Learning Project"
3. Create columns:
   - **📋 Backlog**: New issues and ideas
   - **🏃 In Progress**: Currently being worked on
   - **👀 In Review**: Waiting for code review
   - **✅ Done**: Completed items

**4.3 Create Sample Issues**

Create a few sample issues to demonstrate the template and label system:

**Issue 1: Feature Request**
```markdown
Title: [FEATURE] Add user registration functionality
Labels: enhancement, area: backend, priority: medium

Use the feature request template to create a comprehensive issue for user registration.
```

**Issue 2: Bug Report**  
```markdown
Title: [BUG] Health endpoint returns 500 error on cold start
Labels: bug, area: backend, priority: high

Use the bug report template to document a realistic bug scenario.
```

## Deliverables

1. **Protected Branches:**
   - Main and develop branches with protection rules
   - Evidence that direct pushes are blocked

2. **Repository Templates:**
   - Issue templates for bugs and features
   - Pull request template with comprehensive checklist
   - CODEOWNERS file defining review requirements

3. **Professional Documentation:**
   - Enhanced README with badges and clear structure
   - Security policy document
   - Contributing guidelines

4. **Repository Configuration:**
   - Appropriate settings for collaboration
   - Security features enabled
   - Organized label system
   - Project management setup

## Success Criteria

- Branch protection prevents direct pushes to main/develop
- Templates are used when creating issues and PRs
- Repository looks professional and well-organized
- Security features are properly configured
- Labels and project management are set up
- Documentation is comprehensive and helpful

## Verification Steps

```bash
# Test branch protection (should fail)
git checkout main
echo "test" >> protection-test.txt
git add protection-test.txt
git commit -m "test: should be blocked"
git push origin main

# Verify templates exist
ls -la .github/ISSUE_TEMPLATE/
ls -la .github/

# Check if security features work
# (Create a test commit with a fake API key to test secret scanning)
```

## Integration with Next Exercise

This professional repository setup prepares for Exercise 5:
- Branch protection will enforce CI/CD status checks
- PR templates will include CI/CD verification steps
- CODEOWNERS will ensure DevOps changes are reviewed
- Security scanning will integrate with GitHub Actions

## Real-World Applications

These configurations mirror enterprise development practices:
- **Branch Protection**: Prevents accidental production deployments
- **Code Review**: Ensures code quality and knowledge sharing
- **Templates**: Standardizes issue reporting and PR creation
- **Security**: Automated vulnerability detection and response
- **Project Management**: Organized workflow tracking

## Advanced Challenges

1. **Multiple Environments**: Create different protection rules for staging vs production branches

2. **Team-based CODEOWNERS**: Simulate team ownership with different GitHub accounts

3. **Advanced Security**: Set up dependency scanning and automated security updates

4. **Integration Rules**: Configure status checks that will be implemented in the next exercise

5. **Custom Labels**: Create workflow-specific labels for your project needs

This professional repository configuration establishes the foundation for enterprise-grade development workflows and prepares your project for automated CI/CD integration!
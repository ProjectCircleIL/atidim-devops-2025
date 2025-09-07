# Exercise 2: Branching Strategy Implementation

**Duration:** 60 minutes  
**Difficulty:** Intermediate  
**Type:** Hands-on Implementation

## Objective
Implement a professional Git branching strategy (GitFlow) for your project and practice the complete feature development lifecycle that will support your CI/CD pipeline.

## Prerequisites
- Completed Exercise 1 (Git Workflow Setup)
- Working repository with initial project structure
- Understanding of Git branching concepts

## Scenario
You're implementing the first features of your 3-tier application. You need to establish a branching workflow that supports:
- Parallel feature development
- Code review processes  
- Multiple environments (dev, staging, production)
- Hotfix capabilities
- Release management

You'll implement **GitHub Flow** (simplified GitFlow) which is more suitable for continuous deployment.

## GitHub Flow Strategy

```
main (production) ←--merge-- feature/user-authentication
  ↓                            ↑
develop (staging) ←--branch----┘
```

**Branches:**
- `main`: Production-ready code, protected
- `develop`: Staging environment, integration branch
- `feature/*`: Feature development branches
- `hotfix/*`: Emergency production fixes

## Tasks

### Task 1: Branch Structure Setup (15 minutes)

**1.1 Create and Configure Develop Branch**
```bash
# Navigate to your project directory
cd [your-project-directory]

# Ensure you're on main and up to date
git checkout main
git pull origin main

# Create develop branch
git checkout -b develop
git push -u origin develop

# Set develop as default branch for new features
git config branch.develop.description "Integration branch for staging environment"
```

**1.2 Create Your First Feature Branch**
Based on your application from Module 1, create your first feature. Choose one:

**For Backend API:**
```bash
git checkout develop
git checkout -b feature/api-health-endpoint
```

**For Frontend:**
```bash
git checkout develop  
git checkout -b feature/basic-ui-components
```

**For Database:**
```bash
git checkout develop
git checkout -b feature/user-schema-setup
```

### Task 2: Implement Your First Feature (25 minutes)

**2.1 Backend Feature: API Health Endpoint**
If you chose backend, implement a health check endpoint:

```javascript
// backend/src/routes/health.js (Node.js example)
const express = require('express');
const router = express.Router();

router.get('/health', (req, res) => {
    const healthCheck = {
        uptime: process.uptime(),
        message: 'OK',
        timestamp: new Date().toISOString(),
        environment: process.env.NODE_ENV || 'development',
        version: process.env.npm_package_version || '1.0.0'
    };
    
    res.status(200).json(healthCheck);
});

router.get('/health/detailed', (req, res) => {
    const detailed = {
        ...healthCheck,
        memory: process.memoryUsage(),
        cpu: process.cpuUsage()
    };
    
    res.status(200).json(detailed);
});

module.exports = router;
```

```javascript
// backend/src/app.js - Add the route
const healthRoutes = require('./routes/health');
app.use('/api', healthRoutes);
```

**2.2 Frontend Feature: Basic UI Components**
If you chose frontend, create basic components:

```jsx
// frontend/src/components/HealthStatus.jsx (React example)
import React, { useState, useEffect } from 'react';

const HealthStatus = () => {
    const [health, setHealth] = useState(null);
    const [loading, setLoading] = useState(true);

    useEffect(() => {
        fetch('/api/health')
            .then(response => response.json())
            .then(data => {
                setHealth(data);
                setLoading(false);
            })
            .catch(error => {
                console.error('Health check failed:', error);
                setLoading(false);
            });
    }, []);

    if (loading) return <div>Checking system health...</div>;
    
    return (
        <div className="health-status">
            <h3>System Status</h3>
            <p>Status: {health?.message}</p>
            <p>Uptime: {Math.floor(health?.uptime / 60)} minutes</p>
            <p>Environment: {health?.environment}</p>
        </div>
    );
};

export default HealthStatus;
```

**2.3 Database Feature: User Schema**
If you chose database, create basic schema:

```sql
-- database/migrations/001_create_users_table.sql
CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username ON users(username);
```

```javascript
// database/seeders/001_default_users.js
const defaultUsers = [
    {
        username: 'admin',
        email: 'admin@example.com',
        password_hash: '$2b$10$example_hash_here'
    },
    {
        username: 'testuser',
        email: 'test@example.com', 
        password_hash: '$2b$10$example_hash_here'
    }
];

module.exports = { defaultUsers };
```

**2.4 Add Tests for Your Feature**
Create a simple test for your feature:

```javascript
// Example: backend/tests/health.test.js
const request = require('supertest');
const app = require('../src/app');

describe('Health Endpoints', () => {
    test('GET /api/health should return 200 and health info', async () => {
        const response = await request(app)
            .get('/api/health')
            .expect(200);
        
        expect(response.body).toHaveProperty('message', 'OK');
        expect(response.body).toHaveProperty('timestamp');
        expect(response.body).toHaveProperty('uptime');
    });
    
    test('GET /api/health/detailed should return detailed info', async () => {
        const response = await request(app)
            .get('/api/health/detailed')
            .expect(200);
        
        expect(response.body).toHaveProperty('memory');
        expect(response.body).toHaveProperty('cpu');
    });
});
```

### Task 3: Git Workflow Implementation (20 minutes)

**3.1 Commit Your Changes with Proper Messages**
```bash
# Stage your changes
git add .

# Commit with proper conventional commit format
git commit -m "feat(backend): add health check endpoints with detailed system info

- Add /api/health endpoint for basic health status
- Add /api/health/detailed endpoint for system metrics
- Include uptime, memory, and CPU usage information
- Add comprehensive tests for health endpoints

Closes #1"

# Push feature branch
git push -u origin feature/api-health-endpoint
```

**3.2 Create Pull Request Workflow Simulation**

Since you're working alone for now, simulate the PR process:

```bash
# Create a second feature to practice merging
git checkout develop
git checkout -b feature/update-readme

# Update README.md with your health endpoint documentation
cat >> README.md << 'EOF'

## API Endpoints

### Health Check
- `GET /api/health` - Basic health status
- `GET /api/health/detailed` - Detailed system metrics

### Testing
Run tests with: `npm test`
EOF

git add README.md
git commit -m "docs(readme): add API endpoint documentation

- Document health check endpoints
- Add testing instructions
- Improve project documentation"

git push -u origin feature/update-readme
```

**3.3 Practice Merging Workflow**
```bash
# Merge first feature to develop
git checkout develop
git merge feature/api-health-endpoint --no-ff
git push origin develop

# Merge second feature to develop  
git merge feature/update-readme --no-ff
git push origin develop

# Clean up feature branches (optional for now)
# git branch -d feature/api-health-endpoint
# git branch -d feature/update-readme
```

**3.4 Simulate Release Process**
```bash
# Create a release branch
git checkout develop
git checkout -b release/v1.0.0

# Update version information
echo "v1.0.0" > VERSION
git add VERSION
git commit -m "chore(release): bump version to 1.0.0"

# Merge to main (production)
git checkout main
git merge release/v1.0.0 --no-ff
git tag -a v1.0.0 -m "Release version 1.0.0

Features:
- Health check endpoints
- Basic project structure
- Testing framework setup"

git push origin main --tags

# Merge back to develop
git checkout develop
git merge release/v1.0.0 --no-ff
git push origin develop

# Clean up release branch
git branch -d release/v1.0.0
```

## Deliverables

1. **Working Branch Structure:**
   - `main` branch with tagged release
   - `develop` branch with integrated features
   - Feature branches (can be kept or deleted)

2. **Implemented Features:**
   - At least one working feature with tests
   - Proper commit messages following conventions
   - Updated documentation

3. **Git History:**
   - Clean commit history showing branching strategy
   - Merge commits showing feature integration
   - Tagged release showing version management

4. **Documentation:**
   - Updated README with new features
   - Git workflow evidence in commit history

## Success Criteria

- Successfully created and merged at least 2 feature branches
- Demonstrated proper commit message conventions
- Created a tagged release following semantic versioning
- Clean git history showing professional workflow
- All features are functional and tested

## Verification Commands

```bash
# View branch structure
git log --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit --all

# View tags
git tag -l

# Check current branches
git branch -a

# Verify feature functionality
# (run your application and test the health endpoint)
curl http://localhost:3001/api/health

# Check test coverage
npm test
```

## Integration with Next Exercise

This branching strategy prepares for Exercise 3:
- Feature branches will be used for collaborative development simulation
- Branch protection rules will be applied
- Pull request templates will be created
- CI/CD will run on branch pushes

## Advanced Challenges

1. **Hotfix Simulation:**
   Create a critical bug in main and practice hotfix workflow:
   ```bash
   git checkout main
   git checkout -b hotfix/critical-security-fix
   # Make fix, test, commit
   git checkout main
   git merge hotfix/critical-security-fix --no-ff
   git tag v1.0.1
   git checkout develop
   git merge hotfix/critical-security-fix --no-ff
   ```

2. **Conflict Resolution:**
   Create intentional conflicts between features and practice resolution

3. **Interactive Rebase:**
   Clean up commit history before merging:
   ```bash
   git rebase -i develop
   ```

4. **Cherry-picking:**
   Practice selecting specific commits for hotfixes

## Common Pitfalls and Solutions

**Problem:** Forgot to branch from develop
**Solution:** 
```bash
git checkout feature-branch
git rebase develop
```

**Problem:** Accidentally committed to develop
**Solution:**
```bash
git reset --soft HEAD~1
git checkout -b feature/new-feature
git commit -m "proper message"
```

**Problem:** Need to update feature branch with develop changes
**Solution:**
```bash
git checkout feature-branch
git rebase develop
# or
git merge develop
```

This branching strategy will support your entire development lifecycle and integrate seamlessly with CI/CD pipelines in upcoming modules!
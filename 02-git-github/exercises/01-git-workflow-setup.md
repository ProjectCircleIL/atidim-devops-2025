# Exercise 1: Git Workflow Setup

**Duration:** 45 minutes  
**Difficulty:** Beginner to Intermediate  
**Type:** Hands-on Implementation

## Objective
Set up a professional Git workflow for your 3-tier application project with proper repository structure and initial codebase from Module 1.

## Prerequisites
- Completed Module 1 exercises (especially Exercise 4: automation scripts)
- Git installed and configured locally
- GitHub account ready
- Your application technology stack decided from Module 1

## Scenario
You're setting up version control for your final project application. You need to establish a repository structure that supports:
- 3-tier application development (Frontend, Backend, Database)
- Multi-environment deployments (Dev, Staging, Production)
- Team collaboration
- Future CI/CD integration

## Repository Strategy Decision

Choose one of these repository strategies:

### Option A: Monorepo (Recommended for Learning)
Single repository containing:
```
my-devops-project/
├── frontend/
├── backend/ 
├── database/
├── infrastructure/
├── .github/
└── docs/
```

### Option B: Multi-repo
Separate repositories:
- `my-app-frontend`
- `my-app-backend`
- `my-app-infrastructure`
- `my-app-configs`

**Choose your strategy and justify your choice in your documentation.**

## Tasks

### Task 1: Repository Initialization (15 minutes)

**1.1 Create GitHub Repository**
1. Create a new repository on GitHub named `[your-name]-devops-project`
2. Initialize with README
3. Choose appropriate .gitignore template for your technology stack
4. Add MIT license (or preferred license)

**1.2 Clone and Initial Setup**
```bash
# Clone your repository
git clone https://github.com/[username]/[your-name]-devops-project.git
cd [your-name]-devops-project

# Configure Git (if not done globally)
git config user.name "Your Name"
git config user.email "your.email@example.com"

# Verify remote configuration
git remote -v
```

**1.3 Create Project Structure**
Based on your chosen repository strategy, create the directory structure:

```bash
# For monorepo approach
mkdir -p frontend backend database infrastructure docs scripts
mkdir -p .github/workflows .github/ISSUE_TEMPLATE .github/PULL_REQUEST_TEMPLATE

# Create basic files
touch frontend/README.md backend/README.md database/README.md
touch infrastructure/README.md docs/README.md
```

### Task 2: Import Your Module 1 Work (15 minutes)

**2.1 Import Automation Scripts**
Copy your automation scripts from Module 1 Exercise 4:
```bash
# Copy your scripts to the scripts directory
cp /path/to/your/setup-dev-env.sh scripts/
cp /path/to/your/bootstrap-project.sh scripts/
cp /path/to/your/validate-setup.sh scripts/

# Make them executable
chmod +x scripts/*.sh
```

**2.2 Import Application Boilerplate**
Copy your application code from Module 1 Exercise 4:
- Frontend boilerplate → `frontend/`
- Backend boilerplate → `backend/`
- Database scripts → `database/`

**2.3 Update Scripts for Git Workflow**
Modify your `bootstrap-project.sh` to initialize Git workflow:
```bash
# Add to your bootstrap script
print_status "Initializing Git workflow..."

# Create development branch
git checkout -b develop
git push -u origin develop

# Create basic .gitignore if not exists
if [ ! -f .gitignore ]; then
    # Add .gitignore content based on your tech stack
    echo "node_modules/" >> .gitignore
    echo "*.log" >> .gitignore
    echo ".env" >> .gitignore
    # Add more based on your stack
fi

git add .gitignore
git commit -m "Add project .gitignore"
```

### Task 3: Git Configuration and Standards (15 minutes)

**3.1 Create Git Standards Documentation**
Create `docs/git-standards.md`:
```markdown
# Git Standards and Workflow

## Branching Strategy
- `main`: Production-ready code
- `develop`: Integration branch for features  
- `feature/*`: Feature development branches
- `hotfix/*`: Emergency fixes for production
- `release/*`: Preparation for production releases

## Commit Message Convention
Format: `<type>(<scope>): <description>`

Types:
- feat: New feature
- fix: Bug fix
- docs: Documentation changes
- style: Code style changes (no functionality change)
- refactor: Code refactoring
- test: Adding or modifying tests
- chore: Build process or auxiliary tool changes

Examples:
- `feat(backend): add user authentication API`
- `fix(frontend): resolve login form validation issue`
- `docs(readme): update installation instructions`

## Code Review Process
1. Create feature branch from `develop`
2. Make changes and commit with proper messages
3. Push branch and create Pull Request to `develop`
4. Ensure all CI checks pass
5. Request review from team members
6. Address feedback and re-request review
7. Merge after approval
```

**3.2 Configure Git Hooks**
Create a pre-commit hook to enforce commit standards:
```bash
# Create pre-commit hook
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/bash

# Check commit message format (this runs on commit, not pre-commit)
# But we can check for common issues here

echo "Running pre-commit checks..."

# Check for large files
large_files=$(git diff --cached --name-only | xargs -I {} sh -c 'test -f {} && test $(stat -c%s {}) -gt 1048576 && echo {}' || true)

if [ ! -z "$large_files" ]; then
    echo "Error: Large files detected (>1MB):"
    echo "$large_files"
    echo "Please use Git LFS for large files or ensure they should be committed."
    exit 1
fi

# Check for secrets (basic check)
if git diff --cached | grep -E "(password|secret|key|token)" | grep -v "# password" | grep -q .; then
    echo "Warning: Potential secrets detected in commit. Please review."
    echo "If these are intentional, add '# password' comment to bypass this check."
fi

echo "Pre-commit checks passed!"
EOF

chmod +x .git/hooks/pre-commit
```

## Deliverables

1. **GitHub Repository:** Properly configured repository with:
   - Appropriate .gitignore file
   - Professional README.md
   - License file
   - Initial project structure

2. **Git Configuration:**
   - Local Git configuration
   - Pre-commit hooks setup
   - Git standards documentation

3. **Imported Codebase:**
   - Module 1 automation scripts in `scripts/`
   - Application boilerplate in appropriate directories
   - Updated scripts that work with Git workflow

4. **Documentation:**
   - Git standards and workflow documentation
   - Updated README with project overview and setup instructions

## Success Criteria

- Repository is properly structured and organized
- All Module 1 work is imported and functional
- Git hooks are working correctly
- Documentation is clear and professional
- First commits follow established standards

## Verification Steps

Run these commands to verify your setup:
```bash
# Check repository structure
tree -a -L 2

# Verify Git configuration
git config --list | grep user

# Test pre-commit hook
echo "test" > test-file.txt
git add test-file.txt
git commit -m "test: verify pre-commit hook works"
git reset --soft HEAD~1  # Undo the test commit
git reset HEAD test-file.txt
rm test-file.txt

# Check branch setup
git branch -a

# Verify remote configuration  
git remote -v
```

## Integration with Next Exercise

Your repository will be used in Exercise 2 to:
- Implement feature branch workflows
- Practice collaborative development
- Set up branch protection rules
- Create your first pull requests

## Advanced Challenges

1. **Multi-repo Setup:** If you chose multi-repo strategy, set up all repositories with consistent standards

2. **Git LFS Configuration:** Set up Git Large File Storage for any binary assets

3. **Commit Template:** Create a commit message template:
```bash
git config commit.template ~/.gitmessage
```

4. **Advanced Hooks:** Create commit-msg hook to enforce commit message standards

5. **Submodules:** If using multi-repo, experiment with Git submodules for shared components

## Troubleshooting

**Common Issues:**
- Authentication problems with GitHub → Set up SSH keys or personal access tokens
- Large files rejected → Use Git LFS or add to .gitignore
- Pre-commit hook not executing → Check file permissions with `ls -la .git/hooks/`

**Useful Git Commands:**
```bash
# View commit history
git log --oneline --graph

# Check repository status
git status

# View differences
git diff
git diff --staged

# Unstage files
git reset HEAD <file>

# Undo last commit (keep changes)
git reset --soft HEAD~1
```

This foundation will support all your future development work and CI/CD automation!
# Exercise 4: Basic Automation Script

**Duration:** 90 minutes  
**Difficulty:** Intermediate  
**Type:** Hands-on Coding

## Objective
Create a basic automation script that demonstrates fundamental DevOps principles and will be enhanced throughout the course with more sophisticated tools.

## Scenario
You need to create a development environment setup script that automates common tasks developers perform when starting work on your project. This script will evolve as you learn more tools.

## Prerequisites
- Completed Exercise 2 (technology stack selection)
- Basic command line knowledge
- Text editor or IDE

## Tasks

### Task 1: Environment Setup Script (30 minutes)

Create a script (`setup-dev-env.sh` for Linux/Mac or `setup-dev-env.bat` for Windows) that:

**Basic Requirements:**
1. Checks if required tools are installed
2. Creates necessary project directories
3. Sets up configuration files
4. Provides helpful output messages

**Script Template (Bash):**
```bash
#!/bin/bash

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Function to print colored output
print_status() {
    echo -e "${GREEN}[INFO]${NC} $1"
}

print_warning() {
    echo -e "${YELLOW}[WARNING]${NC} $1"
}

print_error() {
    echo -e "${RED}[ERROR]${NC} $1"
}

print_status "Starting development environment setup..."

# Add your code here
```

**Your script should check for and/or install:**
- Git
- Your chosen programming language runtime (Node.js, Python, Java, etc.)
- Package manager (npm, pip, maven, etc.)  
- Docker (we'll use this in Module 3)
- Text editor/IDE of choice
- Database client tools

**Directory structure to create:**
```
project-name/
├── frontend/
├── backend/
├── database/
├── docs/
├── scripts/
└── .gitignore
```

### Task 2: Project Bootstrap Script (30 minutes)

Create a second script (`bootstrap-project.sh`) that:

1. **Initializes your application structure** based on your technology choices from Exercise 2
2. **Creates boilerplate files:**
   - Frontend: Basic HTML/CSS/JS or framework starter
   - Backend: Basic API structure with one endpoint
   - Database: Initial schema or setup scripts
   - README.md with project description
   - Basic configuration files

**Example for Node.js/Express + React:**
```bash
# Backend setup
mkdir -p backend/src backend/tests
cd backend
npm init -y
npm install express cors dotenv
# Create basic server.js
cat > src/server.js << 'EOF'
const express = require('express');
const cors = require('cors');
require('dotenv').config();

const app = express();
const PORT = process.env.PORT || 3001;

app.use(cors());
app.use(express.json());

app.get('/api/health', (req, res) => {
    res.json({ status: 'OK', timestamp: new Date().toISOString() });
});

app.listen(PORT, () => {
    console.log(`Server running on port ${PORT}`);
});
EOF
cd ..

# Frontend setup  
mkdir -p frontend/src frontend/public
cd frontend
npm init -y
npm install react react-dom
# Create basic index.html and App.js
# ... (add your frontend boilerplate)
cd ..
```

Adapt this example for your chosen technology stack.

### Task 3: Testing and Validation Script (30 minutes)

Create a third script (`validate-setup.sh`) that:

1. **Verifies the environment setup worked correctly**
2. **Runs basic tests:**
   - Check if applications start successfully
   - Verify API endpoints respond
   - Test database connectivity  
   - Validate configuration files

**Example validation checks:**
```bash
# Test backend health endpoint
print_status "Testing backend health endpoint..."
if curl -f http://localhost:3001/api/health > /dev/null 2>&1; then
    print_status "✓ Backend health endpoint responding"
else
    print_error "✗ Backend health endpoint not responding"
fi

# Test frontend loads
print_status "Testing frontend accessibility..."
if curl -f http://localhost:3000 > /dev/null 2>&1; then  
    print_status "✓ Frontend accessible"
else
    print_error "✗ Frontend not accessible"
fi

# Check file structure
print_status "Validating project structure..."
required_dirs=("frontend" "backend" "database" "docs" "scripts")
for dir in "${required_dirs[@]}"; do
    if [ -d "$dir" ]; then
        print_status "✓ $dir directory exists"
    else
        print_error "✗ $dir directory missing"
    fi
done
```

## Deliverables

1. **Three working scripts:**
   - `setup-dev-env.sh` - Environment setup
   - `bootstrap-project.sh` - Project initialization  
   - `validate-setup.sh` - Setup verification

2. **Documentation:**
   - `README.md` with usage instructions
   - List of prerequisites and dependencies
   - Troubleshooting guide for common issues

3. **Project Structure:**
   - Working application skeleton
   - Basic configuration files
   - Initial .gitignore file

## Success Criteria

- Scripts run without errors on a fresh system
- Created project structure matches your design from Exercise 2
- Applications start and respond to basic requests
- All validation checks pass
- Scripts provide clear, helpful output messages
- Error handling for common failure scenarios

## Testing Your Scripts

Test your scripts by:
1. Running them on a clean environment (VM or container)
2. Intentionally breaking things and testing error handling
3. Having a colleague run them on their machine

## Integration with Future Modules

These scripts will be enhanced throughout the course:

- **Module 2:** Version control these scripts and add Git hooks
- **Module 3:** Containerize the applications instead of local setup  
- **Module 4:** Deploy to Kubernetes instead of local environment
- **Module 5:** Integrate with Jenkins for automated execution
- **Module 6:** Use GitOps principles for configuration management
- **Module 7:** Use Terraform to provision infrastructure
- **Module 8:** Add monitoring and alerting to the validation

## Advanced Challenges

1. **Cross-Platform Support:** Make your scripts work on Linux, macOS, and Windows
2. **Configuration Management:** Add support for environment-specific configurations
3. **Dependency Versions:** Pin specific versions of tools and dependencies
4. **Health Monitoring:** Add more sophisticated health checks
5. **Cleanup Script:** Create a script to cleanly remove everything

## Real-World Application

Consider these enterprise scenarios:
- New team member onboarding (reduce setup time from days to minutes)
- Consistent development environments across teams
- Automated testing environment provisioning
- Disaster recovery procedures
- Environment troubleshooting and validation

## Reflection Questions

After completing the exercise, answer:
1. What manual steps did your scripts eliminate?
2. How much time would this save a new developer?
3. What could go wrong with your scripts in production?
4. How would you make these scripts more robust?
5. What additional automation would be valuable?
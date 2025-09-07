# Exercise 6: Branch Management Strategies - Long-Living and Feature Branches

**Duration:** 90 minutes  
**Difficulty:** Advanced  
**Type:** Hands-on Implementation + Strategy

## Objective
Master complex branch management strategies including long-living branch maintenance, feature branch lifecycles, and advanced branching patterns used in enterprise environments.

## Prerequisites
- Completed Exercise 5 (Advanced Git Operations)
- Understanding of merge strategies, rebasing, and conflict resolution
- Repository with protected branches and established workflow

## Scenario
You're implementing a comprehensive branching strategy for a production application with multiple environments, release cycles, and development teams. You need to establish and maintain long-living branches while managing numerous feature branches efficiently.

## Branch Management Concepts

### Branch Types to Master:
1. **Long-Living Branches**: main, develop, release branches
2. **Feature Branches**: short-lived development branches
3. **Hotfix Branches**: emergency production fixes
4. **Support Branches**: maintenance for older versions
5. **Integration Branches**: for complex feature integration
6. **Experimental Branches**: proof-of-concept work

## Tasks

### Task 1: Long-Living Branch Architecture (25 minutes)

**1.1 Establish Production Branch Structure**

Set up a comprehensive long-living branch structure:

```bash
# Ensure clean starting state
git checkout main
git pull origin main

# Create and configure long-living branches
echo "Setting up long-living branch structure..."

# 1. Main branch (production)
git checkout main
echo "# Production Environment" >> ENVIRONMENT.md
echo "This branch represents the current production state" >> ENVIRONMENT.md
echo "Version: $(date +%Y.%m.%d)" >> ENVIRONMENT.md
git add ENVIRONMENT.md
git commit -m "docs: establish main branch as production environment"
git push origin main

# 2. Develop branch (integration)
git checkout -b develop
echo "# Development Environment" >> ENVIRONMENT.md
echo "This branch contains the latest development changes" >> ENVIRONMENT.md
echo "Integration testing occurs here before release" >> ENVIRONMENT.md
git add ENVIRONMENT.md
git commit -m "docs: establish develop branch for integration"
git push -u origin develop

# 3. Staging branch (pre-production)
git checkout -b staging
echo "# Staging Environment" >> ENVIRONMENT.md
echo "This branch mirrors production for final testing" >> ENVIRONMENT.md
echo "User acceptance testing occurs here" >> ENVIRONMENT.md
git add ENVIRONMENT.md
git commit -m "docs: establish staging branch for pre-production testing"
git push -u origin staging

# 4. Create branch protection rules (document the settings needed)
cat > .github/BRANCH_PROTECTION.md << 'EOF'
# Branch Protection Configuration

## Main Branch Protection:
- Require pull request reviews: 2 reviewers
- Dismiss stale reviews when new commits are pushed
- Require status checks to pass: CI, security-scan
- Require branches to be up to date before merging
- Include administrators in restrictions
- Allow force pushes: DISABLED
- Allow deletions: DISABLED

## Develop Branch Protection:
- Require pull request reviews: 1 reviewer
- Require status checks to pass: CI, tests
- Require branches to be up to date before merging

## Staging Branch Protection:
- Require pull request reviews: 1 reviewer
- Restrict pushes to specific teams: release-managers
- Require status checks to pass: CI, integration-tests
EOF

git add .github/BRANCH_PROTECTION.md
git commit -m "docs: document required branch protection settings"
git push origin staging
```

**1.2 Long-Living Branch Maintenance Workflow**

Create maintenance procedures for long-living branches:

```bash
# Create maintenance scripts
mkdir -p scripts/branch-maintenance

cat > scripts/branch-maintenance/sync-branches.sh << 'EOF'
#!/bin/bash

echo "=== Branch Maintenance and Synchronization ==="

# Function to safely sync branches
sync_branch() {
    local branch=$1
    local source=$2
    
    echo "Syncing $branch from $source..."
    git checkout $branch
    git pull origin $branch
    
    if [ "$source" != "" ]; then
        git merge $source --no-ff -m "chore: sync $branch with $source"
        git push origin $branch
    fi
}

# Update all branches from remote
git fetch --all --prune

# Sync develop with any approved changes from feature branches (manual)
sync_branch develop

# Sync staging from develop (controlled release process)
# This would typically be done during release preparation
echo "Staging sync requires manual approval - use release process"

# Sync main from staging (production deployment)
# This would typically be done after successful staging validation
echo "Main sync requires release approval - use deployment process"

echo "Branch maintenance completed"
EOF

chmod +x scripts/branch-maintenance/sync-branches.sh

git add scripts/branch-maintenance/
git commit -m "feat: add branch maintenance scripts"
```

### Task 2: Feature Branch Lifecycle Management (30 minutes)

**2.1 Feature Branch Creation and Naming Standards**

Implement comprehensive feature branch workflows:

```bash
# Create feature branch naming standards
cat > .github/FEATURE_BRANCH_STANDARDS.md << 'EOF'
# Feature Branch Standards

## Naming Convention:
- `feature/TICKET-123-short-description`
- `feature/user-authentication-system`
- `bugfix/TICKET-456-login-error`
- `hotfix/critical-security-patch`
- `experiment/new-ui-framework`

## Lifecycle:
1. Branch from: `develop` (unless hotfix from `main`)
2. Regular updates: rebase from develop weekly
3. Testing: all tests must pass before PR
4. Review: minimum 1 reviewer, 2 for critical features
5. Merge: squash merge for clean history
6. Cleanup: delete after successful merge

## Size Guidelines:
- Small: 1-3 days, < 200 lines of code
- Medium: 1 week, < 500 lines of code  
- Large: > 1 week, requires breakdown into smaller features
EOF

git add .github/FEATURE_BRANCH_STANDARDS.md
git commit -m "docs: establish feature branch standards and lifecycle"

# Create feature branch creation script
cat > scripts/create-feature.sh << 'EOF'
#!/bin/bash

if [ $# -eq 0 ]; then
    echo "Usage: $0 <feature-name> [ticket-number]"
    echo "Example: $0 user-authentication PROJ-123"
    exit 1
fi

FEATURE_NAME=$1
TICKET=${2:-""}
BRANCH_NAME="feature/"

if [ ! -z "$TICKET" ]; then
    BRANCH_NAME="${BRANCH_NAME}${TICKET}-${FEATURE_NAME}"
else
    BRANCH_NAME="${BRANCH_NAME}${FEATURE_NAME}"
fi

echo "Creating feature branch: $BRANCH_NAME"

# Ensure we're on develop and up to date
git checkout develop
git pull origin develop

# Create and push feature branch
git checkout -b $BRANCH_NAME
git push -u origin $BRANCH_NAME

echo "Feature branch $BRANCH_NAME created and pushed"
echo "Remember to:"
echo "1. Follow feature branch standards"
echo "2. Keep branch updated with develop"
echo "3. Write tests for your changes"
echo "4. Create PR when ready for review"
EOF

chmod +x scripts/create-feature.sh
```

**2.2 Multiple Feature Branch Scenarios**

Create and manage multiple feature branches simultaneously:

```bash
# Create multiple feature branches to demonstrate parallel development
./scripts/create-feature.sh user-authentication AUTH-001
./scripts/create-feature.sh product-catalog PROD-002  
./scripts/create-feature.sh payment-integration PAY-003

# Work on Authentication feature
git checkout feature/AUTH-001-user-authentication

mkdir -p backend/src/auth
cat > backend/src/auth/auth.controller.js << 'EOF'
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');

class AuthController {
    async login(req, res) {
        try {
            const { email, password } = req.body;
            
            // Validate input
            if (!email || !password) {
                return res.status(400).json({ error: 'Email and password required' });
            }

            // Find user (mock implementation)
            const user = await this.findUserByEmail(email);
            if (!user) {
                return res.status(401).json({ error: 'Invalid credentials' });
            }

            // Verify password
            const validPassword = await bcrypt.compare(password, user.password);
            if (!validPassword) {
                return res.status(401).json({ error: 'Invalid credentials' });
            }

            // Generate JWT token
            const token = jwt.sign(
                { userId: user.id, email: user.email },
                process.env.JWT_SECRET,
                { expiresIn: '24h' }
            );

            res.json({
                message: 'Login successful',
                token,
                user: {
                    id: user.id,
                    email: user.email,
                    name: user.name
                }
            });
        } catch (error) {
            console.error('Login error:', error);
            res.status(500).json({ error: 'Internal server error' });
        }
    }

    async findUserByEmail(email) {
        // Mock user database lookup
        // In real application, this would query the database
        const mockUsers = [
            {
                id: 1,
                email: 'user@example.com',
                password: await bcrypt.hash('password123', 10),
                name: 'Test User'
            }
        ];
        
        return mockUsers.find(u => u.email === email);
    }
}

module.exports = AuthController;
EOF

git add backend/src/auth/
git commit -m "feat(auth): implement user authentication controller

- Add login endpoint with JWT token generation
- Include password hashing with bcrypt
- Add input validation and error handling
- Mock user database for development testing

Relates to AUTH-001"

# Work on Product Catalog feature
git checkout feature/PROD-002-product-catalog

mkdir -p backend/src/products
cat > backend/src/products/product.model.js << 'EOF'
class Product {
    constructor(data) {
        this.id = data.id;
        this.name = data.name;
        this.description = data.description;
        this.price = data.price;
        this.category = data.category;
        this.inStock = data.inStock || true;
        this.createdAt = data.createdAt || new Date();
        this.updatedAt = data.updatedAt || new Date();
    }

    // Validation methods
    validate() {
        const errors = [];
        
        if (!this.name || this.name.length < 3) {
            errors.push('Product name must be at least 3 characters');
        }
        
        if (!this.price || this.price <= 0) {
            errors.push('Product price must be greater than 0');
        }
        
        if (!this.category) {
            errors.push('Product category is required');
        }
        
        return errors;
    }

    // Business logic methods
    applyDiscount(percentage) {
        if (percentage > 0 && percentage <= 100) {
            this.price = this.price * (1 - percentage / 100);
            this.updatedAt = new Date();
        }
        return this;
    }

    updateStock(inStock) {
        this.inStock = inStock;
        this.updatedAt = new Date();
        return this;
    }

    toJSON() {
        return {
            id: this.id,
            name: this.name,
            description: this.description,
            price: this.price,
            category: this.category,
            inStock: this.inStock,
            createdAt: this.createdAt,
            updatedAt: this.updatedAt
        };
    }
}

module.exports = Product;
EOF

git add backend/src/products/
git commit -m "feat(products): implement product model with validation

- Add Product class with data validation
- Include business logic for discounts and stock management
- Add JSON serialization for API responses
- Include timestamp tracking for audit trail

Relates to PROD-002"

# Work on Payment Integration feature
git checkout feature/PAY-003-payment-integration

mkdir -p backend/src/payments
cat > backend/src/payments/payment.service.js << 'EOF'
const stripe = require('stripe')(process.env.STRIPE_SECRET_KEY);

class PaymentService {
    async createPaymentIntent(amount, currency = 'usd', metadata = {}) {
        try {
            const paymentIntent = await stripe.paymentIntents.create({
                amount: Math.round(amount * 100), // Convert to cents
                currency,
                automatic_payment_methods: {
                    enabled: true,
                },
                metadata
            });

            return {
                clientSecret: paymentIntent.client_secret,
                paymentIntentId: paymentIntent.id,
                amount: amount,
                currency: currency.toUpperCase()
            };
        } catch (error) {
            console.error('Payment intent creation failed:', error);
            throw new Error('Failed to create payment intent');
        }
    }

    async confirmPayment(paymentIntentId) {
        try {
            const paymentIntent = await stripe.paymentIntents.retrieve(paymentIntentId);
            
            if (paymentIntent.status === 'succeeded') {
                return {
                    status: 'success',
                    paymentIntent: {
                        id: paymentIntent.id,
                        amount: paymentIntent.amount / 100,
                        currency: paymentIntent.currency.toUpperCase(),
                        status: paymentIntent.status
                    }
                };
            }
            
            return {
                status: paymentIntent.status,
                paymentIntent: {
                    id: paymentIntent.id,
                    amount: paymentIntent.amount / 100,
                    currency: paymentIntent.currency.toUpperCase(),
                    status: paymentIntent.status
                }
            };
        } catch (error) {
            console.error('Payment confirmation failed:', error);
            throw new Error('Failed to confirm payment');
        }
    }

    async refundPayment(paymentIntentId, amount = null) {
        try {
            const refundData = { payment_intent: paymentIntentId };
            if (amount) {
                refundData.amount = Math.round(amount * 100);
            }

            const refund = await stripe.refunds.create(refundData);

            return {
                refundId: refund.id,
                amount: refund.amount / 100,
                currency: refund.currency.toUpperCase(),
                status: refund.status
            };
        } catch (error) {
            console.error('Payment refund failed:', error);
            throw new Error('Failed to process refund');
        }
    }
}

module.exports = PaymentService;
EOF

git add backend/src/payments/
git commit -m "feat(payments): implement Stripe payment integration

- Add payment intent creation and confirmation
- Include refund functionality for customer service
- Add proper error handling and logging
- Convert between dollars and cents for Stripe API
- Include metadata support for order tracking

Relates to PAY-003"
```

### Task 3: Branch Integration Strategies (20 minutes)

**3.1 Feature Branch Integration and Conflicts**

Demonstrate integration challenges and resolution:

```bash
# Create integration conflicts between features
git checkout feature/AUTH-001-user-authentication
# Add user model that might conflict with products
mkdir -p backend/src/models
cat > backend/src/models/user.model.js << 'EOF'
class User {
    constructor(data) {
        this.id = data.id;
        this.email = data.email;
        this.name = data.name;
        this.password = data.password;
        this.role = data.role || 'user';
        this.createdAt = data.createdAt || new Date();
        this.updatedAt = data.updatedAt || new Date();
    }

    // Common validation (might conflict with product validation)
    validate() {
        const errors = [];
        
        if (!this.email || !this.email.includes('@')) {
            errors.push('Valid email is required');
        }
        
        if (!this.name || this.name.length < 2) {
            errors.push('Name must be at least 2 characters');
        }
        
        return errors;
    }
}

module.exports = User;
EOF

git add backend/src/models/
git commit -m "feat(auth): add user model with validation

- Add User class for authentication system
- Include role-based access control foundation
- Add email and name validation
- Prepare for database integration

Relates to AUTH-001"

# Create shared utility that both features need
git checkout develop
mkdir -p backend/src/utils
cat > backend/src/utils/validation.js << 'EOF'
class ValidationUtils {
    static validateEmail(email) {
        const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
        return emailRegex.test(email);
    }

    static validateRequired(value, fieldName) {
        if (!value || (typeof value === 'string' && value.trim() === '')) {
            return `${fieldName} is required`;
        }
        return null;
    }

    static validateLength(value, min, max, fieldName) {
        if (value && (value.length < min || value.length > max)) {
            return `${fieldName} must be between ${min} and ${max} characters`;
        }
        return null;
    }
}

module.exports = ValidationUtils;
EOF

git add backend/src/utils/
git commit -m "feat(utils): add shared validation utilities

- Add email validation with regex
- Add required field validation
- Add length validation for strings
- Create reusable utility class for all features"

git push origin develop

# Now update each feature branch with the shared utility
git checkout feature/AUTH-001-user-authentication
git rebase develop  # Get the shared utilities

# Update user model to use shared validation
cat > backend/src/models/user.model.js << 'EOF'
const ValidationUtils = require('../utils/validation');

class User {
    constructor(data) {
        this.id = data.id;
        this.email = data.email;
        this.name = data.name;
        this.password = data.password;
        this.role = data.role || 'user';
        this.createdAt = data.createdAt || new Date();
        this.updatedAt = data.updatedAt || new Date();
    }

    validate() {
        const errors = [];
        
        // Use shared validation utilities
        const emailError = ValidationUtils.validateRequired(this.email, 'Email');
        if (emailError) errors.push(emailError);
        else if (!ValidationUtils.validateEmail(this.email)) {
            errors.push('Valid email format is required');
        }
        
        const nameError = ValidationUtils.validateRequired(this.name, 'Name');
        if (nameError) errors.push(nameError);
        else {
            const lengthError = ValidationUtils.validateLength(this.name, 2, 50, 'Name');
            if (lengthError) errors.push(lengthError);
        }
        
        return errors;
    }
}

module.exports = User;
EOF

git add backend/src/models/user.model.js
git commit -m "refactor(auth): use shared validation utilities in user model

- Integrate with ValidationUtils for consistent validation
- Maintain existing validation logic with improved implementation
- Reduce code duplication across models

Relates to AUTH-001"
```

### Task 4: Hotfix Branch Management (15 minutes)

**4.1 Emergency Hotfix Workflow**

Demonstrate critical production hotfix process:

```bash
# Simulate critical production bug discovery
git checkout main
git checkout -b hotfix/critical-auth-vulnerability

# Create security fix
cat > backend/src/auth/security-middleware.js << 'EOF'
const rateLimit = require('express-rate-limit');
const helmet = require('helmet');

// Rate limiting for authentication endpoints
const authRateLimit = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 5, // Limit each IP to 5 requests per windowMs
    message: 'Too many authentication attempts, please try again later',
    standardHeaders: true,
    legacyHeaders: false,
});

// Security headers middleware
const securityHeaders = helmet({
    contentSecurityPolicy: {
        directives: {
            defaultSrc: ["'self'"],
            styleSrc: ["'self'", "'unsafe-inline'"],
            scriptSrc: ["'self'"],
            imgSrc: ["'self'", "data:", "https:"],
        },
    },
    hsts: {
        maxAge: 31536000,
        includeSubDomains: true,
        preload: true
    }
});

module.exports = {
    authRateLimit,
    securityHeaders
};
EOF

git add backend/src/auth/security-middleware.js
git commit -m "fix: add critical security middleware for authentication

SECURITY FIX: CVE-2024-001
- Add rate limiting to prevent brute force attacks
- Implement security headers with Helmet
- Add CSP to prevent XSS attacks
- Configure HSTS for secure connections

URGENT: Deploy immediately to production"

# Create version bump for hotfix
echo "1.2.1" > VERSION
git add VERSION
git commit -m "chore: bump version to 1.2.1 for security hotfix"

# Merge hotfix to main (production)
git checkout main
git merge --no-ff hotfix/critical-auth-vulnerability -m "hotfix: critical security vulnerability fix v1.2.1

Merged hotfix branch containing:
- Authentication rate limiting
- Security headers implementation
- CSP and HSTS configuration

Addresses: CVE-2024-001
Release: v1.2.1"

git tag -a v1.2.1 -m "Hotfix release v1.2.1 - Critical security update"

# Cherry-pick hotfix to develop and feature branches
git checkout develop
git cherry-pick main~1^..main  # Pick the hotfix commits

# Apply to active feature branches
git checkout feature/AUTH-001-user-authentication
git cherry-pick main~1^..main  # Apply security fixes to auth feature

git checkout feature/PROD-002-product-catalog
git cherry-pick main~1^..main  # Apply security fixes to product feature

git checkout feature/PAY-003-payment-integration
git cherry-pick main~1^..main  # Apply security fixes to payment feature

# Clean up hotfix branch
git branch -d hotfix/critical-auth-vulnerability
```

## Deliverables

1. **Long-Living Branch Structure:**
   - main, develop, staging branches with proper configuration
   - Branch protection documentation and settings
   - Maintenance scripts and procedures

2. **Feature Branch Examples:**
   - Multiple active feature branches with realistic development work
   - Proper naming conventions and lifecycle documentation
   - Integration examples with conflict resolution

3. **Branch Management Scripts:**
   - Automated branch creation with standards enforcement
   - Synchronization scripts for long-living branches
   - Maintenance and cleanup automation

4. **Hotfix Workflow Documentation:**
   - Complete hotfix lifecycle from discovery to deployment
   - Multi-branch application strategies
   - Emergency response procedures

## Success Criteria

- Established sustainable long-living branch architecture
- Demonstrated parallel feature development workflows
- Successfully managed complex branch integrations
- Executed emergency hotfix deployment process
- Created reusable branch management procedures
- Documented comprehensive branching strategies

## Verification Commands

```bash
# Verify branch structure
git branch -a
git log --oneline --graph --all -20

# Verify hotfix application
git log --grep="hotfix" --oneline
git tag -l

# Verify feature branch integration
git log --merge
git show-branch feature/* develop

# Check branch protection (requires GitHub CLI)
gh api repos/:owner/:repo/branches/main/protection

# Verify script functionality
./scripts/create-feature.sh test-feature TEST-001
./scripts/branch-maintenance/sync-branches.sh
```

## Integration with DevOps Pipeline

These branch management strategies integrate with:
- **CI/CD Triggers**: Different pipelines for different branch types
- **Environment Deployments**: Branch-to-environment mapping
- **Release Management**: Automated version bumping and tagging
- **Quality Gates**: Branch-specific testing and approval requirements

## Real-World Enterprise Applications

**Common Scenarios:**
- **Multi-team Development**: Coordinating feature development across teams
- **Release Planning**: Managing release branches and feature freezes
- **Production Support**: Handling hotfixes while maintaining development velocity
- **Compliance Requirements**: Maintaining audit trails and approval workflows
- **Scaling Teams**: Onboarding new developers with clear branching standards

This comprehensive branch management exercise prepares students for complex enterprise development environments where proper branching strategy is critical for team productivity and production stability!
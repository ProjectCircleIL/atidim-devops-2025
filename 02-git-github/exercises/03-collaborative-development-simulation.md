# Exercise 3: Collaborative Development Simulation

**Duration:** 90 minutes  
**Difficulty:** Intermediate to Advanced  
**Type:** Hands-on Simulation + Conflict Resolution

## Objective
Simulate realistic collaborative development scenarios including merge conflicts, code reviews, and team coordination. Practice skills needed for professional team environments.

## Prerequisites
- Completed Exercise 2 (Branching Strategy Implementation)  
- Working repository with develop and main branches
- At least one implemented feature from Exercise 2

## Scenario
You're working on a development team building your 3-tier application. Multiple developers are working on different features simultaneously. You'll simulate this by:
1. Playing multiple "developer" roles
2. Creating intentional conflicts
3. Practicing conflict resolution
4. Implementing realistic team scenarios

## Team Roles Simulation

You'll simulate these roles:
- **Developer A (Frontend):** Working on UI components
- **Developer B (Backend):** Working on API endpoints  
- **Developer C (DevOps):** Working on configuration and tooling
- **Tech Lead (You):** Reviewing code and resolving conflicts

## Tasks

### Task 1: Multi-Developer Feature Development (30 minutes)

**1.1 Simulate Developer A - Frontend Feature**
```bash
# Switch to "Developer A" persona
git config user.name "Alice Frontend"
git config user.email "alice@yourproject.com"

git checkout develop
git pull origin develop
git checkout -b feature/user-login-form

# Create login form component
mkdir -p frontend/src/components/auth
```

Create `frontend/src/components/auth/LoginForm.jsx`:
```jsx
import React, { useState } from 'react';

const LoginForm = () => {
    const [credentials, setCredentials] = useState({
        username: '',
        password: ''
    });
    const [loading, setLoading] = useState(false);

    const handleSubmit = async (e) => {
        e.preventDefault();
        setLoading(true);
        
        try {
            const response = await fetch('/api/auth/login', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                },
                body: JSON.stringify(credentials),
            });
            
            if (response.ok) {
                const data = await response.json();
                localStorage.setItem('token', data.token);
                window.location.href = '/dashboard';
            } else {
                alert('Login failed');
            }
        } catch (error) {
            console.error('Login error:', error);
            alert('Login error occurred');
        }
        
        setLoading(false);
    };

    return (
        <form onSubmit={handleSubmit} className="login-form">
            <h2>Login</h2>
            <div>
                <label>Username:</label>
                <input
                    type="text"
                    value={credentials.username}
                    onChange={(e) => setCredentials({
                        ...credentials,
                        username: e.target.value
                    })}
                    required
                />
            </div>
            <div>
                <label>Password:</label>
                <input
                    type="password"
                    value={credentials.password}
                    onChange={(e) => setCredentials({
                        ...credentials,
                        password: e.target.value
                    })}
                    required
                />
            </div>
            <button type="submit" disabled={loading}>
                {loading ? 'Logging in...' : 'Login'}
            </button>
        </form>
    );
};

export default LoginForm;
```

```bash
# Commit Developer A's work
git add .
git commit -m "feat(frontend): add user login form component

- Create LoginForm component with state management
- Add form validation and error handling
- Integrate with backend authentication API
- Add loading states for better UX"

git push -u origin feature/user-login-form
```

**1.2 Simulate Developer B - Backend Feature**
```bash
# Switch to "Developer B" persona  
git config user.name "Bob Backend"
git config user.email "bob@yourproject.com"

git checkout develop
git pull origin develop
git checkout -b feature/user-authentication-api

# Create authentication routes
mkdir -p backend/src/routes/auth
```

Create `backend/src/routes/auth.js`:
```javascript
const express = require('express');
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');
const router = express.Router();

// Mock user database (in real app, this would be a proper database)
const users = [
    {
        id: 1,
        username: 'admin',
        email: 'admin@example.com',
        password: '$2b$10$example_hash_here' // 'password123'
    },
    {
        id: 2,
        username: 'testuser',
        email: 'test@example.com',
        password: '$2b$10$example_hash_here' // 'password123'
    }
];

// Login endpoint
router.post('/auth/login', async (req, res) => {
    try {
        const { username, password } = req.body;
        
        if (!username || !password) {
            return res.status(400).json({ 
                error: 'Username and password are required' 
            });
        }

        // Find user
        const user = users.find(u => u.username === username);
        if (!user) {
            return res.status(401).json({ error: 'Invalid credentials' });
        }

        // Check password (in real app, use bcrypt.compare)
        // const validPassword = await bcrypt.compare(password, user.password);
        const validPassword = password === 'password123'; // Simplified for demo
        
        if (!validPassword) {
            return res.status(401).json({ error: 'Invalid credentials' });
        }

        // Generate JWT
        const token = jwt.sign(
            { userId: user.id, username: user.username },
            process.env.JWT_SECRET || 'dev-secret-key',
            { expiresIn: '24h' }
        );

        res.json({
            message: 'Login successful',
            token,
            user: {
                id: user.id,
                username: user.username,
                email: user.email
            }
        });
    } catch (error) {
        console.error('Login error:', error);
        res.status(500).json({ error: 'Internal server error' });
    }
});

// Logout endpoint
router.post('/auth/logout', (req, res) => {
    // In a real app, you might blacklist the token
    res.json({ message: 'Logged out successfully' });
});

// Verify token middleware
const verifyToken = (req, res, next) => {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
        return res.status(401).json({ error: 'Access denied' });
    }

    try {
        const decoded = jwt.verify(token, process.env.JWT_SECRET || 'dev-secret-key');
        req.user = decoded;
        next();
    } catch (error) {
        res.status(401).json({ error: 'Invalid token' });
    }
};

// Protected route example
router.get('/auth/me', verifyToken, (req, res) => {
    const user = users.find(u => u.id === req.user.userId);
    if (!user) {
        return res.status(404).json({ error: 'User not found' });
    }

    res.json({
        id: user.id,
        username: user.username,
        email: user.email
    });
});

module.exports = router;
```

```bash
# Also update package.json to include new dependencies
# This will cause a merge conflict later!
cd backend
npm install bcrypt jsonwebtoken
cd ..

# Commit Developer B's work
git add .
git commit -m "feat(backend): implement user authentication API

- Add login endpoint with JWT token generation
- Add logout endpoint
- Implement token verification middleware
- Add protected route for user profile
- Add input validation and error handling
- Install required dependencies (bcrypt, jsonwebtoken)"

git push -u origin feature/user-authentication-api
```

**1.3 Simulate Developer C - DevOps Configuration**
```bash
# Switch to "Developer C" persona
git config user.name "Charlie DevOps"
git config user.email "charlie@yourproject.com"

git checkout develop
git pull origin develop
git checkout -b feature/docker-development-setup

# Create Docker development environment
```

Create `docker-compose.dev.yml`:
```yaml
version: '3.8'

services:
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile.dev
    ports:
      - "3001:3001"
    environment:
      - NODE_ENV=development
      - JWT_SECRET=dev-jwt-secret-key-change-in-production
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_NAME=devops_app_dev
      - DB_USER=postgres
      - DB_PASS=devpass123
    volumes:
      - ./backend:/app
      - /app/node_modules
    depends_on:
      - postgres
    command: npm run dev

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile.dev
    ports:
      - "3000:3000"
    environment:
      - REACT_APP_API_URL=http://localhost:3001
    volumes:
      - ./frontend:/app
      - /app/node_modules
    command: npm start

  postgres:
    image: postgres:13
    environment:
      - POSTGRES_DB=devops_app_dev
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=devpass123
    ports:
      - "5432:5432"
    volumes:
      - postgres_dev_data:/var/lib/postgresql/data
      - ./database/init.sql:/docker-entrypoint-initdb.d/init.sql

  redis:
    image: redis:6-alpine
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes
    volumes:
      - redis_dev_data:/data

volumes:
  postgres_dev_data:
  redis_dev_data:
```

Create `backend/Dockerfile.dev`:
```dockerfile
FROM node:16-alpine

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm install

# Copy source code
COPY . .

# Expose port
EXPOSE 3001

# Start development server
CMD ["npm", "run", "dev"]
```

```bash
# Commit Developer C's work
git add .
git commit -m "feat(docker): add Docker development environment setup

- Add docker-compose.dev.yml for local development
- Configure backend, frontend, postgres, and redis services
- Add Dockerfile.dev for backend development
- Set up volume mounts for hot reloading
- Configure environment variables for development
- Add database initialization support"

git push -u origin feature/docker-development-setup
```

### Task 2: Create and Resolve Merge Conflicts (30 minutes)

**2.1 Simulate Conflicting Changes**

Let's create a realistic conflict scenario where multiple developers modify the same files:

```bash
# Developer A updates package.json (conflicting with Developer B's changes)
git checkout feature/user-login-form

# Update package.json in frontend
cd frontend
npm install axios react-router-dom
cd ..

# Also update the main README.md
cat >> README.md << 'EOF'

## Authentication
The application includes user authentication with login/logout functionality.

### Frontend Features:
- Login form component
- Token-based authentication
- Protected routes
- User session management
EOF

git add .
git commit -m "feat(frontend): add authentication dependencies and documentation

- Install axios for API calls
- Install react-router-dom for routing
- Update README with authentication features"

git push origin feature/user-login-form
```

**2.2 Create Merge Conflict Scenario**
```bash
# Developer B also updates README.md differently
git checkout feature/user-authentication-api

cat >> README.md << 'EOF'

## API Documentation
The backend provides RESTful API endpoints for authentication and user management.

### Authentication Endpoints:
- POST /api/auth/login - User login
- POST /api/auth/logout - User logout
- GET /api/auth/me - Get current user profile

### Security:
- JWT token-based authentication
- Password hashing with bcrypt
- Input validation and sanitization
EOF

git add .
git commit -m "docs(api): add comprehensive API documentation

- Document authentication endpoints
- Add security implementation details
- Include usage examples for developers"

git push origin feature/user-authentication-api
```

**2.3 Attempt Merge and Resolve Conflicts**
```bash
# Switch to tech lead persona
git config user.name "Your Name"
git config user.email "your.email@example.com"

# Try to merge features into develop (this will cause conflicts)
git checkout develop
git pull origin develop

# Merge first feature (should work)
git merge feature/user-login-form --no-ff
echo "First merge completed successfully"

# Merge second feature (this should create conflicts)
git merge feature/user-authentication-api --no-ff
```

You should see a merge conflict in README.md. Let's resolve it:

```bash
# Check conflict status
git status

# Open README.md and resolve conflicts manually
# The file will show conflict markers like:
# <<<<<<< HEAD
# (Developer A's changes)
# =======
# (Developer B's changes)  
# >>>>>>> feature/user-authentication-api
```

Resolve the conflict by creating a comprehensive combined version:
```markdown
## Authentication
The application includes user authentication with login/logout functionality.

### Frontend Features:
- Login form component
- Token-based authentication
- Protected routes
- User session management

### API Documentation
The backend provides RESTful API endpoints for authentication and user management.

### Authentication Endpoints:
- POST /api/auth/login - User login
- POST /api/auth/logout - User logout
- GET /api/auth/me - Get current user profile

### Security:
- JWT token-based authentication
- Password hashing with bcrypt
- Input validation and sanitization
```

```bash
# Stage the resolved conflict
git add README.md

# Complete the merge
git commit -m "merge: resolve authentication documentation conflicts

- Combine frontend and backend authentication documentation
- Preserve all features from both branches
- Create comprehensive authentication guide"

# Merge the third feature
git merge feature/docker-development-setup --no-ff

# Push updated develop branch
git push origin develop
```

### Task 3: Advanced Collaboration Scenarios (30 minutes)

**3.1 Simulate Code Review Process**

Create a feature that needs review and improvement:

```bash
git checkout develop
git checkout -b feature/user-dashboard

# Create a basic dashboard (with intentional issues for review)
mkdir -p frontend/src/components/dashboard
```

Create `frontend/src/components/dashboard/Dashboard.jsx`:
```jsx
// Intentionally poor code for review practice
import React, { useState, useEffect } from 'react';

function Dashboard() {
    const [user, setUser] = useState(null);
    const [data, setData] = useState([]);
    
    useEffect(() => {
        // BAD: No error handling
        fetch('/api/auth/me', {
            headers: {
                'Authorization': 'Bearer ' + localStorage.getItem('token')
            }
        })
        .then(response => response.json())
        .then(data => setUser(data));

        // BAD: Hardcoded data, no loading state
        setData([
            { id: 1, name: 'Item 1', value: 100 },
            { id: 2, name: 'Item 2', value: 200 },
            { id: 3, name: 'Item 3', value: 300 }
        ]);
    }, []);

    // BAD: No loading state, poor error handling
    return (
        <div>
            <h1>Dashboard</h1>
            {user && <p>Welcome, {user.username}!</p>}
            <div>
                {data.map(item => (
                    // BAD: No key prop, inline styles
                    <div style={{border: '1px solid black', margin: '5px'}}>
                        <h3>{item.name}</h3>
                        <p>Value: {item.value}</p>
                    </div>
                ))}
            </div>
        </div>
    );
}

export default Dashboard;
```

```bash
git add .
git commit -m "feat(dashboard): add basic user dashboard component

- Create dashboard component with user greeting
- Display data items in card format
- Integrate with authentication API"

git push -u origin feature/user-dashboard
```

**3.2 Simulate Code Review and Improvements**

Now "review" the code and create improvements:

```bash
# Create review branch with improvements
git checkout -b feature/user-dashboard-improvements

# Improve the Dashboard component
```

Create improved `frontend/src/components/dashboard/Dashboard.jsx`:
```jsx
import React, { useState, useEffect, useCallback } from 'react';
import './Dashboard.css';

const Dashboard = () => {
    const [user, setUser] = useState(null);
    const [data, setData] = useState([]);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState(null);

    const fetchUserData = useCallback(async () => {
        try {
            const token = localStorage.getItem('token');
            if (!token) {
                throw new Error('No authentication token found');
            }

            const response = await fetch('/api/auth/me', {
                headers: {
                    'Authorization': `Bearer ${token}`,
                    'Content-Type': 'application/json',
                },
            });

            if (!response.ok) {
                throw new Error(`HTTP error! status: ${response.status}`);
            }

            const userData = await response.json();
            setUser(userData);
        } catch (err) {
            console.error('Failed to fetch user data:', err);
            setError(err.message);
        }
    }, []);

    const fetchDashboardData = useCallback(async () => {
        try {
            // In a real app, this would be an API call
            const mockData = [
                { id: 1, name: 'Active Projects', value: 5, color: 'blue' },
                { id: 2, name: 'Completed Tasks', value: 23, color: 'green' },
                { id: 3, name: 'Pending Reviews', value: 7, color: 'orange' }
            ];
            
            // Simulate network delay
            await new Promise(resolve => setTimeout(resolve, 500));
            setData(mockData);
        } catch (err) {
            console.error('Failed to fetch dashboard data:', err);
            setError(err.message);
        }
    }, []);

    useEffect(() => {
        const loadData = async () => {
            setLoading(true);
            await Promise.all([fetchUserData(), fetchDashboardData()]);
            setLoading(false);
        };

        loadData();
    }, [fetchUserData, fetchDashboardData]);

    if (loading) {
        return (
            <div className="dashboard loading">
                <div className="spinner">Loading dashboard...</div>
            </div>
        );
    }

    if (error) {
        return (
            <div className="dashboard error">
                <h2>Error Loading Dashboard</h2>
                <p>{error}</p>
                <button onClick={() => window.location.reload()}>
                    Retry
                </button>
            </div>
        );
    }

    return (
        <div className="dashboard">
            <header className="dashboard-header">
                <h1>Dashboard</h1>
                {user && (
                    <div className="user-welcome">
                        <p>Welcome back, <strong>{user.username}</strong>!</p>
                        <small>Last login: {new Date().toLocaleDateString()}</small>
                    </div>
                )}
            </header>

            <main className="dashboard-content">
                <div className="stats-grid">
                    {data.map(item => (
                        <div 
                            key={item.id} 
                            className={`stat-card ${item.color}`}
                        >
                            <h3>{item.name}</h3>
                            <p className="stat-value">{item.value}</p>
                        </div>
                    ))}
                </div>
            </main>
        </div>
    );
};

export default Dashboard;
```

Create `frontend/src/components/dashboard/Dashboard.css`:
```css
.dashboard {
    max-width: 1200px;
    margin: 0 auto;
    padding: 20px;
}

.dashboard.loading {
    display: flex;
    justify-content: center;
    align-items: center;
    min-height: 400px;
}

.spinner {
    font-size: 18px;
    color: #666;
}

.dashboard.error {
    text-align: center;
    padding: 40px;
}

.dashboard-header {
    margin-bottom: 30px;
    border-bottom: 1px solid #eee;
    padding-bottom: 20px;
}

.user-welcome {
    margin-top: 10px;
}

.user-welcome small {
    color: #666;
}

.stats-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
    gap: 20px;
}

.stat-card {
    background: white;
    border-radius: 8px;
    padding: 20px;
    box-shadow: 0 2px 4px rgba(0,0,0,0.1);
    border-left: 4px solid;
}

.stat-card.blue {
    border-left-color: #3498db;
}

.stat-card.green {
    border-left-color: #2ecc71;
}

.stat-card.orange {
    border-left-color: #f39c12;
}

.stat-card h3 {
    margin: 0 0 10px 0;
    color: #333;
    font-size: 16px;
}

.stat-value {
    font-size: 32px;
    font-weight: bold;
    color: #2c3e50;
    margin: 0;
}
```

```bash
git add .
git commit -m "refactor(dashboard): improve code quality and user experience

Code improvements:
- Add proper error handling and loading states
- Implement async/await pattern for better readability
- Add proper key props for React elements
- Extract styles to separate CSS file
- Add responsive design with CSS Grid

UX improvements:
- Add loading spinner during data fetch
- Display error messages with retry option
- Improve visual design with cards and colors
- Add user welcome message with timestamp

Technical improvements:
- Use useCallback to prevent unnecessary re-renders
- Implement proper error boundaries
- Add TypeScript-ready prop patterns
- Follow React best practices"

git push -u origin feature/user-dashboard-improvements
```

**3.3 Merge Improved Version**
```bash
git checkout develop
git merge feature/user-dashboard-improvements --no-ff
git push origin develop

# Clean up branches
git branch -d feature/user-dashboard
git push origin --delete feature/user-dashboard
```

## Deliverables

1. **Multi-Feature Integration:**
   - Frontend authentication components
   - Backend API endpoints
   - Docker development environment
   - Resolved merge conflicts

2. **Professional Git History:**
   - Clean commit messages following conventions
   - Proper merge commits with --no-ff
   - Evidence of conflict resolution
   - Multiple developer personas in git log

3. **Code Review Simulation:**
   - Initial implementation with intentional issues
   - Improved version addressing code quality
   - Documentation of improvements made

4. **Collaboration Evidence:**
   - Multiple feature branches merged
   - Conflicts identified and resolved
   - Team coordination simulation

## Success Criteria

- Successfully merged at least 4 different features
- Resolved at least one merge conflict properly
- Demonstrated code review and improvement process
- Git history shows realistic team collaboration patterns
- All features work together as integrated application

## Verification Commands

```bash
# View complete git history
git log --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit --all

# Check merged features
git branch --merged develop

# Test integrated application
# Start your application and verify:
# 1. Health endpoint works
# 2. Authentication API responds
# 3. Frontend components render
# 4. Docker environment runs (if implemented)
```

## Integration with Next Exercise

This collaborative foundation prepares for Exercise 4:
- Repository will be configured with branch protection
- Pull request templates will be added
- Automated CI/CD will run on feature branches
- Team workflows will be formalized

## Real-World Applications

This exercise simulates common scenarios you'll encounter:
- **Merge Conflicts:** Multiple developers modifying same files
- **Code Reviews:** Identifying and fixing code quality issues  
- **Feature Integration:** Combining work from different team members
- **Documentation Conflicts:** Resolving competing documentation updates
- **Dependency Management:** Handling conflicting package requirements

## Advanced Challenges

1. **Three-Way Merge Conflicts:** Create conflicts involving three branches
2. **Binary File Conflicts:** Add images and practice resolving binary conflicts
3. **Submodule Conflicts:** If using submodules, practice submodule conflict resolution
4. **Large Team Simulation:** Create 5+ feature branches and merge them strategically

This exercise provides hands-on experience with the collaboration challenges you'll face in professional development environments!
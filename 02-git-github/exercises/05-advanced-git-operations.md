# Exercise 5: Advanced Git Operations - Merges, Rebases, and Cherry-picking

**Duration:** 120 minutes  
**Difficulty:** Advanced  
**Type:** Hands-on Implementation

## Objective
Master advanced Git operations including different merge strategies, interactive rebasing, cherry-picking, and complex conflict resolution scenarios that are essential for professional development workflows.

## Prerequisites
- Completed Exercise 4 (GitHub Repository Configuration)
- Working repository with protected branches
- Understanding of basic Git commands
- Multiple feature branches from previous exercises

## Scenario
You're working in a complex development environment where multiple developers are working on different features simultaneously. You need to master advanced Git operations to handle sophisticated branching scenarios, maintain clean commit history, and manage complex integrations.

## Advanced Git Concepts Covered

### Git Operations to Master:
1. **Merge Strategies**: fast-forward, no-fast-forward, squash merges
2. **Rebase Operations**: interactive rebase, rebase vs merge
3. **Cherry-picking**: selective commit application
4. **Commit Inspection**: log analysis, diff examination, blame tracking
5. **Branch Management**: long-living branches, feature branches, hotfix workflows
6. **Conflict Resolution**: complex merge conflicts, resolution strategies
7. **History Manipulation**: commit amendments, history rewriting

## Tasks

### Task 1: Advanced Merge Strategies (30 minutes)

**1.1 Fast-Forward vs No-Fast-Forward Merges**

Create a scenario to demonstrate different merge strategies:

```bash
# Start fresh with clean state
git checkout develop
git pull origin develop

# Create a simple feature branch for fast-forward demo
git checkout -b feature/fast-forward-demo
echo "Fast forward content" >> fast-forward-file.txt
git add fast-forward-file.txt
git commit -m "feat: add fast-forward demonstration file"
git push -u origin feature/fast-forward-demo

# Perform fast-forward merge
git checkout develop
git merge feature/fast-forward-demo  # This will be fast-forward
git log --oneline --graph -5  # Notice linear history

# Create scenario for no-fast-forward merge
git checkout -b feature/no-fast-forward-demo
echo "No fast-forward content" >> no-ff-file.txt
git add no-ff-file.txt
git commit -m "feat: add no-fast-forward demonstration file"

# Meanwhile, make a change on develop to prevent fast-forward
git checkout develop
echo "Parallel development" >> parallel-dev.txt
git add parallel-dev.txt
git commit -m "feat: parallel development on develop"

# Now merge with explicit no-fast-forward
git checkout develop
git merge --no-ff feature/no-fast-forward-demo -m "merge: integrate no-fast-forward demo feature"
git log --oneline --graph -8  # Notice merge commit created
```

**1.2 Squash Merges**

Practice squash merges for clean history:

```bash
# Create feature branch with multiple commits
git checkout develop
git checkout -b feature/multiple-commits-squash
echo "First change" >> squash-demo.txt
git add squash-demo.txt
git commit -m "feat: first incremental change"

echo "Second change" >> squash-demo.txt
git add squash-demo.txt
git commit -m "feat: second incremental change"

echo "Third change" >> squash-demo.txt
git add squash-demo.txt
git commit -m "feat: third incremental change"

echo "Bug fix" >> squash-demo.txt
git add squash-demo.txt
git commit -m "fix: resolve issue in feature"

git log --oneline -4  # See multiple commits

# Squash merge to develop
git checkout develop
git merge --squash feature/multiple-commits-squash
git status  # Notice changes are staged but not committed
git commit -m "feat: complete feature implementation with squash merge

- Added multiple incremental improvements
- Resolved issues during development
- Consolidated into single commit for clean history"

git log --oneline --graph -5  # Notice single commit instead of multiple
```

### Task 2: Interactive Rebasing Mastery (35 minutes)

**2.1 Interactive Rebase for History Cleanup**

Create a messy commit history and clean it up:

```bash
# Create feature branch with messy history
git checkout develop
git checkout -b feature/interactive-rebase-demo

# Create intentionally messy commits
echo "Initial feature work" > rebase-demo.txt
git add rebase-demo.txt
git commit -m "feat: start feature work"

echo "More work" >> rebase-demo.txt
git add rebase-demo.txt
git commit -m "wip: more stuff"  # Bad commit message

echo "Fix typo" >> rebase-demo.txt
git add rebase-demo.txt
git commit -m "typo fix"  # Should be squashed

echo "Actually complete feature" >> rebase-demo.txt
git add rebase-demo.txt
git commit -m "feat: complete the feature implementation"

echo "Another typo fix" >> rebase-demo.txt
git add rebase-demo.txt
git commit -m "fix typo again"  # Should be squashed

echo "Final documentation" >> README-feature.md
git add README-feature.md
git commit -m "docs: add feature documentation"

# View messy history
git log --oneline -6

# Interactive rebase to clean up history
git rebase -i HEAD~6
```

In the interactive rebase editor, demonstrate these operations:
```
# Example of what to do in interactive rebase:
pick a1b2c3d feat: start feature work
squash d4e5f6g wip: more stuff
squash g7h8i9j typo fix
pick j1k2l3m feat: complete the feature implementation
squash m4n5o6p fix typo again
reword p7q8r9s docs: add feature documentation
```

**2.2 Rebase vs Merge Decision Making**

Create scenarios to practice when to use rebase vs merge:

```bash
# Scenario 1: Feature branch rebase (recommended)
git checkout develop
git checkout -b feature/clean-history
echo "Clean feature work" > clean-feature.txt
git add clean-feature.txt
git commit -m "feat: implement clean feature"

# Simulate develop branch moving forward
git checkout develop
echo "Other developer work" >> other-work.txt
git add other-work.txt
git commit -m "feat: other team member contribution"

# Rebase feature branch onto latest develop
git checkout feature/clean-history
git rebase develop  # Clean, linear history
git log --oneline --graph -5

# Scenario 2: Release branch merge (recommended for traceability)
git checkout develop
git checkout -b release/v1.1.0
echo "v1.1.0" > VERSION
git add VERSION
git commit -m "chore: bump version to 1.1.0"

# Merge release branch to main (preserve merge commit for traceability)
git checkout main
git merge --no-ff release/v1.1.0 -m "release: version 1.1.0 with new features"
git tag -a v1.1.0 -m "Release version 1.1.0"
```

### Task 3: Cherry-picking Operations (25 minutes)

**3.1 Selective Commit Application**

Practice cherry-picking for hotfixes and selective feature application:

```bash
# Create hotfix scenario
git checkout main
git checkout -b hotfix/critical-security-fix
echo "Security improvement" >> security-fix.txt
git add security-fix.txt
git commit -m "fix: resolve critical security vulnerability CVE-2023-001"

echo "Additional security hardening" >> security-fix.txt
git add security-fix.txt
git commit -m "feat: add additional security measures"

# Get commit hashes
SECURITY_FIX_HASH=$(git rev-parse HEAD~1)
SECURITY_FEATURE_HASH=$(git rev-parse HEAD)

# Apply hotfix to develop branch via cherry-pick
git checkout develop
git cherry-pick $SECURITY_FIX_HASH
git log --oneline -3  # See cherry-picked commit

# Apply to main branch as well
git checkout main
git cherry-pick $SECURITY_FIX_HASH

# Create release with cherry-picked security feature
git checkout -b release/v1.0.1
git cherry-pick $SECURITY_FEATURE_HASH
```

**3.2 Cherry-picking with Conflicts**

Create and resolve cherry-pick conflicts:

```bash
# Create conflicting changes
git checkout develop
echo "Develop branch changes" >> conflict-file.txt
git add conflict-file.txt
git commit -m "feat: develop branch modifications"

git checkout feature/clean-history
echo "Feature branch changes" >> conflict-file.txt
git add conflict-file.txt
git commit -m "feat: feature branch modifications"

FEATURE_COMMIT=$(git rev-parse HEAD)

# Try to cherry-pick (this will conflict)
git checkout develop
git cherry-pick $FEATURE_COMMIT
# Resolve conflict manually
echo "Resolved: Both develop and feature changes" > conflict-file.txt
git add conflict-file.txt
git cherry-pick --continue
```

### Task 4: Commit Inspection and Analysis (20 minutes)

**4.1 Advanced Log Analysis**

Master different ways to inspect Git history:

```bash
# Comprehensive log analysis
git log --oneline --graph --all --decorate -10  # Visual branch history
git log --stat -5  # Show files changed in each commit
git log --patch -3  # Show actual changes (diff) in commits
git log --grep="feat"  # Find commits by message content
git log --author="your-name" --since="1 week ago"  # Find commits by author and time
git log --follow -- specific-file.txt  # Track file history across renames

# Advanced commit inspection
git show HEAD  # Show latest commit details
git show HEAD~2  # Show commit 2 steps back
git show --stat HEAD  # Show commit statistics only
git show --name-only HEAD  # Show only changed file names

# Blame and annotate
git blame README.md  # See who changed each line
git annotate README.md  # Alternative to blame

# Find when bug was introduced
git bisect start
git bisect bad HEAD  # Current version has bug
git bisect good v1.0.0  # This version was good
# Git will checkout middle commit, test and mark good/bad
# Continue until bug commit is found
```

**4.2 Diff Analysis**

Practice different diff operations:

```bash
# Various diff operations
git diff  # Unstaged changes
git diff --staged  # Staged changes
git diff HEAD  # All changes since last commit
git diff HEAD~3  # Changes since 3 commits ago
git diff develop..feature/branch-name  # Changes between branches
git diff --name-only develop..feature/branch-name  # Only file names
git diff --word-diff  # Word-level differences
git diff --color-words  # Colored word-level differences

# Advanced diff analysis
git diff --stat develop..main  # Statistical summary
git diff --dirstat develop..main  # Directory change statistics
git difftool  # Use external diff tool (if configured)
```

### Task 5: Complex Conflict Resolution (30 minutes)

**5.1 Multi-way Merge Conflicts**

Create and resolve complex merge conflicts:

```bash
# Create complex conflict scenario
git checkout develop
git checkout -b feature/complex-conflicts

# Create base file
cat > complex-conflict.js << 'EOF'
function calculateTotal(items) {
    let total = 0;
    for (let item of items) {
        total += item.price;
    }
    return total;
}

function applyDiscount(total, discount) {
    return total * (1 - discount);
}
EOF

git add complex-conflict.js
git commit -m "feat: add basic calculation functions"

# Developer A changes (you)
cat > complex-conflict.js << 'EOF'
function calculateTotal(items) {
    let total = 0;
    for (let item of items) {
        total += item.price * item.quantity;  // Added quantity
    }
    return total;
}

function applyDiscount(total, discount) {
    return total * (1 - discount);
}

function calculateTax(total, taxRate) {  // New function
    return total * taxRate;
}
EOF

git add complex-conflict.js
git commit -m "feat: add quantity calculation and tax function"

# Switch to develop and make different changes (simulate Developer B)
git checkout develop
cat > complex-conflict.js << 'EOF'
function calculateTotal(items) {
    let total = 0;
    for (let item of items) {
        if (item.price > 0) {  // Added validation
            total += item.price;
        }
    }
    return total;
}

function applyDiscount(total, discount) {
    if (discount >= 0 && discount <= 1) {  // Added validation
        return total * (1 - discount);
    }
    return total;
}

function formatCurrency(amount) {  // Different new function
    return '$' + amount.toFixed(2);
}
EOF

git add complex-conflict.js
git commit -m "feat: add input validation and currency formatting"

# Attempt merge (will conflict)
git merge feature/complex-conflicts

# The conflict will need manual resolution combining both sets of changes
# Resolve to combine all improvements:
cat > complex-conflict.js << 'EOF'
function calculateTotal(items) {
    let total = 0;
    for (let item of items) {
        if (item.price > 0) {  // Validation from develop
            total += item.price * (item.quantity || 1);  // Quantity from feature
        }
    }
    return total;
}

function applyDiscount(total, discount) {
    if (discount >= 0 && discount <= 1) {  // Validation from develop
        return total * (1 - discount);
    }
    return total;
}

function calculateTax(total, taxRate) {  // From feature branch
    return total * taxRate;
}

function formatCurrency(amount) {  // From develop branch
    return '$' + amount.toFixed(2);
}
EOF

git add complex-conflict.js
git commit -m "merge: resolve complex conflicts combining validation and new features

- Integrated quantity calculation with price validation
- Combined tax calculation and currency formatting features
- Ensured all edge cases are handled properly"
```

**5.2 Binary and Rename Conflicts**

Practice handling different types of conflicts:

```bash
# Rename conflicts
git checkout develop
git mv old-filename.txt new-filename.txt
git commit -m "refactor: rename file for clarity"

git checkout feature/complex-conflicts
echo "Feature changes" >> old-filename.txt
git add old-filename.txt
git commit -m "feat: update old filename content"

# Merge will create rename conflict
git checkout develop
git merge feature/complex-conflicts
# Resolve by keeping renamed file with feature changes
git add new-filename.txt
git rm old-filename.txt
git commit -m "merge: resolve rename conflict preserving feature changes"
```

## Deliverables

1. **Repository with Advanced Git History:**
   - Examples of different merge strategies (fast-forward, no-ff, squash)
   - Clean commit history from interactive rebasing
   - Cherry-picked commits across branches
   - Resolved complex conflicts

2. **Documented Git Operations:**
   - Screenshots or logs showing each type of operation
   - Before/after comparisons of git log output
   - Documentation of resolution strategies used

3. **Branch Management Evidence:**
   - Long-living branches (main, develop) with proper protection
   - Feature branches with different lifecycle patterns
   - Hotfix branches with emergency fix workflows
   - Release branches with version management

4. **Conflict Resolution Examples:**
   - Complex multi-way merge conflict resolution
   - Rename conflict resolution
   - Cherry-pick conflict resolution
   - Documentation of resolution strategies

## Success Criteria

- Successfully demonstrated all merge strategies with clear understanding of when to use each
- Completed interactive rebase operations that improve commit history
- Successfully cherry-picked commits across different branches
- Resolved complex merge conflicts while preserving all intended functionality
- Used advanced Git inspection commands to analyze repository history
- Maintained clean, professional Git history throughout all operations

## Verification Commands

```bash
# Verify merge strategies
git log --oneline --graph --all -20

# Verify rebase operations
git log --oneline -10

# Verify cherry-pick operations
git log --grep="cherry-pick" --oneline

# Verify conflict resolution
git log --merge  # Show commits involved in conflicts

# Advanced analysis
git shortlog -sn  # Contribution summary
git log --stat --since="1 week ago"  # Recent activity with stats
git reflog -10  # Recent Git operations history
```

## Integration with Next Exercise

These advanced Git skills will be essential for:
- Managing complex CI/CD pipeline triggers
- Handling hotfixes in production environments
- Coordinating multiple developer workflows
- Maintaining clean deployment history
- Managing release branching strategies

## Real-World Applications

**Enterprise Scenarios:**
- **Hotfix Deployment**: Cherry-picking critical fixes to production
- **Release Management**: Managing release branches with selective merging
- **Code Review Process**: Using rebase to clean up PR history
- **Incident Response**: Using Git bisect to find regression sources
- **Compliance**: Maintaining audit trails with proper merge commits

## Advanced Challenges

1. **Git Hooks Integration**: Create pre-commit hooks that enforce commit message standards
2. **Automated Conflict Resolution**: Script common conflict resolution patterns
3. **Branch Policy Enforcement**: Implement branch policies that require specific Git operations
4. **Git Workflow Automation**: Create scripts that automate complex Git workflows
5. **Multi-Repository Management**: Practice Git operations across multiple related repositories

## Troubleshooting Common Issues

**When Rebase Goes Wrong:**
```bash
git rebase --abort  # Cancel rebase
git reflog  # Find previous state
git reset --hard HEAD@{n}  # Recovery to previous state
```

**When Cherry-pick Conflicts:**
```bash
git cherry-pick --abort  # Cancel cherry-pick
git cherry-pick --skip  # Skip problematic commit
git cherry-pick --continue  # Continue after resolving conflicts
```

**When Merge is Messy:**
```bash
git merge --abort  # Cancel merge
git reset --merge  # Reset to pre-merge state
git clean -fd  # Clean up working directory
```

This comprehensive exercise ensures students master all advanced Git operations essential for professional DevOps environments!
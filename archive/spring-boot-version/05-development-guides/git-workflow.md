# Git Workflow Guide

This document defines the Git workflow and branching strategy for the First Viscount microservices platform. All team members must follow these guidelines to ensure smooth collaboration and code management.

## Branching Strategy

We follow a modified GitFlow workflow with the following branch types:

```
main (production)
  │
  ├── develop (integration)
  │     │
  │     ├── feature/feature-name
  │     ├── feature/another-feature
  │     │
  │     └── bugfix/bug-description
  │
  ├── release/v1.2.0
  │
  └── hotfix/critical-fix
```

### Branch Types

#### 1. Main Branch (`main`)
- **Purpose**: Production-ready code
- **Protected**: Yes
- **Direct commits**: Forbidden
- **Merged from**: `release/*` and `hotfix/*` branches only
- **Tags**: Version tags (e.g., `v1.0.0`)

#### 2. Develop Branch (`develop`)
- **Purpose**: Integration branch for features
- **Protected**: Yes
- **Direct commits**: Forbidden
- **Merged from**: `feature/*` and `bugfix/*` branches
- **Deploys to**: Development environment

#### 3. Feature Branches (`feature/*`)
- **Purpose**: New features and enhancements
- **Naming**: `feature/ticket-number-brief-description`
- **Created from**: `develop`
- **Merged to**: `develop`
- **Examples**: 
  - `feature/FV-123-product-search`
  - `feature/FV-456-payment-integration`

#### 4. Bugfix Branches (`bugfix/*`)
- **Purpose**: Non-critical bug fixes
- **Naming**: `bugfix/ticket-number-brief-description`
- **Created from**: `develop`
- **Merged to**: `develop`
- **Examples**: 
  - `bugfix/FV-789-fix-inventory-sync`
  - `bugfix/FV-234-order-validation`

#### 5. Release Branches (`release/*`)
- **Purpose**: Prepare for production release
- **Naming**: `release/vX.Y.Z`
- **Created from**: `develop`
- **Merged to**: `main` and `develop`
- **Examples**: 
  - `release/v1.2.0`
  - `release/v2.0.0-rc1`

#### 6. Hotfix Branches (`hotfix/*`)
- **Purpose**: Critical production fixes
- **Naming**: `hotfix/ticket-number-brief-description`
- **Created from**: `main`
- **Merged to**: `main` and `develop`
- **Examples**: 
  - `hotfix/FV-999-critical-payment-fix`
  - `hotfix/FV-888-security-patch`

## Workflow Steps

### 1. Starting a New Feature

```bash
# Update local develop branch
git checkout develop
git pull origin develop

# Create feature branch
git checkout -b feature/FV-123-product-search

# Push branch to remote
git push -u origin feature/FV-123-product-search
```

### 2. Daily Development Workflow

```bash
# Start your day - sync with develop
git checkout develop
git pull origin develop
git checkout feature/FV-123-product-search
git merge develop

# Make changes and commit regularly
git add .
git commit -m "feat(product): implement search algorithm"

# Push changes at end of day
git push origin feature/FV-123-product-search
```

### 3. Creating a Pull Request

Before creating a PR:

```bash
# Update feature branch with latest develop
git checkout develop
git pull origin develop
git checkout feature/FV-123-product-search
git merge develop

# Fix any conflicts
git add .
git commit -m "fix: resolve merge conflicts with develop"

# Run tests locally
mvn clean test
mvn verify -P integration-test

# Push final changes
git push origin feature/FV-123-product-search
```

Create PR via GitHub/GitLab with:
- **Title**: `FV-123: Add product search functionality`
- **Description**: Use PR template
- **Reviewers**: Assign at least 2 team members
- **Labels**: `feature`, `needs-review`

### 4. Code Review Process

#### For Reviewers:
```bash
# Check out PR locally
git fetch origin pull/123/head:pr-123
git checkout pr-123

# Run tests
mvn clean test

# Test functionality
mvn spring-boot:run

# Switch back
git checkout develop
```

#### Review Checklist:
- [ ] Code follows coding standards
- [ ] Tests pass and coverage adequate
- [ ] No merge conflicts
- [ ] Documentation updated
- [ ] Performance impact considered
- [ ] Security implications reviewed

### 5. Merging Features

After approval:

```bash
# Final sync with develop
git checkout develop
git pull origin develop
git checkout feature/FV-123-product-search
git merge develop

# If conflicts, resolve and push
git push origin feature/FV-123-product-search
```

Then merge via GitHub/GitLab UI using **Squash and Merge**.

### 6. Creating a Release

```bash
# Create release branch from develop
git checkout develop
git pull origin develop
git checkout -b release/v1.2.0

# Update version numbers
./scripts/update-version.sh 1.2.0

# Commit version changes
git add .
git commit -m "chore: bump version to 1.2.0"

# Push release branch
git push -u origin release/v1.2.0
```

During release stabilization:
```bash
# Only bug fixes allowed
git checkout -b bugfix/FV-456-release-fix release/v1.2.0
# ... fix bug ...
git commit -m "fix: correct calculation in order total"
git push origin bugfix/FV-456-release-fix
# Create PR to merge into release/v1.2.0
```

### 7. Completing a Release

```bash
# Merge to main
git checkout main
git pull origin main
git merge --no-ff release/v1.2.0
git tag -a v1.2.0 -m "Release version 1.2.0"
git push origin main --tags

# Merge back to develop
git checkout develop
git pull origin develop
git merge --no-ff release/v1.2.0
git push origin develop

# Delete release branch
git branch -d release/v1.2.0
git push origin --delete release/v1.2.0
```

### 8. Creating a Hotfix

```bash
# Create hotfix from main
git checkout main
git pull origin main
git checkout -b hotfix/FV-999-critical-fix

# Make fix
# ... implement fix ...
git commit -m "fix: prevent null pointer in payment processing"

# Update version (patch)
./scripts/update-version.sh 1.2.1

git commit -m "chore: bump version to 1.2.1"
git push -u origin hotfix/FV-999-critical-fix
```

### 9. Completing a Hotfix

```bash
# Merge to main
git checkout main
git pull origin main
git merge --no-ff hotfix/FV-999-critical-fix
git tag -a v1.2.1 -m "Hotfix version 1.2.1"
git push origin main --tags

# Merge to develop
git checkout develop
git pull origin develop
git merge --no-ff hotfix/FV-999-critical-fix
git push origin develop

# Clean up
git branch -d hotfix/FV-999-critical-fix
git push origin --delete hotfix/FV-999-critical-fix
```

## Commit Message Guidelines

### Format
```
<type>(<scope>): <subject>

<body>

<footer>
```

### Types
- **feat**: New feature
- **fix**: Bug fix
- **docs**: Documentation only
- **style**: Code style (formatting, semicolons)
- **refactor**: Code change that neither fixes a bug nor adds a feature
- **perf**: Performance improvement
- **test**: Adding missing tests
- **build**: Changes to build system or dependencies
- **ci**: Changes to CI configuration
- **chore**: Other changes that don't modify src or test files
- **revert**: Reverts a previous commit

### Scope Examples
- **product**: Product service
- **order**: Order service
- **inventory**: Inventory service
- **api**: API gateway
- **common**: Shared libraries
- **infra**: Infrastructure

### Subject Rules
- Use imperative mood ("add" not "added")
- Don't capitalize first letter
- No period at the end
- Limit to 50 characters

### Body Rules
- Wrap at 72 characters
- Explain what and why, not how
- Reference issues and PRs

### Footer Rules
- Reference issues: `Closes #123`
- Breaking changes: `BREAKING CHANGE: description`
- Co-authors: `Co-authored-by: Name <email>`

### Examples

```bash
feat(product): add advanced search functionality

Implement full-text search using PostgreSQL's tsvector
- Add search endpoint with filters
- Include pagination and sorting
- Add search indexing for better performance

Closes #FV-123

---

fix(order): prevent duplicate order creation in high concurrency

Add distributed lock using Redis to prevent race conditions
when multiple requests try to create orders simultaneously.
The lock is released automatically after 30 seconds or when
the transaction completes.

Fixes #FV-456
Co-authored-by: Jane Doe <jane@example.com>

---

refactor(inventory): extract validation logic to separate class

Move all inventory validation rules to InventoryValidator
class to improve:
- Testability
- Reusability across services
- Separation of concerns

Part of technical debt reduction initiative

---

BREAKING CHANGE: update API response format

Change error response format from:
{
  "error": "message",
  "code": 400
}

To:
{
  "error": {
    "message": "message",
    "code": "ERROR_CODE",
    "timestamp": "2024-01-01T00:00:00Z"
  }
}

All API clients need to update their error handling logic.
```

## Pull Request Guidelines

### PR Template

```markdown
## Description
Brief description of changes and why they're needed.

## Type of Change
- [ ] Bug fix (non-breaking change which fixes an issue)
- [ ] New feature (non-breaking change which adds functionality)
- [ ] Breaking change (fix or feature that would cause existing functionality to not work as expected)
- [ ] Documentation update

## Related Issues
Closes #FV-123

## Testing
- [ ] Unit tests pass
- [ ] Integration tests pass
- [ ] Manual testing completed
- [ ] Performance tests (if applicable)

## Checklist
- [ ] My code follows the project's style guidelines
- [ ] I have performed a self-review of my own code
- [ ] I have commented my code, particularly in hard-to-understand areas
- [ ] I have made corresponding changes to the documentation
- [ ] My changes generate no new warnings
- [ ] I have added tests that prove my fix is effective or that my feature works
- [ ] New and existing unit tests pass locally with my changes
- [ ] Any dependent changes have been merged and published

## Screenshots (if applicable)
Add screenshots here

## Additional Notes
Any additional information that reviewers should know
```

### PR Best Practices

1. **Keep PRs Small**: Aim for < 400 lines of code
2. **One Feature Per PR**: Don't mix features
3. **Update Often**: Sync with develop frequently
4. **Respond Quickly**: Address review comments within 24 hours
5. **Test Thoroughly**: Include unit and integration tests
6. **Document Changes**: Update README, API docs, etc.

## Git Configuration

### Required Git Configuration

```bash
# Set your identity
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# Set default branch name
git config --global init.defaultBranch main

# Enable colored output
git config --global color.ui auto

# Set default editor
git config --global core.editor "code --wait"  # VS Code
# or
git config --global core.editor "vim"

# Configure line endings
git config --global core.autocrlf input  # Mac/Linux
git config --global core.autocrlf true   # Windows

# Set up aliases
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.st status
git config --global alias.unstage 'reset HEAD --'
git config --global alias.last 'log -1 HEAD'
git config --global alias.visual '!gitk'
git config --global alias.lg "log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit"
```

### Git Hooks

Create `.git/hooks/pre-commit`:

```bash
#!/bin/bash
# pre-commit hook

echo "Running pre-commit checks..."

# Check for formatting issues
mvn spotless:check
if [ $? -ne 0 ]; then
    echo "❌ Code formatting issues found. Run 'mvn spotless:apply' to fix."
    exit 1
fi

# Run quick tests
mvn test -Dtest=*Test -DfailIfNoTests=false
if [ $? -ne 0 ]; then
    echo "❌ Tests failed. Fix failing tests before committing."
    exit 1
fi

echo "✅ Pre-commit checks passed!"
```

Create `.git/hooks/commit-msg`:

```bash
#!/bin/bash
# commit-msg hook

commit_regex='^(feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert)(\([a-z]+\))?: .{1,50}$'

if ! grep -qE "$commit_regex" "$1"; then
    echo "❌ Invalid commit message format!"
    echo "Format: <type>(<scope>): <subject>"
    echo "Example: feat(product): add search functionality"
    exit 1
fi
```

Make hooks executable:
```bash
chmod +x .git/hooks/pre-commit
chmod +x .git/hooks/commit-msg
```

## Resolving Conflicts

### Merge Conflicts

```bash
# When you encounter conflicts
git status  # See conflicted files

# Open conflicted file and look for:
<<<<<<< HEAD
Your changes
=======
Their changes
>>>>>>> branch-name

# Resolve manually, then:
git add resolved-file.java
git commit
```

### Rebase Conflicts

```bash
# Interactive rebase
git rebase -i HEAD~3

# If conflicts occur:
# 1. Resolve conflicts
git add resolved-file.java
git rebase --continue

# Or abort:
git rebase --abort
```

## Advanced Git Commands

### Useful Commands

```bash
# Stash changes
git stash save "WIP: feature implementation"
git stash list
git stash pop
git stash apply stash@{1}

# Cherry-pick specific commits
git cherry-pick abc123

# Reset changes
git reset --soft HEAD~1  # Keep changes staged
git reset --mixed HEAD~1 # Keep changes unstaged
git reset --hard HEAD~1  # Discard changes

# Clean untracked files
git clean -fd  # Remove untracked files and directories
git clean -fdn # Dry run

# Find who changed a line
git blame src/main/java/com/example/Service.java

# Search commit history
git log --grep="fix.*payment"
git log --author="John Doe"
git log --since="2 weeks ago"

# Show changes
git diff HEAD~1
git diff develop...feature/my-feature
git diff --staged

# Manage remotes
git remote -v
git remote add upstream https://github.com/original/repo.git
git fetch upstream
git merge upstream/develop
```

### Recovery Commands

```bash
# Recover deleted branch
git reflog
git checkout -b recovered-branch abc123

# Undo last commit but keep changes
git reset --soft HEAD~1

# Amend last commit
git commit --amend -m "New commit message"

# Fix commit on wrong branch
git checkout correct-branch
git cherry-pick wrong-branch
git checkout wrong-branch
git reset --hard HEAD~1
```

## CI/CD Integration

### Branch Protection Rules

Configure in GitHub/GitLab:

1. **main branch**:
   - Require pull request reviews (2 approvals)
   - Dismiss stale PR approvals
   - Require status checks (build, tests, security scan)
   - Require branches to be up to date
   - Include administrators
   - Restrict who can push

2. **develop branch**:
   - Require pull request reviews (1 approval)
   - Require status checks (build, tests)
   - Require branches to be up to date

### Automated Checks

```yaml
# .github/workflows/pr-check.yml
name: PR Checks

on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Validate PR title
        uses: amannn/action-semantic-pull-request@v5
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Check commit messages
        uses: wagoid/commitlint-github-action@v5
      
      - name: Run tests
        run: |
          mvn clean test
          mvn verify -P integration-test
```

## Best Practices

1. **Commit Often**: Small, logical commits
2. **Pull Before Push**: Always sync with remote
3. **Review Before Commit**: Self-review changes
4. **Write Meaningful Messages**: Future you will thank you
5. **Don't Commit Secrets**: Use `.gitignore` and secrets management
6. **Keep History Clean**: Use interactive rebase when needed
7. **Tag Releases**: Semantic versioning (v1.2.3)
8. **Document Complex Changes**: In commit body or PR description
9. **Test Before Push**: Run tests locally
10. **Communicate**: Let team know about breaking changes

---
*Last Updated: January 2025*
# Repository Structure and Organization

This document defines the multi-repository structure for the First Viscount microservices platform and provides guidelines for repository management.

## Repository Overview

The First Viscount platform uses a multi-repository approach for better isolation, independent deployment, and team ownership.

### Repository Layout

```
firstviscount-platform/              # GitHub Organization
├── platform-docs/                   # Documentation repository (this repo)
├── platform-libs/                   # Shared libraries repository
│   ├── common-core/
│   ├── common-messaging/
│   ├── common-security/
│   └── common-testing/
├── platform-coordination-service/   # Platform Coordination Service
├── product-catalog-service/         # Product Catalog Service
├── inventory-service/              # Inventory Management Service
├── order-service/                  # Order Management Service
├── delivery-service/               # Delivery Management Service
├── notification-service/           # Notification Service
├── platform-gateway/               # API Gateway
├── platform-infrastructure/        # Infrastructure as Code
│   ├── terraform/
│   ├── kubernetes/
│   └── docker/
└── platform-tools/                 # Development tools and scripts
```

## Service Repository Structure

Each microservice follows a standard structure:

```
service-name/
├── .github/
│   ├── workflows/
│   │   ├── build.yml
│   │   ├── test.yml
│   │   └── deploy.yml
│   └── pull_request_template.md
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/firstviscount/[service]/
│   │   │       ├── api/            # REST controllers
│   │   │       ├── config/         # Configuration classes
│   │   │       ├── domain/         # Domain models
│   │   │       ├── dto/            # Data transfer objects
│   │   │       ├── event/          # Event definitions
│   │   │       ├── exception/      # Custom exceptions
│   │   │       ├── mapper/         # Object mappers
│   │   │       ├── repository/     # Data repositories
│   │   │       ├── service/        # Business logic
│   │   │       └── util/           # Utilities
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── application-dev.yml
│   │       ├── application-test.yml
│   │       ├── application-prod.yml
│   │       └── db/migration/       # Flyway migrations
│   └── test/
│       ├── java/
│       │   └── com/firstviscount/[service]/
│       │       ├── unit/           # Unit tests
│       │       ├── integration/    # Integration tests
│       │       └── contract/       # Contract tests
│       └── resources/
│           └── test-data/
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
├── k8s/
│   ├── base/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── configmap.yaml
│   └── overlays/
│       ├── dev/
│       ├── staging/
│       └── prod/
├── docs/
│   ├── api/                        # API documentation
│   ├── architecture/               # Service-specific architecture
│   └── runbook/                    # Operational runbooks
├── scripts/
│   ├── build.sh
│   ├── test.sh
│   └── deploy.sh
├── .gitignore
├── .editorconfig
├── pom.xml                         # Maven configuration
├── README.md
├── CHANGELOG.md
└── LICENSE
```

## Shared Libraries Repository

The platform-libs repository contains shared code:

```
platform-libs/
├── common-core/
│   ├── src/main/java/
│   │   └── com/firstviscount/common/core/
│   │       ├── exception/          # Common exceptions
│   │       ├── validation/         # Validators
│   │       ├── util/              # Utilities
│   │       └── model/             # Shared models
│   └── pom.xml
├── common-messaging/
│   ├── src/main/java/
│   │   └── com/firstviscount/common/messaging/
│   │       ├── kafka/             # Kafka utilities
│   │       ├── event/             # Base event classes
│   │       └── serialization/     # Event serializers
│   └── pom.xml
├── common-security/
│   ├── src/main/java/
│   │   └── com/firstviscount/common/security/
│   │       ├── jwt/               # JWT utilities
│   │       ├── oauth/             # OAuth configuration
│   │       └── filter/            # Security filters
│   └── pom.xml
├── common-testing/
│   ├── src/main/java/
│   │   └── com/firstviscount/common/testing/
│   │       ├── containers/        # Testcontainers setup
│   │       ├── fixtures/          # Test fixtures
│   │       └── assertions/        # Custom assertions
│   └── pom.xml
├── bom/                           # Bill of Materials
│   └── pom.xml
└── parent/                        # Parent POM
    └── pom.xml
```

## Repository Naming Conventions

### Service Repositories
- Format: `[service-name]-service`
- Examples: `product-catalog-service`, `order-service`
- Always use lowercase with hyphens

### Library Repositories
- Format: `platform-[purpose]`
- Examples: `platform-libs`, `platform-tools`

### Infrastructure Repositories
- Format: `platform-infrastructure`
- Contains all IaC code

## Branch Strategy

### Main Branches
```
main                  # Production-ready code
├── develop          # Integration branch
├── release/*        # Release preparation
└── hotfix/*         # Emergency fixes
```

### Feature Branches
```
feature/JIRA-123-description     # New features
bugfix/JIRA-456-description      # Bug fixes
chore/JIRA-789-description       # Maintenance tasks
```

### Branch Protection Rules

**Main Branch**
- Require pull request reviews (2 approvers)
- Require status checks to pass
- Require branches to be up to date
- Include administrators
- Restrict force pushes

**Develop Branch**
- Require pull request reviews (1 approver)
- Require status checks to pass
- Dismiss stale reviews

## Versioning Strategy

### Semantic Versioning
All services and libraries follow semantic versioning:
```
MAJOR.MINOR.PATCH
```

### Version Management

**Services**
```xml
<version>1.2.3</version>
```

**Libraries**
```xml
<version>1.2.3</version>
<!-- In BOM -->
<firstviscount.common.version>1.2.3</firstviscount.common.version>
```

### Release Tags
```
v1.2.3              # Production releases
v1.2.3-RC1          # Release candidates
v1.2.3-beta.1       # Beta releases
```

## Repository Access Control

### Teams and Permissions

```yaml
teams:
  platform-architects:
    permission: admin
    repositories: all
  
  platform-developers:
    permission: write
    repositories: all
  
  service-owners:
    product-team:
      permission: admin
      repositories:
        - product-catalog-service
        - inventory-service
    
    order-team:
      permission: admin
      repositories:
        - order-service
        - delivery-service
    
    notification-team:
      permission: admin
      repositories:
        - notification-service
```

### CODEOWNERS File

Each repository must have a CODEOWNERS file:

```
# Default owners
* @firstviscount/platform-developers

# Service-specific owners
/src/ @firstviscount/product-team
/k8s/ @firstviscount/platform-architects
/docs/ @firstviscount/technical-writers

# Sensitive files
/src/main/resources/application-prod.yml @firstviscount/platform-architects
```

## Repository Templates

### Service Template Repository

Create `service-template` repository with:
- Standard directory structure
- Pre-configured CI/CD
- Basic Spring Boot setup
- Example tests
- Documentation templates

### Creating New Services

```bash
# Clone template
git clone https://github.com/firstviscount/service-template new-service

# Update service name
cd new-service
./scripts/rename-service.sh new-service

# Initialize repository
git init
git remote add origin https://github.com/firstviscount/new-service.git
git push -u origin main
```

## Dependency Management

### Service Dependencies

```xml
<parent>
    <groupId>com.firstviscount</groupId>
    <artifactId>platform-parent</artifactId>
    <version>1.0.0</version>
</parent>

<dependencies>
    <!-- Platform libraries -->
    <dependency>
        <groupId>com.firstviscount</groupId>
        <artifactId>common-core</artifactId>
    </dependency>
    
    <!-- External dependencies managed by BOM -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

### Library Publishing

```bash
# In platform-libs repository
mvn clean deploy -Prelease

# Version will be published to:
# - Internal Maven repository
# - GitHub Packages
```

## Repository Maintenance

### Regular Tasks

**Weekly**
- Review and merge dependabot PRs
- Update development dependencies
- Clean up stale branches

**Monthly**
- Review and update documentation
- Audit repository access
- Check for security vulnerabilities

**Quarterly**
- Major dependency updates
- Repository structure review
- Archive inactive repositories

### Repository Health Checks

```yaml
# .github/repository-health.yml
checks:
  - required_files:
      - README.md
      - LICENSE
      - CODEOWNERS
      - .gitignore
  
  - branch_protection:
      - main
      - develop
  
  - security:
      - dependency_scanning: enabled
      - secret_scanning: enabled
      - code_scanning: enabled
```

## Migration from Monorepo

If migrating from a monorepo:

### Step 1: Extract Service History
```bash
# Use git-filter-repo to extract service with history
git filter-repo --path services/product-catalog/ \
                --path-rename services/product-catalog/:
```

### Step 2: Update Import Paths
```bash
# Update package imports
find . -name "*.java" -exec sed -i \
  's/com.firstviscount.monorepo/com.firstviscount.productcatalog/g' {} \;
```

### Step 3: Update Build Configuration
- Update Maven/Gradle configurations
- Update CI/CD pipelines
- Update deployment scripts

## Best Practices

### 1. Repository Independence
- Each repository should be independently buildable
- Minimize inter-repository dependencies
- Use published artifacts, not source dependencies

### 2. Consistent Structure
- Follow the standard directory layout
- Use common build tools and scripts
- Maintain consistent naming

### 3. Documentation
- Each repository must have comprehensive README
- Include architecture diagrams
- Document build and deployment processes

### 4. Security
- Never commit secrets
- Use GitHub secrets for CI/CD
- Regular security audits

### 5. Automation
- Automate repetitive tasks
- Use GitHub Actions for CI/CD
- Implement automated testing

## Tooling

### Repository Management Tools

**GitHub CLI**
```bash
# Create repository
gh repo create firstviscount/new-service --private

# Clone all service repositories
gh repo list firstviscount --limit 100 | \
  grep -E "-service$" | \
  xargs -L1 gh repo clone
```

**Repository Scanner**
```bash
# scripts/scan-repos.sh
#!/bin/bash
# Scan all repositories for compliance
for repo in $(gh repo list firstviscount --limit 100 --json name -q '.[].name'); do
  echo "Checking $repo..."
  gh api repos/firstviscount/$repo/contents/README.md > /dev/null 2>&1 || \
    echo "  ❌ Missing README.md"
  # Add more checks...
done
```

## Repository Metrics

Track these metrics for repository health:

1. **Code Quality**
   - Test coverage > 80%
   - Code smells < 5%
   - Technical debt < 2 days

2. **Activity**
   - Regular commits
   - PR turnaround < 2 days
   - Issue resolution < 1 week

3. **Dependencies**
   - Up-to-date dependencies
   - No critical vulnerabilities
   - Regular updates

---
*Last Updated: January 2025*
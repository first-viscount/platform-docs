# ADR-003: Multi-Repository Structure

## Status

**Accepted** - January 17, 2024

## Context

With our microservices architecture decision, we need to determine how to structure our source code repositories. This decision impacts development workflow, CI/CD pipeline complexity, dependency management, and team autonomy.

### Requirements

- **Team Autonomy**: Teams should be able to work independently without blocking each other
- **Independent Deployment**: Services should be deployable independently
- **Code Sharing**: Ability to share common libraries and utilities
- **CI/CD Efficiency**: Build and deployment pipelines should be efficient
- **Version Management**: Clear versioning strategy for services and shared components
- **Code Discovery**: Easy to find and navigate between related code

### Current Architecture

- 6 microservices with clear domain boundaries
- Multiple development teams (3-4 teams, 4-6 developers each)
- Shared infrastructure components and libraries
- Platform documentation and deployment configurations

## Alternatives Considered

### 1. Monorepo (Single Repository)

**Structure Example:**
```
first-viscount-platform/
├── services/
│   ├── product-catalog/
│   ├── inventory/
│   ├── order-management/
│   ├── delivery/
│   ├── notification/
│   └── platform-coordination/
├── shared/
│   ├── libraries/
│   ├── schemas/
│   └── tools/
├── infrastructure/
├── docs/
└── deployment/
```

**Pros:**
- Single source of truth for all code
- Easy code sharing and refactoring
- Simplified dependency management
- Atomic commits across services
- Single CI/CD pipeline configuration
- Easy to enforce coding standards

**Cons:**
- Single point of failure for CI/CD
- Large repository size affects clone/checkout performance
- Tight coupling between team workflows
- Complex branch management with multiple teams
- All teams need access to entire codebase
- Difficult to enforce service boundaries

### 2. Multi-Repo (Service Per Repository)

**Structure Example:**
```
Repositories:
- first-viscount-product-catalog
- first-viscount-inventory
- first-viscount-order-management
- first-viscount-delivery
- first-viscount-notification
- first-viscount-platform-coordination
- first-viscount-shared-libraries
- first-viscount-infrastructure
- first-viscount-platform-docs
```

**Pros:**
- True service independence
- Teams have full autonomy over their codebase
- Independent CI/CD pipelines per service
- Clear ownership boundaries
- Smaller, faster repositories
- Service-specific access control

**Cons:**
- Code sharing complexity
- Dependency management overhead
- Cross-service refactoring challenges
- Multiple repositories to manage
- Potential for drift in standards
- More complex development setup

### 3. Hybrid Approach (Domain-Based Repositories)

**Structure Example:**
```
Repositories:
- first-viscount-core-services (order, inventory, catalog)
- first-viscount-fulfillment-services (delivery, notification)
- first-viscount-platform-services (coordination)
- first-viscount-shared
- first-viscount-infrastructure
```

**Pros:**
- Balance between autonomy and sharing
- Related services grouped together
- Reduced repository count
- Easier cross-service coordination within domains

**Cons:**
- Still requires coordination between teams
- Less clear ownership boundaries
- Potential for unintended coupling
- Complex CI/CD logic within repositories

## Decision

We will adopt a **Multi-Repository Structure** with one repository per microservice, plus dedicated repositories for shared components.

### Repository Structure

#### Service Repositories
1. **first-viscount-product-catalog** - Product Catalog Service
2. **first-viscount-inventory** - Inventory Management Service  
3. **first-viscount-order-management** - Order Management Service
4. **first-viscount-delivery** - Delivery Management Service
5. **first-viscount-notification** - Notification Service
6. **first-viscount-platform-coordination** - Platform Coordination Service

#### Shared Repositories
7. **first-viscount-shared-libraries** - Common libraries and utilities
8. **first-viscount-event-schemas** - Event schema definitions
9. **first-viscount-infrastructure** - Infrastructure as Code
10. **first-viscount-platform-docs** - Platform documentation
11. **first-viscount-api-gateway** - API Gateway configuration

### Repository Standards

Each service repository follows a standard structure:
```
service-name/
├── src/
│   ├── main/
│   │   ├── java/
│   │   └── resources/
│   └── test/
├── docs/
│   ├── api/
│   ├── architecture/
│   └── runbooks/
├── docker/
├── k8s/
├── scripts/
├── .github/
│   └── workflows/
├── README.md
├── CHANGELOG.md
└── API.md
```

## Rationale

### Why Multi-Repo Over Alternatives

1. **True Service Independence**: Aligns with microservices principles
   - Each service can evolve independently
   - No coordination required for simple changes
   - Clear ownership and responsibility

2. **Team Autonomy**: Supports our team structure
   - Teams have full control over their service
   - Independent release cycles
   - Technology stack flexibility

3. **CI/CD Efficiency**: 
   - Faster builds (only changed service)
   - Independent deployment pipelines
   - Reduced blast radius for failures

4. **Security and Access Control**:
   - Service-specific access permissions
   - Sensitive configurations isolated
   - Clear audit trails per service

5. **Repository Performance**:
   - Smaller repositories for faster operations
   - Focused development context
   - Reduced cognitive overhead

### Shared Component Strategy

To address the code sharing challenges:

1. **Shared Libraries Repository**:
   - Common utilities and frameworks
   - Published as Maven/NPM packages
   - Semantic versioning for compatibility

2. **Event Schemas Repository**:
   - Centralized event definitions
   - Schema evolution management
   - Generated client libraries

3. **Infrastructure Repository**:
   - Terraform/Kubernetes configurations
   - Shared deployment pipelines
   - Environment definitions

## Implementation Guidelines

### Repository Naming Convention

```
Pattern: first-viscount-{service-name}
Examples:
- first-viscount-product-catalog
- first-viscount-inventory
- first-viscount-shared-libraries
```

### Branching Strategy

Each repository uses **GitFlow** with:
- `main` - Production-ready code
- `develop` - Integration branch
- `feature/*` - Feature development
- `release/*` - Release preparation
- `hotfix/*` - Emergency fixes

### Dependency Management

1. **Shared Libraries**:
   ```xml
   <dependency>
     <groupId>com.firstviscount</groupId>
     <artifactId>shared-common</artifactId>
     <version>${shared.version}</version>
   </dependency>
   ```

2. **Event Schemas**:
   ```xml
   <dependency>
     <groupId>com.firstviscount</groupId>
     <artifactId>event-schemas</artifactId>
     <version>${schemas.version}</version>
   </dependency>
   ```

3. **Version Management**:
   - Use Bill of Materials (BOM) for version alignment
   - Automated dependency updates via Renovate/Dependabot
   - Regular compatibility testing

### CI/CD Pipeline Structure

Each service repository has:

```yaml
# .github/workflows/main.yml
name: CI/CD Pipeline
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run Tests
        run: ./gradlew test
  
  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - name: Build Docker Image
        run: docker build -t ${{ github.repository }}:${{ github.sha }} .
  
  deploy:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Production
        run: ./scripts/deploy.sh
```

### Cross-Repository Coordination

1. **API Contracts**:
   - OpenAPI specifications in each service repository
   - Contract testing to ensure compatibility
   - Breaking change notifications

2. **Documentation**:
   - Central platform documentation repository
   - Service-specific documentation in service repos
   - Automated documentation aggregation

3. **Development Environment**:
   - Docker Compose for local development
   - Shared development tools and configurations
   - Service discovery for local testing

## Consequences

### Positive

- **Team Independence**: Teams can work autonomously without blocking others
- **Service Isolation**: Clear boundaries prevent accidental coupling
- **Deployment Freedom**: Independent release cycles and rollback capabilities
- **Technology Diversity**: Teams can choose optimal tools for their domain
- **Security**: Fine-grained access control and audit capabilities
- **Performance**: Faster repository operations and focused development context

### Negative

- **Coordination Overhead**: Cross-service changes require coordination
- **Dependency Management**: Complex dependency tracking and updates
- **Development Setup**: More complex initial environment setup
- **Code Duplication**: Risk of duplicating common functionality
- **Tooling Complexity**: Multiple repositories to manage and monitor
- **Discovery**: Harder to navigate and understand the complete system

### Mitigation Strategies

1. **Coordination Overhead**:
   - Regular architecture sync meetings
   - Clear API contract testing
   - Automated cross-service compatibility checks
   - Shared communication channels (Slack, etc.)

2. **Dependency Management**:
   - Automated dependency updates
   - Bill of Materials for version alignment
   - Regular dependency audits
   - Clear upgrade strategies

3. **Development Setup**:
   - Comprehensive onboarding documentation
   - Automated setup scripts
   - Docker Compose for local development
   - IDE configurations and templates

4. **Code Duplication**:
   - Regular refactoring to shared libraries
   - Code review processes to identify duplication
   - Architecture guidelines and patterns
   - Automated code analysis tools

5. **Repository Management**:
   - Standardized repository templates
   - Automated policy enforcement
   - Central monitoring and reporting
   - Repository health metrics

## Development Workflow

### New Feature Development

1. **Single Service Feature**:
   ```bash
   # Clone service repository
   git clone git@github.com:first-viscount/first-viscount-order-management.git
   cd first-viscount-order-management
   
   # Create feature branch
   git checkout -b feature/payment-improvements
   
   # Develop, test, and commit
   # Create pull request
   # Deploy independently
   ```

2. **Cross-Service Feature**:
   ```bash
   # Coordinate via GitHub issues/project boards
   # Implement in dependent service first
   # Update contracts and schemas
   # Implement in consuming services
   # Deploy in correct order
   ```

### Shared Library Updates

```bash
# Update shared library
cd first-viscount-shared-libraries
# Make changes and release new version

# Update consuming services
cd first-viscount-order-management
# Update dependency version
# Test compatibility
# Deploy when ready
```

## Monitoring and Governance

### Repository Health Metrics

- Build success rate per repository
- Test coverage per service
- Dependency freshness
- Security vulnerability count
- Documentation completeness

### Governance Policies

- Required code reviews for all changes
- Automated security scanning
- Dependency vulnerability scanning
- License compliance checking
- API breaking change detection

### Cross-Repository Tooling

- **GitHub Organizations**: Centralized management
- **Repository Templates**: Consistent structure
- **GitHub Apps**: Automated policy enforcement
- **Dependabot**: Automated dependency updates
- **CodeQL**: Security scanning across all repos

## Related Decisions

- [ADR-001: Microservices Architecture](./adr-001-microservices.md)
- [ADR-004: Testing Strategy](./adr-004-testing-strategy.md)

## Migration Plan

### Phase 1: Repository Creation (Week 1)
- Create all service repositories
- Set up repository templates
- Configure CI/CD pipelines

### Phase 2: Code Migration (Weeks 2-3)
- Extract services from existing codebase
- Migrate shared libraries
- Update dependency configurations

### Phase 3: Team Transition (Weeks 4-5)
- Team training on new workflows
- Documentation updates
- Process refinement

### Phase 4: Optimization (Ongoing)
- Monitor and improve workflows
- Automate common tasks
- Refine shared libraries

## References

- [Microservices and the Multi-Repo vs Mono-Repo Debate](https://blog.shippable.com/our-journey-to-microservices-and-a-mono-repository)
- [Git Repository Structure for Microservices](https://docs.microsoft.com/en-us/azure/devops/repos/git/repository-structure)
- [Monorepo vs Multi-Repo: Pros and Cons](https://kinsta.com/blog/monorepo-vs-multi-repo/)
- [GitHub Best Practices for Organizations](https://docs.github.com/en/organizations)
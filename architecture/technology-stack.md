# Technology Stack

**Last Updated**: July 2025

This document defines the current technology stack and version requirements for the First Viscount platform. For architectural decisions and rationale, see [decisions.md](../decisions.md).

## Core Technologies

### Programming Language
- **Python**: 3.13+ (latest stable as of July 2025)
  - Released: October 2024
  - Key features: Performance improvements, better error messages
  - End of support: October 2029

### Web Framework
- **FastAPI**: 0.115.13
  - Latest stable release
  - Async support, automatic OpenAPI docs
  - Compatible with Python 3.13

### Database
- **PostgreSQL**: 17.5
  - Latest stable release
  - Improved query performance, better JSON support
  - Compatible with all major ORMs

### Message Broker
- **Redpanda**: 25.1
  - Kafka-compatible, no JVM required
  - Better resource efficiency than Kafka
  - Native WebAssembly transforms support

### Service Mesh / API Gateway
- **Traefik**: 3.3.0
  - Latest stable release
  - Native Kubernetes integration
  - Built-in middleware support

### Container & Orchestration
- **Docker**: 27.5.0
  - Latest stable release
  - Improved build performance
- **Docker Compose**: 2.32.0
  - Latest stable release
  - Better secret management
- **K3s**: 1.32.0 (future migration)
  - Lightweight Kubernetes
  - Production-ready

## Development Tools

### Code Quality
- **ruff**: 0.9.0
  - Extremely fast Python linter
  - Replaces flake8, isort, and more
- **mypy**: 1.14.0
  - Static type checker
  - Full Python 3.13 support
- **black**: 24.12.0
  - Code formatter
  - Python 3.13 compatible

### Testing
- **pytest**: 8.3.5
  - Testing framework
  - Excellent async support
- **pytest-asyncio**: 0.25.0
  - Async test support
- **pytest-cov**: 6.0.0
  - Coverage reporting
- **httpx**: 0.28.1
  - Async HTTP client for testing

### Security
- **bandit**: 1.8.0
  - Security linter
- **safety**: 3.3.0
  - Dependency vulnerability scanner
- **pip-audit**: 2.8.0
  - Python package vulnerability scanner

### Documentation
- **mkdocs**: 1.6.2
  - Documentation generator
- **mkdocs-material**: 9.6.0
  - Material theme for mkdocs

## Python Dependencies

### Core Libraries
- **pydantic**: 2.10.5
  - Data validation using Python type annotations
  - FastAPI dependency
- **sqlalchemy**: 2.0.38
  - SQL toolkit and ORM
  - Async support included
- **alembic**: 1.15.0
  - Database migration tool
- **asyncpg**: 0.31.0
  - PostgreSQL async driver

### API & Web
- **uvicorn**: 0.34.0
  - ASGI server
  - Production-ready with proper configuration
- **python-jose**: 3.3.0
  - JWT implementation
- **passlib**: 1.7.4
  - Password hashing
- **python-multipart**: 0.0.20
  - Form data parsing
- **httpx**: 0.28.1
  - Async HTTP client

### Messaging
- **aiokafka**: 0.12.0
  - Async Kafka client
  - Redpanda compatible
- **confluent-kafka**: 2.7.0
  - Alternative Kafka client
  - Better performance for some use cases

### Observability
- **opentelemetry-api**: 1.29.0
- **opentelemetry-sdk**: 1.29.0
- **opentelemetry-instrumentation-fastapi**: 0.51b0
- **prometheus-client**: 0.22.0

### Utilities
- **python-dotenv**: 1.0.1
  - Environment variable management
- **structlog**: 24.6.0
  - Structured logging
- **tenacity**: 9.0.0
  - Retry library
- **redis**: 5.2.1
  - Redis client (for caching/rate limiting)

## Version Compatibility Matrix

| Component | Minimum Version | Tested With | Notes |
|-----------|----------------|-------------|-------|
| Python | 3.13.0 | 3.13.2 | No 3.14 until stable |
| PostgreSQL | 17.0 | 17.5 | Schema compatible |
| Redpanda | 25.0 | 25.1 | Kafka protocol 3.8 |
| Docker | 26.0 | 27.5.0 | BuildKit required |
| Node.js (for tools) | 22.0 | 22.12.0 | LTS version |

## Update Policy

1. **Security Updates**: Applied immediately
2. **Patch Updates**: Applied weekly during maintenance
3. **Minor Updates**: Evaluated monthly, applied if stable
4. **Major Updates**: Evaluated quarterly with full testing

## Version Pinning Strategy

### Production
- Exact versions in requirements.txt
- Lock file committed to repository
- Automated dependency updates via Dependabot

### Development
- Minimum versions in pyproject.toml
- Allow patch updates automatically
- Regular dependency audits

## Migration Notes

### From Python 3.11 to 3.13
- Update all type hints to use new syntax
- Remove typing_extensions where possible
- Update CI/CD to use Python 3.13 image
- Test all async code thoroughly

### From Docker Compose to K3s
- Planned for after first service is stable
- Will maintain both for transition period
- Separate migration guide to be created

## References

- Python 3.13 Release Notes: https://docs.python.org/3/whatsnew/3.13.html
- FastAPI Releases: https://github.com/tiangolo/fastapi/releases
- PostgreSQL 17 Features: https://www.postgresql.org/about/news/postgresql-17-released-2936/
- Redpanda Documentation: https://docs.redpanda.com/
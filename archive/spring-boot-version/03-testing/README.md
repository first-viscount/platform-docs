# Testing Documentation

## Overview

This directory contains comprehensive testing strategies and guidelines for the First Viscount e-commerce platform. Our testing approach covers all aspects of microservices testing, from unit tests to production-like chaos engineering.

## Testing Philosophy

1. **Test Early, Test Often**: Catch issues before they reach production
2. **Automate Everything**: Manual testing should be the exception
3. **Test in Production-Like Environments**: Use k3s for realistic testing
4. **Data-Driven Testing**: Use realistic test data and scenarios
5. **Performance is a Feature**: Load test regularly

## Documentation Structure

### 1. [Testing Without Frontend](./testing-without-frontend.md)
Complete guide for backend API testing without UI dependencies:
- REST API testing with REST Assured and Postman
- MockMvc for Spring Boot integration
- Contract testing strategies
- Event-driven testing approaches
- GraphQL testing (if applicable)
- Automated test execution

**Use this when**: You need to validate backend functionality independently of frontend development.

### 2. [Integration Testing](./integration-testing.md)
Comprehensive integration testing strategies:
- Service integration tests (database, Kafka, Redis)
- Inter-service communication testing
- End-to-end workflow validation
- Saga pattern testing
- TestContainers setup
- Contract testing with Spring Cloud Contract

**Use this when**: You need to verify that services work correctly together.

### 3. [Load Testing](./load-testing.md)
Performance testing with K6 and JMeter:
- K6 test scenarios (spike, stress, soak)
- JMeter test plans
- Custom metrics and monitoring
- CI/CD integration
- Performance baselines
- Bottleneck identification

**Use this when**: You need to validate system performance and capacity.

### 4. [Test Data Management](./test-data-management.md)
Strategies for managing test data:
- Test data factories
- Fixture management
- Database seeding
- Test data builders
- Data isolation strategies
- Edge case data generation

**Use this when**: You need consistent, realistic test data across all testing scenarios.

### 5. [Kubernetes Testing Strategy](./kubernetes-testing-strategy.md)
Testing in k3s/Kubernetes environments:
- k3s test environment setup
- Pod-to-pod testing
- Service mesh testing
- Chaos engineering
- ConfigMap/Secret testing
- Persistent volume testing

**Use this when**: You need to test the platform in a Kubernetes environment.

## Quick Start

### Setting Up Test Environment

1. **Local Testing (Docker Compose)**:
```bash
cd platform
docker-compose -f docker-compose.test.yml up -d
mvn test
```

2. **k3s Testing**:
```bash
# Setup k3s
curl -sfL https://get.k3s.io | sh -

# Deploy test infrastructure
kubectl apply -f k8s/test/infrastructure/

# Run tests
mvn test -Dspring.profiles.active=k8s-test
```

3. **Load Testing**:
```bash
# Install k6
brew install k6  # or apt-get install k6

# Run load test
k6 run tests/k6/basic-load-test.js
```

## Testing Pyramid

```
         /\
        /  \  E2E Tests (10%)
       /    \ - Full workflow tests
      /      \ - Chaos engineering
     /--------\
    /          \ Integration Tests (30%)
   /            \ - Service integration
  /              \ - Contract tests
 /                \ - API tests
/------------------\
     Unit Tests (60%)
  - Business logic
  - Data validation
  - Error handling
```

## Test Execution Strategy

### Local Development
1. Unit tests on save (IDE)
2. Integration tests before commit
3. API tests for feature validation

### CI Pipeline
1. Unit tests (all services)
2. Integration tests (TestContainers)
3. Contract tests
4. Load tests (on schedule)

### Pre-Production
1. Full k8s deployment
2. End-to-end tests
3. Chaos engineering
4. Performance validation

## Key Testing Patterns

### 1. Given-When-Then
```java
@Test
void orderCreation_WithValidData_ShouldSucceed() {
    // Given
    Customer customer = testDataFactory.createCustomer();
    Product product = testDataFactory.createProduct();
    
    // When
    Order order = orderService.createOrder(customer, product);
    
    // Then
    assertThat(order.getStatus()).isEqualTo(OrderStatus.PENDING);
    verify(inventoryService).reserveStock(product.getId(), 1);
}
```

### 2. Test Data Builders
```java
Order order = TestDataBuilder.order()
    .forCustomer(customer)
    .withItem(product, 2)
    .withShipping(address)
    .build();
```

### 3. Async Testing
```java
await().atMost(Duration.ofSeconds(5))
    .untilAsserted(() -> {
        Order updated = orderRepository.findById(orderId);
        assertThat(updated.getStatus()).isEqualTo(OrderStatus.PROCESSED);
    });
```

## Testing Standards

### Naming Conventions
- Test classes: `*Test` for unit tests, `*IntegrationTest` for integration
- Test methods: `methodName_scenario_expectedBehavior()`
- Test data: Prefix with `test` (e.g., `testCustomer`, `testOrder`)

### Assertions
- Use AssertJ for fluent assertions
- One logical assertion per test
- Descriptive failure messages

### Test Independence
- No shared state between tests
- Clean up after each test
- Use `@DirtiesContext` sparingly

## Monitoring Test Health

### Metrics to Track
1. **Test Coverage**: Aim for >80% for critical paths
2. **Test Execution Time**: Keep under 10 minutes for CI
3. **Flaky Test Rate**: Should be <1%
4. **Test Failure Rate**: Track trends over time

### Test Reports
- JUnit XML reports for CI integration
- HTML reports for human consumption
- Coverage reports with JaCoCo
- Performance test reports with graphs

## Common Testing Scenarios

### 1. New Feature Development
1. Write unit tests for business logic
2. Add integration tests for service boundaries
3. Create API tests for endpoints
4. Add test data factories
5. Include in load test scenarios

### 2. Bug Fixes
1. Write failing test that reproduces bug
2. Fix the bug
3. Verify test passes
4. Add regression test

### 3. Performance Issues
1. Create load test to reproduce issue
2. Profile and identify bottleneck
3. Fix and verify with load test
4. Add performance regression test

## Troubleshooting

### Flaky Tests
- Check for timing issues (use proper waits)
- Verify test data isolation
- Look for external dependencies
- Check for race conditions

### Slow Tests
- Use test slicing (`@WebMvcTest`, `@DataJpaTest`)
- Mock expensive operations
- Parallelize test execution
- Use in-memory databases

### Environment Issues
- Verify Docker/k3s is running
- Check port availability
- Ensure test data is seeded
- Review environment variables

## Tools and Libraries

### Testing Frameworks
- **JUnit 5**: Core testing framework
- **Mockito**: Mocking framework
- **AssertJ**: Fluent assertions
- **REST Assured**: API testing
- **TestContainers**: Integration testing

### Performance Testing
- **K6**: Modern load testing
- **JMeter**: Traditional load testing
- **Gatling**: Scala-based load testing

### Utilities
- **Faker**: Test data generation
- **Awaitility**: Async testing
- **WireMock**: API mocking
- **Swagger/OpenAPI**: API documentation/testing

## Next Steps

1. Start with [Testing Without Frontend](./testing-without-frontend.md) for API testing basics
2. Progress to [Integration Testing](./integration-testing.md) for service integration
3. Add [Load Testing](./load-testing.md) to your CI pipeline
4. Implement proper [Test Data Management](./test-data-management.md)
5. Graduate to [Kubernetes Testing](./kubernetes-testing-strategy.md) for production-like testing

## Contributing

When adding new test strategies:
1. Document the approach clearly
2. Provide working examples
3. Include troubleshooting tips
4. Update this README
5. Share learnings with the team

---

*"Quality is not an act, it is a habit." - Aristotle*
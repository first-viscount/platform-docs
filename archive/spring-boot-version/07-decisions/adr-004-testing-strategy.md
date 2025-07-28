# ADR-004: Comprehensive Testing Strategy

## Status

**Accepted** - January 18, 2024

## Context

With our microservices architecture and multi-repository structure, we need a comprehensive testing strategy that ensures system reliability while maintaining development velocity. The strategy must address testing at multiple levels: unit, integration, contract, and end-to-end.

### Requirements

- **Quality Assurance**: Maintain high code quality and system reliability
- **Development Velocity**: Tests should not significantly slow down development
- **Confidence**: High confidence in deployments and releases
- **Cost Efficiency**: Optimal balance between test coverage and execution cost
- **Maintainability**: Tests should be easy to maintain and update
- **Feedback Speed**: Fast feedback for developers during development

### Challenges

- **Distributed System Testing**: Testing interactions between multiple services
- **Data Consistency**: Managing test data across multiple databases
- **Environment Management**: Coordinating test environments for multiple services
- **Test Isolation**: Ensuring tests don't interfere with each other
- **Contract Evolution**: Managing API contract changes between services

## Alternatives Considered

### 1. Traditional Testing Pyramid

**Structure**: Heavy unit testing, moderate integration testing, minimal E2E testing

**Pros:**
- Fast feedback from unit tests
- Low cost for most tests
- Easy to maintain unit tests

**Cons:**
- Limited confidence in system integration
- Gap between unit tests and real system behavior
- Insufficient for microservices architecture

### 2. Testing Trophy/Diamond

**Structure**: Heavy integration testing, moderate unit and E2E testing

**Pros:**
- Higher confidence in actual system behavior
- Better balance for microservices
- Tests closer to production scenarios

**Cons:**
- Slower feedback than pure unit testing
- More complex test setup
- Higher maintenance overhead

### 3. Contract-First Testing

**Structure**: Heavy contract testing, moderate unit testing, minimal integration

**Pros:**
- Excellent for microservices
- Independent service testing
- Clear API boundaries

**Cons:**
- Requires sophisticated tooling
- Complex contract management
- May miss integration issues

## Decision

We will adopt a **Hybrid Testing Strategy** that combines elements from multiple approaches, optimized for our microservices architecture.

### Testing Levels

#### 1. Unit Tests (Foundation)
- **Scope**: Individual classes and methods
- **Coverage Target**: 80%+ for business logic
- **Tools**: JUnit 5, Mockito, AssertJ
- **Execution**: Every commit, sub-second feedback

#### 2. Component Tests (Service Level)
- **Scope**: Single service with real database, mocked external dependencies
- **Coverage Target**: 70%+ of service functionality
- **Tools**: Spring Boot Test, TestContainers
- **Execution**: Every pull request

#### 3. Contract Tests (API Boundaries)
- **Scope**: API contracts between services
- **Coverage Target**: All public APIs
- **Tools**: Spring Cloud Contract, Pact
- **Execution**: Contract changes and releases

#### 4. Integration Tests (Cross-Service)
- **Scope**: Multiple services with real dependencies
- **Coverage Target**: Critical business flows
- **Tools**: TestContainers, Docker Compose
- **Execution**: Before deployment

#### 5. End-to-End Tests (User Journeys)
- **Scope**: Complete user workflows
- **Coverage Target**: Happy paths and critical error scenarios
- **Tools**: Selenium, Playwright, REST Assured
- **Execution**: Deployment pipeline

## Implementation Strategy

### Unit Testing Standards

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    
    @Mock
    private InventoryService inventoryService;
    
    @Mock
    private PaymentService paymentService;
    
    @InjectMocks
    private OrderService orderService;
    
    @Test
    @DisplayName("Should create order when inventory available and payment successful")
    void shouldCreateOrderSuccessfully() {
        // Given
        CreateOrderRequest request = OrderTestDataBuilder.validOrder()
            .withCustomerId(CUSTOMER_ID)
            .withItems(List.of(
                OrderItemTestDataBuilder.validItem()
                    .withProductId(PRODUCT_ID)
                    .withQuantity(2)
                    .build()
            ))
            .build();
            
        when(inventoryService.checkAvailability(any()))
            .thenReturn(InventoryCheckResult.available());
        when(paymentService.authorizePayment(any()))
            .thenReturn(PaymentResult.success("auth_123"));
        
        // When
        Order order = orderService.createOrder(request);
        
        // Then
        assertThat(order.getStatus()).isEqualTo(OrderStatus.PENDING_FULFILLMENT);
        assertThat(order.getItems()).hasSize(1);
        assertThat(order.getTotalAmount()).isEqualTo(new BigDecimal("1199.98"));
        
        verify(inventoryService).reserveInventory(any());
        verify(paymentService).authorizePayment(any());
    }
    
    @Test
    @DisplayName("Should throw exception when inventory insufficient")
    void shouldFailWhenInventoryInsufficient() {
        // Given
        CreateOrderRequest request = OrderTestDataBuilder.validOrder().build();
        when(inventoryService.checkAvailability(any()))
            .thenReturn(InventoryCheckResult.insufficient());
        
        // When/Then
        assertThatThrownBy(() -> orderService.createOrder(request))
            .isInstanceOf(InsufficientInventoryException.class)
            .hasMessage("Insufficient inventory for product: " + PRODUCT_ID);
            
        verify(paymentService, never()).authorizePayment(any());
    }
}
```

### Component Testing with TestContainers

```java
@SpringBootTest
@Testcontainers
class OrderServiceComponentTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
            .withDatabaseName("order_test")
            .withUsername("test")
            .withPassword("test");
            
    @Container
    static KafkaContainer kafka = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.5.0"));
    
    @Autowired
    private OrderService orderService;
    
    @Autowired
    private OrderRepository orderRepository;
    
    @MockBean
    private InventoryService inventoryService;
    
    @MockBean
    private PaymentService paymentService;
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }
    
    @Test
    void shouldPersistOrderSuccessfully() {
        // Given
        when(inventoryService.checkAvailability(any()))
            .thenReturn(InventoryCheckResult.available());
        when(paymentService.authorizePayment(any()))
            .thenReturn(PaymentResult.success("auth_123"));
            
        CreateOrderRequest request = OrderTestDataBuilder.validOrder().build();
        
        // When
        Order order = orderService.createOrder(request);
        
        // Then
        Order savedOrder = orderRepository.findById(order.getId()).orElseThrow();
        assertThat(savedOrder.getStatus()).isEqualTo(OrderStatus.PENDING_FULFILLMENT);
        assertThat(savedOrder.getCustomerId()).isEqualTo(request.getCustomerId());
    }
}
```

### Contract Testing with Spring Cloud Contract

```groovy
// order-service/src/test/resources/contracts/shouldCreateOrderSuccessfully.groovy
Contract.make {
    description "Should create order when valid request provided"
    
    request {
        method POST()
        url "/api/v1/orders"
        headers {
            contentType(applicationJson())
            header("Authorization", "Bearer valid_token")
        }
        body([
            customerId: "123e4567-e89b-12d3-a456-426614174000",
            items: [[
                productId: "550e8400-e29b-41d4-a716-446655440000",
                quantity: 2,
                unitPrice: 599.99
            ]]
        ])
    }
    
    response {
        status OK()
        headers {
            contentType(applicationJson())
        }
        body([
            orderId: anyUuid(),
            status: "PENDING_PAYMENT",
            totalAmount: 1199.98
        ])
    }
}
```

### Integration Testing Setup

```java
@SpringBootTest
@TestPropertySource(properties = {
    "spring.profiles.active=integration-test"
})
class OrderFulfillmentIntegrationTest {
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Autowired
    private WireMockServer inventoryServiceMock;
    
    @Autowired
    private WireMockServer paymentServiceMock;
    
    @Test
    void shouldCompleteOrderFulfillmentFlow() {
        // Given - Mock external services
        inventoryServiceMock.stubFor(post(urlEqualTo("/api/v1/inventory/reserve"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("""
                    {
                        "reservationId": "res-123",
                        "status": "RESERVED"
                    }
                    """)));
                    
        paymentServiceMock.stubFor(post(urlEqualTo("/api/v1/payments/authorize"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("""
                    {
                        "paymentId": "pay-456",
                        "status": "AUTHORIZED"
                    }
                    """)));
        
        // When - Create order
        CreateOrderRequest request = OrderTestDataBuilder.validOrder().build();
        ResponseEntity<OrderResponse> response = restTemplate.postForEntity(
            "/api/v1/orders", 
            request, 
            OrderResponse.class
        );
        
        // Then - Verify order created
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody().getStatus()).isEqualTo("PENDING_FULFILLMENT");
        
        // And - Verify external service calls
        inventoryServiceMock.verify(postRequestedFor(urlEqualTo("/api/v1/inventory/reserve")));
        paymentServiceMock.verify(postRequestedFor(urlEqualTo("/api/v1/payments/authorize")));
    }
}
```

### End-to-End Testing Framework

```java
@ExtendWith(SeleniumExtension.class)
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class OrderE2ETest {
    
    private WebDriver driver;
    private OrderPage orderPage;
    private PaymentPage paymentPage;
    
    @BeforeAll
    void setUp() {
        driver = new ChromeDriver();
        orderPage = new OrderPage(driver);
        paymentPage = new PaymentPage(driver);
    }
    
    @Test
    @DisplayName("Complete order flow from product selection to confirmation")
    void shouldCompleteOrderFlow() {
        // Given - User has items in cart
        CartTestData.withItems(
            ProductTestData.laptop().withQuantity(1),
            ProductTestData.mouse().withQuantity(2)
        ).setupInSession(driver);
        
        // When - User proceeds to checkout
        orderPage.navigateToCheckout()
            .fillShippingAddress(AddressTestData.validAddress())
            .selectShippingMethod("STANDARD")
            .proceedToPayment();
            
        paymentPage.fillPaymentDetails(PaymentTestData.validCreditCard())
            .submitOrder();
        
        // Then - Order confirmation displayed
        OrderConfirmationPage confirmationPage = new OrderConfirmationPage(driver);
        assertThat(confirmationPage.isDisplayed()).isTrue();
        assertThat(confirmationPage.getOrderNumber()).matches("ORD-\\d{4}-\\d{6}");
        assertThat(confirmationPage.getTotalAmount()).isEqualTo("$1,499.97");
        
        // And - Confirmation email sent
        EmailTestHelper.waitForEmail()
            .to(CustomerTestData.defaultCustomer().getEmail())
            .withSubject(containsString("Order Confirmed"))
            .assertReceived();
    }
}
```

## Test Data Management

### Test Data Builders

```java
public class OrderTestDataBuilder {
    private UUID customerId = UUID.randomUUID();
    private List<OrderItem> items = new ArrayList<>();
    private Address shippingAddress = AddressTestData.validAddress();
    private PaymentMethod paymentMethod = PaymentTestData.validCreditCard();
    
    public static OrderTestDataBuilder validOrder() {
        return new OrderTestDataBuilder()
            .withItems(List.of(OrderItemTestData.validItem()));
    }
    
    public OrderTestDataBuilder withCustomerId(UUID customerId) {
        this.customerId = customerId;
        return this;
    }
    
    public OrderTestDataBuilder withItems(List<OrderItem> items) {
        this.items = items;
        return this;
    }
    
    public CreateOrderRequest build() {
        return CreateOrderRequest.builder()
            .customerId(customerId)
            .items(items)
            .shippingAddress(shippingAddress)
            .paymentMethod(paymentMethod)
            .build();
    }
}
```

### Database Test Fixtures

```java
@Component
public class TestDataFixtures {
    
    @Autowired
    private CustomerRepository customerRepository;
    
    @Autowired
    private ProductRepository productRepository;
    
    @PostConstruct
    @Transactional
    public void loadTestData() {
        // Only load in test profiles
        if (Arrays.asList(environment.getActiveProfiles()).contains("test")) {
            loadCustomers();
            loadProducts();
        }
    }
    
    private void loadCustomers() {
        customerRepository.saveAll(List.of(
            Customer.builder()
                .id(UUID.fromString("123e4567-e89b-12d3-a456-426614174000"))
                .email("john.doe@example.com")
                .name("John Doe")
                .build(),
            Customer.builder()
                .id(UUID.fromString("456e7890-e89b-12d3-a456-426614174000"))
                .email("jane.smith@example.com")
                .name("Jane Smith")
                .build()
        ));
    }
}
```

## Test Execution Strategy

### Local Development

```bash
# Run unit tests (fast feedback)
./gradlew test

# Run component tests
./gradlew componentTest

# Run contract tests
./gradlew contractTest

# Run all tests
./gradlew check
```

### CI/CD Pipeline

```yaml
# .github/workflows/test.yml
name: Test Pipeline

on: [push, pull_request]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-java@v3
        with:
          java-version: '17'
      - name: Run Unit Tests
        run: ./gradlew test
      - name: Publish Test Results
        uses: dorny/test-reporter@v1
        if: always()
        with:
          name: Unit Tests
          path: build/test-results/test/*.xml
          reporter: java-junit

  component-tests:
    runs-on: ubuntu-latest
    needs: unit-tests
    steps:
      - uses: actions/checkout@v3
      - name: Run Component Tests
        run: ./gradlew componentTest
        
  contract-tests:
    runs-on: ubuntu-latest
    needs: unit-tests
    steps:
      - uses: actions/checkout@v3
      - name: Run Contract Tests
        run: ./gradlew contractTest
      - name: Publish Contracts
        run: ./gradlew publishContracts

  integration-tests:
    runs-on: ubuntu-latest
    needs: [component-tests, contract-tests]
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v3
      - name: Start Test Environment
        run: docker-compose -f docker-compose.test.yml up -d
      - name: Run Integration Tests
        run: ./gradlew integrationTest
      - name: Stop Test Environment
        run: docker-compose -f docker-compose.test.yml down

  e2e-tests:
    runs-on: ubuntu-latest
    needs: integration-tests
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v3
      - name: Deploy to Staging
        run: ./scripts/deploy-staging.sh
      - name: Run E2E Tests
        run: ./gradlew e2eTest
```

## Quality Gates

### Code Coverage Requirements

- **Unit Tests**: Minimum 80% line coverage for service layer
- **Component Tests**: Minimum 70% API endpoint coverage
- **Integration Tests**: 100% critical business flow coverage
- **Overall**: Minimum 75% combined coverage

### Performance Criteria

- **Unit Tests**: Complete within 30 seconds
- **Component Tests**: Complete within 5 minutes
- **Integration Tests**: Complete within 15 minutes
- **E2E Tests**: Complete within 30 minutes

### Quality Metrics

```java
// Gradle build configuration
jacocoTestReport {
    reports {
        xml.enabled true
        html.enabled true
    }
    
    afterEvaluate {
        classDirectories.setFrom(files(classDirectories.files.collect {
            fileTree(dir: it, exclude: [
                '**/*Application*',
                '**/config/**',
                '**/dto/**',
                '**/entity/**'
            ])
        }))
    }
}

jacocoTestCoverageVerification {
    violationRules {
        rule {
            limit {
                minimum = 0.80
            }
        }
        rule {
            element = 'CLASS'
            excludes = ['*.config.*', '*.dto.*', '*.entity.*']
            limit {
                counter = 'LINE'
                value = 'COVEREDRATIO'
                minimum = 0.75
            }
        }
    }
}
```

## Consequences

### Positive

- **Comprehensive Coverage**: Multiple testing levels provide high confidence
- **Fast Feedback**: Unit tests provide immediate feedback to developers
- **Service Isolation**: Component tests validate service behavior independently
- **Contract Safety**: Contract tests prevent breaking API changes
- **System Validation**: Integration and E2E tests validate complete workflows
- **Maintainability**: Clear test categories and standards improve maintainability

### Negative

- **Complexity**: Multiple testing approaches increase learning curve
- **Execution Time**: Comprehensive testing takes significant time
- **Maintenance Overhead**: More tests require more maintenance effort
- **Infrastructure Requirements**: TestContainers and test environments require resources
- **Coordination**: Contract testing requires coordination between teams

### Mitigation Strategies

1. **Complexity Management**:
   - Comprehensive documentation and examples
   - Team training and knowledge sharing
   - Standardized testing utilities and helpers
   - Clear guidelines for when to use each testing approach

2. **Execution Time Optimization**:
   - Parallel test execution
   - Test categorization and selective execution
   - Caching of test dependencies
   - Optimized test environments

3. **Maintenance Reduction**:
   - Test data builders for consistent test setup
   - Shared testing utilities across services
   - Automated test maintenance tools
   - Regular refactoring of test code

## Testing Anti-Patterns to Avoid

### 1. Ice Cream Cone (Inverted Pyramid)
- **Problem**: Too many E2E tests, few unit tests
- **Solution**: Follow pyramid with solid unit test foundation

### 2. Test-Driven Damage
- **Problem**: Tests that don't add value but slow development
- **Solution**: Focus on behavior testing, not implementation details

### 3. Fragile Tests
- **Problem**: Tests that break with unrelated changes
- **Solution**: Test behavior, not implementation; use stable selectors

### 4. Slow Test Feedback
- **Problem**: Waiting too long for test results
- **Solution**: Fast unit tests, parallel execution, test categorization

## Related Decisions

- [ADR-001: Microservices Architecture](./adr-001-microservices.md)
- [ADR-003: Multi-Repository Structure](./adr-003-repository-structure.md)

## References

- [The Testing Pyramid](https://martinfowler.com/articles/practical-test-pyramid.html)
- [Testing Microservices](https://martinfowler.com/articles/microservice-testing/)
- [Spring Boot Testing](https://spring.io/guides/gs/testing-web/)
- [TestContainers Documentation](https://www.testcontainers.org/)
- [Contract Testing with Pact](https://docs.pact.io/)
- [Testing Strategies in a Microservice Architecture](https://martinfowler.com/articles/microservice-testing/)
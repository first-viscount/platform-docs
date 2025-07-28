# Integration Testing

This document provides comprehensive strategies for integration testing in the First Viscount microservices platform, focusing on service-to-service communication, event-driven architecture, and distributed transactions.

## Overview

Integration testing validates:
- Service interactions and contracts
- Database integrations
- Message broker communication
- External API integrations
- End-to-end workflows
- Distributed transaction consistency

## Testing Infrastructure

### TestContainers Setup

```java
@SpringBootTest
@Testcontainers
@ActiveProfiles("integration-test")
public abstract class IntegrationTestBase {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");
    
    @Container
    static KafkaContainer kafka = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.4.0"))
        .withKraft();
    
    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);
    
    @Container
    static MongoDBContainer mongodb = new MongoDBContainer("mongo:6");
    
    @DynamicPropertySource
    static void properties(DynamicPropertyRegistry registry) {
        // PostgreSQL
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        
        // Kafka
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
        
        // Redis
        registry.add("spring.redis.host", redis::getHost);
        registry.add("spring.redis.port", () -> redis.getMappedPort(6379));
        
        // MongoDB
        registry.add("spring.data.mongodb.uri", mongodb::getReplicaSetUrl);
    }
    
    @BeforeAll
    static void setup() {
        // Start containers in parallel for faster startup
        Startables.deepStart(postgres, kafka, redis, mongodb).join();
    }
}
```

### Docker Compose Alternative

```yaml
# docker-compose.integration-test.yml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: testdb
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U test"]
      interval: 10s
      timeout: 5s
      retries: 5

  kafka:
    image: confluentinc/cp-kafka:7.4.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    healthcheck:
      test: ["CMD", "kafka-topics", "--bootstrap-server", "localhost:9092", "--list"]
      interval: 10s
      timeout: 10s
      retries: 5

  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  mongodb:
    image: mongo:6
    ports:
      - "27017:27017"
    healthcheck:
      test: echo 'db.runCommand("ping").ok' | mongosh localhost:27017/test --quiet
      interval: 10s
      timeout: 5s
      retries: 5
```

## Service Integration Testing

### 1. Database Integration Tests

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Import(TestConfig.class)
@Sql(scripts = "/sql/test-data.sql", executionPhase = Sql.ExecutionPhase.BEFORE_TEST_METHOD)
@Sql(scripts = "/sql/cleanup.sql", executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)
public class ProductRepositoryIT extends IntegrationTestBase {
    
    @Autowired
    private ProductRepository productRepository;
    
    @Test
    @Transactional
    @Rollback
    void testComplexProductQuery() {
        // Given
        Category electronics = new Category("Electronics");
        Product laptop = Product.builder()
            .name("Gaming Laptop")
            .price(new BigDecimal("1499.99"))
            .category(electronics)
            .attributes(Map.of("RAM", "16GB", "CPU", "Intel i7"))
            .build();
        
        productRepository.save(laptop);
        
        // When
        Page<Product> results = productRepository.findByCategoryAndPriceRange(
            "Electronics", 
            new BigDecimal("1000"), 
            new BigDecimal("2000"),
            PageRequest.of(0, 10)
        );
        
        // Then
        assertThat(results.getTotalElements()).isEqualTo(1);
        assertThat(results.getContent().get(0).getName()).isEqualTo("Gaming Laptop");
    }
    
    @Test
    void testOptimisticLocking() {
        // Given
        Product product = productRepository.save(
            Product.builder().name("Test").price(BigDecimal.TEN).build()
        );
        
        // When - Simulate concurrent updates
        Product product1 = productRepository.findById(product.getId()).orElseThrow();
        Product product2 = productRepository.findById(product.getId()).orElseThrow();
        
        product1.setPrice(new BigDecimal("20"));
        productRepository.save(product1);
        
        product2.setPrice(new BigDecimal("30"));
        
        // Then
        assertThatThrownBy(() -> productRepository.save(product2))
            .isInstanceOf(OptimisticLockingFailureException.class);
    }
}
```

### 2. Kafka Integration Tests

```java
@SpringBootTest
@EmbeddedKafka(
    partitions = 1,
    topics = {"product-events", "order-events", "inventory-events"}
)
@TestPropertySource(properties = {
    "spring.kafka.consumer.auto-offset-reset=earliest",
    "spring.kafka.consumer.group-id=test-group"
})
public class KafkaIntegrationTest extends IntegrationTestBase {
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    @Autowired
    private CountDownLatchContainer latchContainer;
    
    @SpyBean
    private OrderEventHandler orderEventHandler;
    
    @Test
    void testOrderCreatedEventFlow() throws InterruptedException {
        // Given
        OrderCreatedEvent event = OrderCreatedEvent.builder()
            .orderId("ORD-123")
            .customerId("CUST-456")
            .items(List.of(
                new OrderItem("PROD-1", 2, new BigDecimal("50.00")),
                new OrderItem("PROD-2", 1, new BigDecimal("30.00"))
            ))
            .totalAmount(new BigDecimal("130.00"))
            .timestamp(Instant.now())
            .build();
        
        CountDownLatch latch = latchContainer.getLatch("order-created", 1);
        
        // When
        kafkaTemplate.send("order-events", event.getOrderId(), event);
        
        // Then
        assertThat(latch.await(10, TimeUnit.SECONDS)).isTrue();
        verify(orderEventHandler).handleOrderCreated(argThat(e -> 
            e.getOrderId().equals("ORD-123") && 
            e.getTotalAmount().compareTo(new BigDecimal("130.00")) == 0
        ));
    }
    
    @Test
    void testEventChainReaction() throws InterruptedException {
        // Test complete event flow: Order -> Inventory -> Notification
        
        // Given
        CountDownLatch orderLatch = new CountDownLatch(1);
        CountDownLatch inventoryLatch = new CountDownLatch(1);
        CountDownLatch notificationLatch = new CountDownLatch(1);
        
        // Setup listeners
        kafkaTemplate.setProducerListener(new ProducerListener<String, Object>() {
            @Override
            public void onSuccess(ProducerRecord<String, Object> producerRecord, 
                                RecordMetadata recordMetadata) {
                if (producerRecord.topic().equals("order-events")) orderLatch.countDown();
                if (producerRecord.topic().equals("inventory-events")) inventoryLatch.countDown();
                if (producerRecord.topic().equals("notification-events")) notificationLatch.countDown();
            }
        });
        
        // When - Create order
        OrderCreatedEvent orderEvent = createTestOrderEvent();
        kafkaTemplate.send("order-events", orderEvent.getOrderId(), orderEvent);
        
        // Then - Verify cascade
        assertThat(orderLatch.await(5, TimeUnit.SECONDS)).isTrue();
        assertThat(inventoryLatch.await(5, TimeUnit.SECONDS)).isTrue();
        assertThat(notificationLatch.await(5, TimeUnit.SECONDS)).isTrue();
    }
}

@Component
public class CountDownLatchContainer {
    private final Map<String, CountDownLatch> latches = new ConcurrentHashMap<>();
    
    public CountDownLatch getLatch(String key, int count) {
        return latches.computeIfAbsent(key, k -> new CountDownLatch(count));
    }
    
    @KafkaListener(topics = "order-events")
    public void handleOrderEvent(OrderCreatedEvent event) {
        Optional.ofNullable(latches.get("order-created")).ifPresent(CountDownLatch::countDown);
    }
}
```

### 3. Redis Integration Tests

```java
@SpringBootTest
@AutoConfigureMockMvc
public class RedisCacheIntegrationTest extends IntegrationTestBase {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Autowired
    private ProductService productService;
    
    @Autowired
    private ProductRepository productRepository;
    
    @Test
    void testCachingBehavior() {
        // Given
        Product product = productRepository.save(
            Product.builder().name("Cached Product").price(BigDecimal.valueOf(99.99)).build()
        );
        
        // When - First call should hit database
        Product result1 = productService.findById(product.getId());
        
        // Verify cache is populated
        String cacheKey = "product:" + product.getId();
        Product cachedProduct = (Product) redisTemplate.opsForValue().get(cacheKey);
        assertThat(cachedProduct).isNotNull();
        assertThat(cachedProduct.getName()).isEqualTo("Cached Product");
        
        // When - Second call should hit cache
        Product result2 = productService.findById(product.getId());
        
        // Then
        assertThat(result2).isEqualTo(result1);
        
        // Verify database wasn't called again (would need to check logs or use spy)
    }
    
    @Test
    void testCacheEviction() {
        // Given
        Product product = productRepository.save(
            Product.builder().name("Test Product").price(BigDecimal.valueOf(50)).build()
        );
        
        // Populate cache
        productService.findById(product.getId());
        
        // When - Update should evict cache
        product.setPrice(BigDecimal.valueOf(60));
        productService.updateProduct(product);
        
        // Then - Cache should be empty
        String cacheKey = "product:" + product.getId();
        assertThat(redisTemplate.opsForValue().get(cacheKey)).isNull();
        
        // Next call should repopulate cache
        Product updated = productService.findById(product.getId());
        assertThat(updated.getPrice()).isEqualByComparingTo(BigDecimal.valueOf(60));
    }
}
```

## Inter-Service Communication Testing

### 1. REST Client Integration

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
public class InterServiceCommunicationIT {
    
    @LocalServerPort
    private int port;
    
    @MockBean
    private InventoryServiceClient inventoryClient;
    
    @MockBean
    private PaymentServiceClient paymentClient;
    
    @Autowired
    private OrderService orderService;
    
    @Test
    void testOrderCreationWithExternalServices() {
        // Given - Mock external service responses
        when(inventoryClient.checkAvailability(anyList()))
            .thenReturn(InventoryCheckResponse.builder()
                .available(true)
                .items(List.of(
                    new InventoryItem("PROD-1", 10, true),
                    new InventoryItem("PROD-2", 5, true)
                ))
                .build());
        
        when(paymentClient.processPayment(any()))
            .thenReturn(PaymentResponse.builder()
                .transactionId("TXN-123")
                .status(PaymentStatus.APPROVED)
                .amount(new BigDecimal("130.00"))
                .build());
        
        // When
        CreateOrderRequest request = CreateOrderRequest.builder()
            .customerId("CUST-123")
            .items(List.of(
                new OrderItemRequest("PROD-1", 2),
                new OrderItemRequest("PROD-2", 1)
            ))
            .paymentMethod(PaymentMethod.CREDIT_CARD)
            .build();
        
        Order order = orderService.createOrder(request);
        
        // Then
        assertThat(order).isNotNull();
        assertThat(order.getStatus()).isEqualTo(OrderStatus.CONFIRMED);
        assertThat(order.getPaymentDetails().getTransactionId()).isEqualTo("TXN-123");
        
        // Verify service calls
        verify(inventoryClient).checkAvailability(argThat(items -> items.size() == 2));
        verify(inventoryClient).reserveItems(any());
        verify(paymentClient).processPayment(argThat(req -> 
            req.getAmount().compareTo(new BigDecimal("130.00")) == 0
        ));
    }
    
    @Test
    void testOrderCreationRollbackOnPaymentFailure() {
        // Given
        when(inventoryClient.checkAvailability(anyList()))
            .thenReturn(InventoryCheckResponse.available());
        
        when(inventoryClient.reserveItems(any()))
            .thenReturn(ReservationResponse.success("RES-123"));
        
        when(paymentClient.processPayment(any()))
            .thenReturn(PaymentResponse.failed("Insufficient funds"));
        
        // When/Then
        CreateOrderRequest request = createTestOrderRequest();
        
        assertThatThrownBy(() -> orderService.createOrder(request))
            .isInstanceOf(PaymentFailedException.class)
            .hasMessageContaining("Insufficient funds");
        
        // Verify rollback
        verify(inventoryClient).cancelReservation("RES-123");
    }
}
```

### 2. Circuit Breaker Testing

```java
@SpringBootTest
@Import(TestCircuitBreakerConfig.class)
public class CircuitBreakerIntegrationTest {
    
    @Autowired
    private ProductService productService;
    
    @MockBean
    private InventoryServiceClient inventoryClient;
    
    @Autowired
    private CircuitBreakerRegistry circuitBreakerRegistry;
    
    @Test
    void testCircuitBreakerOpensAfterFailures() {
        // Given
        when(inventoryClient.getStock(anyLong()))
            .thenThrow(new RuntimeException("Service unavailable"));
        
        // When - Make calls until circuit opens
        for (int i = 0; i < 5; i++) {
            try {
                productService.getProductWithStock(1L);
            } catch (Exception ignored) {
                // Expected failures
            }
        }
        
        // Then
        CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker("inventory-service");
        assertThat(circuitBreaker.getState()).isEqualTo(CircuitBreaker.State.OPEN);
        
        // Verify fallback is used
        ProductWithStock result = productService.getProductWithStock(1L);
        assertThat(result.getStockLevel()).isEqualTo(-1); // Fallback value
        assertThat(result.isStockCheckFailed()).isTrue();
    }
}
```

## End-to-End Workflow Testing

### 1. Complete Order Flow Test

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
public class OrderWorkflowE2ETest extends IntegrationTestBase {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Autowired
    private ProductRepository productRepository;
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    private static String orderId;
    private static String authToken;
    
    @BeforeAll
    static void authenticate() throws Exception {
        // Get auth token for subsequent requests
        authToken = authenticateTestUser();
    }
    
    @Test
    @Order(1)
    void step1_CreateProducts() throws Exception {
        // Create test products
        Product product1 = productRepository.save(
            Product.builder()
                .name("Test Product 1")
                .price(new BigDecimal("50.00"))
                .stockQuantity(100)
                .build()
        );
        
        Product product2 = productRepository.save(
            Product.builder()
                .name("Test Product 2")
                .price(new BigDecimal("30.00"))
                .stockQuantity(50)
                .build()
        );
        
        assertThat(product1.getId()).isNotNull();
        assertThat(product2.getId()).isNotNull();
    }
    
    @Test
    @Order(2)
    void step2_PlaceOrder() throws Exception {
        // Place order via REST API
        String orderRequest = """
            {
                "customerId": "CUST-123",
                "items": [
                    {"productId": 1, "quantity": 2},
                    {"productId": 2, "quantity": 1}
                ],
                "shippingAddress": {
                    "street": "123 Main St",
                    "city": "Seattle",
                    "state": "WA",
                    "zipCode": "98101"
                },
                "paymentMethod": "CREDIT_CARD",
                "cardNumber": "4111111111111111"
            }
            """;
        
        MvcResult result = mockMvc.perform(post("/api/v1/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content(orderRequest)
                .header("Authorization", "Bearer " + authToken))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.status").value("PENDING"))
            .andExpect(jsonPath("$.totalAmount").value(130.00))
            .andReturn();
        
        orderId = JsonPath.parse(result.getResponse().getContentAsString())
            .read("$.orderId", String.class);
    }
    
    @Test
    @Order(3)
    void step3_VerifyInventoryReduction() {
        // Verify inventory was reduced
        Product product1 = productRepository.findById(1L).orElseThrow();
        Product product2 = productRepository.findById(2L).orElseThrow();
        
        assertThat(product1.getStockQuantity()).isEqualTo(98); // 100 - 2
        assertThat(product2.getStockQuantity()).isEqualTo(49); // 50 - 1
    }
    
    @Test
    @Order(4)
    void step4_ProcessPayment() throws Exception {
        // Simulate payment processing
        await().atMost(Duration.ofSeconds(10)).untilAsserted(() -> {
            com.firstviscount.order.domain.Order order = orderRepository
                .findByOrderNumber(orderId)
                .orElseThrow();
            assertThat(order.getStatus()).isEqualTo(OrderStatus.PAYMENT_CONFIRMED);
        });
    }
    
    @Test
    @Order(5)
    void step5_VerifyNotificationsSent() {
        // Verify notification events were published
        NotificationRecorder recorder = new NotificationRecorder();
        
        await().atMost(Duration.ofSeconds(10)).untilAsserted(() -> {
            assertThat(recorder.getNotifications()).anyMatch(n -> 
                n.getType().equals("ORDER_CONFIRMATION") &&
                n.getRecipient().equals("CUST-123")
            );
        });
    }
    
    @Test
    @Order(6)
    void step6_CreateShipment() throws Exception {
        // Create shipment for the order
        mockMvc.perform(post("/api/v1/orders/{orderId}/ship", orderId)
                .header("Authorization", "Bearer " + authToken))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.trackingNumber").exists())
            .andExpect(jsonPath("$.carrier").value("FEDEX"));
    }
    
    @Test
    @Order(7)
    void step7_CompleteDelivery() throws Exception {
        // Simulate delivery completion
        String deliveryUpdate = """
            {
                "status": "DELIVERED",
                "location": "Customer doorstep",
                "signature": "John Doe",
                "deliveredAt": "2024-01-15T14:30:00Z"
            }
            """;
        
        mockMvc.perform(patch("/api/v1/deliveries/tracking/{trackingNumber}", "TRK-" + orderId)
                .contentType(MediaType.APPLICATION_JSON)
                .content(deliveryUpdate)
                .header("Authorization", "Bearer " + authToken))
            .andExpect(status().isOk());
        
        // Verify order status updated
        await().atMost(Duration.ofSeconds(10)).untilAsserted(() -> {
            com.firstviscount.order.domain.Order order = orderRepository
                .findByOrderNumber(orderId)
                .orElseThrow();
            assertThat(order.getStatus()).isEqualTo(OrderStatus.DELIVERED);
        });
    }
}
```

### 2. Saga Pattern Testing

```java
@SpringBootTest
@ActiveProfiles("saga-test")
public class SagaIntegrationTest extends IntegrationTestBase {
    
    @Autowired
    private SagaOrchestrator sagaOrchestrator;
    
    @Autowired
    private SagaRepository sagaRepository;
    
    @MockBean
    private InventoryService inventoryService;
    
    @MockBean
    private PaymentService paymentService;
    
    @MockBean
    private ShippingService shippingService;
    
    @Test
    void testSuccessfulOrderSaga() {
        // Given
        mockSuccessfulServices();
        
        OrderSagaContext context = OrderSagaContext.builder()
            .orderId("ORD-123")
            .customerId("CUST-456")
            .items(createTestItems())
            .paymentDetails(createTestPaymentDetails())
            .build();
        
        // When
        String sagaId = sagaOrchestrator.startSaga("ORDER_PLACEMENT", context);
        
        // Then
        await().atMost(Duration.ofSeconds(30)).untilAsserted(() -> {
            SagaInstance saga = sagaRepository.findById(sagaId).orElseThrow();
            assertThat(saga.getStatus()).isEqualTo(SagaStatus.COMPLETED);
            assertThat(saga.getCompletedSteps()).containsExactly(
                "VALIDATE_ORDER",
                "RESERVE_INVENTORY",
                "PROCESS_PAYMENT",
                "CREATE_SHIPMENT",
                "SEND_CONFIRMATION"
            );
        });
        
        // Verify all services were called in order
        InOrder inOrder = inOrder(inventoryService, paymentService, shippingService);
        inOrder.verify(inventoryService).reserveItems(any());
        inOrder.verify(paymentService).processPayment(any());
        inOrder.verify(shippingService).createShipment(any());
    }
    
    @Test
    void testSagaCompensation() {
        // Given - Payment will fail
        when(inventoryService.reserveItems(any()))
            .thenReturn(ReservationResult.success("RES-123"));
        
        when(paymentService.processPayment(any()))
            .thenThrow(new PaymentException("Card declined"));
        
        OrderSagaContext context = createTestOrderContext();
        
        // When
        String sagaId = sagaOrchestrator.startSaga("ORDER_PLACEMENT", context);
        
        // Then
        await().atMost(Duration.ofSeconds(30)).untilAsserted(() -> {
            SagaInstance saga = sagaRepository.findById(sagaId).orElseThrow();
            assertThat(saga.getStatus()).isEqualTo(SagaStatus.COMPENSATED);
            assertThat(saga.getCompensatedSteps()).contains("CANCEL_INVENTORY_RESERVATION");
        });
        
        // Verify compensation was called
        verify(inventoryService).cancelReservation("RES-123");
    }
    
    @Test
    void testSagaTimeout() {
        // Given - Service will timeout
        when(inventoryService.reserveItems(any()))
            .thenAnswer(invocation -> {
                Thread.sleep(35000); // Longer than saga timeout
                return ReservationResult.success("RES-123");
            });
        
        OrderSagaContext context = createTestOrderContext();
        
        // When
        String sagaId = sagaOrchestrator.startSaga("ORDER_PLACEMENT", context);
        
        // Then
        await().atMost(Duration.ofSeconds(40)).untilAsserted(() -> {
            SagaInstance saga = sagaRepository.findById(sagaId).orElseThrow();
            assertThat(saga.getStatus()).isEqualTo(SagaStatus.TIMEOUT);
            assertThat(saga.getFailureReason()).contains("timeout");
        });
    }
}
```

## Contract Testing

### 1. Spring Cloud Contract

```groovy
// Producer contract (in inventory-service)
package contracts.inventory

import org.springframework.cloud.contract.spec.Contract

Contract.make {
    description "should reserve inventory items"
    request {
        method POST()
        url "/api/v1/inventory/reserve"
        headers {
            contentType(applicationJson())
        }
        body([
            items: [
                [productId: 1, quantity: 2],
                [productId: 2, quantity: 1]
            ]
        ])
    }
    response {
        status 200
        headers {
            contentType(applicationJson())
        }
        body([
            reservationId: $(regex('[A-Z]{3}-[0-9]{6}')),
            status: "RESERVED",
            items: [
                [productId: 1, quantity: 2, reserved: true],
                [productId: 2, quantity: 1, reserved: true]
            ],
            expiresAt: $(regex(isoDateTime()))
        ])
    }
}
```

```java
// Consumer test (in order-service)
@SpringBootTest
@AutoConfigureMockMvc
public class InventoryServiceContractTest {
    
    @Autowired
    private InventoryServiceClient inventoryClient;
    
    @RegisterExtension
    static StubRunnerExtension stubRunner = new StubRunnerExtension()
        .downloadStub("com.firstviscount", "inventory-service", "1.0.0")
        .withPort(8083)
        .withStubPerConsumer(true);
    
    @Test
    void testInventoryReservation() {
        // Given
        ReservationRequest request = ReservationRequest.builder()
            .items(List.of(
                new ReservationItem(1L, 2),
                new ReservationItem(2L, 1)
            ))
            .build();
        
        // When
        ReservationResponse response = inventoryClient.reserveItems(request);
        
        // Then
        assertThat(response.getReservationId()).matches("[A-Z]{3}-[0-9]{6}");
        assertThat(response.getStatus()).isEqualTo("RESERVED");
        assertThat(response.getItems()).hasSize(2);
        assertThat(response.getItems()).allMatch(item -> item.isReserved());
    }
}
```

### 2. Pact Testing

```java
@ExtendWith(PactConsumerTestExt.class)
public class ProductServicePactTest {
    
    @Pact(consumer = "OrderService", provider = "ProductService")
    public RequestResponsePact createPact(PactDslWithProvider builder) {
        return builder
            .given("products exist")
            .uponReceiving("a request for product details")
                .path("/api/v1/products/1")
                .method("GET")
            .willRespondWith()
                .status(200)
                .headers(Map.of("Content-Type", "application/json"))
                .body(new PactDslJsonBody()
                    .integerType("id", 1)
                    .stringType("name", "Test Product")
                    .decimalType("price", 99.99)
                    .booleanType("inStock", true)
                )
            .toPact();
    }
    
    @Test
    @PactTestFor(providerName = "ProductService", port = "8082")
    void testGetProduct(MockServer mockServer) {
        // Given
        ProductClient client = new ProductClient(mockServer.getUrl());
        
        // When
        Product product = client.getProduct(1L);
        
        // Then
        assertThat(product.getId()).isEqualTo(1L);
        assertThat(product.getName()).isEqualTo("Test Product");
        assertThat(product.getPrice()).isEqualByComparingTo(new BigDecimal("99.99"));
        assertThat(product.isInStock()).isTrue();
    }
}
```

## Performance Integration Testing

```java
@Test
@Timeout(value = 5, unit = TimeUnit.SECONDS)
void testConcurrentOrderProcessing() throws InterruptedException {
    int numberOfOrders = 100;
    CountDownLatch latch = new CountDownLatch(numberOfOrders);
    List<Future<Order>> futures = new ArrayList<>();
    
    ExecutorService executor = Executors.newFixedThreadPool(10);
    
    // When - Submit concurrent orders
    for (int i = 0; i < numberOfOrders; i++) {
        int orderId = i;
        Future<Order> future = executor.submit(() -> {
            try {
                CreateOrderRequest request = createOrderRequest("CUST-" + orderId);
                return orderService.createOrder(request);
            } finally {
                latch.countDown();
            }
        });
        futures.add(future);
    }
    
    // Then - All orders should complete
    assertThat(latch.await(5, TimeUnit.SECONDS)).isTrue();
    
    List<Order> orders = futures.stream()
        .map(f -> {
            try {
                return f.get();
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        })
        .collect(Collectors.toList());
    
    assertThat(orders).hasSize(numberOfOrders);
    assertThat(orders).allMatch(o -> o.getStatus() == OrderStatus.CONFIRMED);
    
    executor.shutdown();
}
```

## Test Configuration

### Test Application Properties

```yaml
# application-integration-test.yml
spring:
  datasource:
    url: jdbc:tc:postgresql:15:///testdb
    driver-class-name: org.testcontainers.jdbc.ContainerDatabaseDriver
  
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: true
  
  kafka:
    consumer:
      auto-offset-reset: earliest
      enable-auto-commit: false
    producer:
      retries: 0
  
  cloud:
    circuitbreaker:
      resilience4j:
        configs:
          default:
            slidingWindowSize: 5
            failureRateThreshold: 50
            waitDurationInOpenState: 1000

logging:
  level:
    org.springframework.kafka: DEBUG
    org.springframework.web: DEBUG
    com.firstviscount: DEBUG

test:
  kafka:
    embedded: true
  timeout:
    default: 30s
```

### Maven Configuration

```xml
<profiles>
    <profile>
        <id>integration-test</id>
        <build>
            <plugins>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-failsafe-plugin</artifactId>
                    <configuration>
                        <includes>
                            <include>**/*IT.java</include>
                            <include>**/*IntegrationTest.java</include>
                        </includes>
                        <systemPropertyVariables>
                            <spring.profiles.active>integration-test</spring.profiles.active>
                        </systemPropertyVariables>
                    </configuration>
                    <executions>
                        <execution>
                            <goals>
                                <goal>integration-test</goal>
                                <goal>verify</goal>
                            </goals>
                        </execution>
                    </executions>
                </plugin>
            </plugins>
        </build>
    </profile>
</profiles>
```

## Best Practices

1. **Test Isolation**: Each test should be independent with its own test data
2. **Container Management**: Use TestContainers for consistent environments
3. **Async Testing**: Use proper wait strategies for eventual consistency
4. **Mock External Services**: Only test what you own
5. **Contract Testing**: Ensure API compatibility between services
6. **Performance Bounds**: Set reasonable timeouts to catch performance regressions
7. **Test Data Management**: Use builders and fixtures for maintainable test data
8. **Rollback Strategies**: Always test compensation and rollback scenarios

---
*Last Updated: January 2025*
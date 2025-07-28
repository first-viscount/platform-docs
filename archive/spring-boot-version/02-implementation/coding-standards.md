# Coding Standards

This document defines the coding standards and conventions for the First Viscount platform services built with Spring Boot and Java 21.

## Java Coding Standards

### General Principles

1. **Clarity over Cleverness**: Write code that is easy to understand
2. **Consistency**: Follow established patterns within the codebase
3. **SOLID Principles**: Design classes and interfaces following SOLID principles
4. **DRY (Don't Repeat Yourself)**: Avoid code duplication
5. **YAGNI (You Aren't Gonna Need It)**: Don't add functionality until needed

### Code Organization

#### Package Structure

```java
com.firstviscount.[service-name]
├── api                  // REST controllers
│   ├── controller      // HTTP endpoints
│   ├── dto            // Request/Response DTOs
│   └── mapper         // DTO mappers
├── application         // Application services
│   ├── service        // Business logic
│   ├── usecase        // Use case implementations
│   └── port           // Interfaces (hexagonal architecture)
├── domain             // Domain layer
│   ├── model         // Domain entities
│   ├── event         // Domain events
│   ├── exception     // Domain exceptions
│   └── repository    // Repository interfaces
├── infrastructure     // Infrastructure layer
│   ├── persistence   // JPA/Database implementation
│   ├── messaging     // Kafka implementation
│   ├── client        // External service clients
│   └── config        // Configuration classes
└── common            // Shared utilities
    ├── annotation    // Custom annotations
    ├── validation    // Validators
    └── util          // Utility classes
```

### Naming Conventions

#### Classes and Interfaces

```java
// Classes - PascalCase, noun
public class ProductCatalog { }
public class OrderService { }

// Interfaces - PascalCase, adjective or noun
public interface Searchable { }
public interface ProductRepository { }

// Abstract classes - prefix with Abstract
public abstract class AbstractEntity { }

// Enums - PascalCase, singular
public enum OrderStatus {
    PENDING, PROCESSING, COMPLETED
}

// Exceptions - suffix with Exception
public class ProductNotFoundException extends RuntimeException { }
```

#### Methods and Variables

```java
// Methods - camelCase, verb
public Product findProductById(Long id) { }
public void updateInventory(Long productId, int quantity) { }

// Variables - camelCase, meaningful names
private final ProductRepository productRepository;
private String productName;
private BigDecimal unitPrice;

// Constants - UPPER_SNAKE_CASE
public static final int MAX_RETRY_ATTEMPTS = 3;
private static final String DEFAULT_CURRENCY = "USD";

// Boolean variables - prefix with is/has/can
private boolean isActive;
private boolean hasInventory;
private boolean canShip;
```

#### Test Classes

```java
// Unit tests - suffix with Test
public class ProductServiceTest { }

// Integration tests - suffix with IT
public class ProductControllerIT { }

// Test methods - should_ExpectedBehavior_When_StateUnderTest
@Test
void should_ReturnProduct_When_ProductExists() { }

@Test
void should_ThrowException_When_ProductNotFound() { }
```

### Code Style

#### Imports

```java
// Order: java, javax, spring, other third party, com.firstviscount
import java.util.List;
import java.util.Optional;

import javax.persistence.Entity;
import javax.validation.Valid;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import com.fasterxml.jackson.annotation.JsonProperty;
import lombok.RequiredArgsConstructor;

import com.firstviscount.productcatalog.domain.model.Product;
import com.firstviscount.productcatalog.domain.repository.ProductRepository;

// No wildcard imports
// Remove unused imports
```

#### Class Structure

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class ProductService {
    
    // Constants
    private static final int DEFAULT_PAGE_SIZE = 20;
    
    // Fields (dependency injection via constructor)
    private final ProductRepository productRepository;
    private final InventoryClient inventoryClient;
    private final EventPublisher eventPublisher;
    
    // Public methods
    @Transactional
    public Product createProduct(CreateProductRequest request) {
        log.info("Creating product with name: {}", request.getName());
        
        Product product = Product.builder()
            .name(request.getName())
            .description(request.getDescription())
            .price(request.getPrice())
            .build();
            
        product = productRepository.save(product);
        
        eventPublisher.publish(new ProductCreatedEvent(product));
        
        return product;
    }
    
    // Private methods
    private void validateProduct(Product product) {
        if (product.getPrice().compareTo(BigDecimal.ZERO) <= 0) {
            throw new InvalidProductException("Product price must be positive");
        }
    }
}
```

### Spring Boot Specific Standards

#### REST Controllers

```java
@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor
@Validated
@Slf4j
@Tag(name = "Product Management", description = "Product catalog operations")
public class ProductController {
    
    private final ProductService productService;
    
    @GetMapping("/{id}")
    @Operation(summary = "Get product by ID")
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "Product found"),
        @ApiResponse(responseCode = "404", description = "Product not found")
    })
    public ResponseEntity<ProductResponse> getProduct(
            @PathVariable @Positive Long id) {
        
        log.debug("Fetching product with id: {}", id);
        
        return productService.findById(id)
            .map(product -> ResponseEntity.ok(toResponse(product)))
            .orElse(ResponseEntity.notFound().build());
    }
    
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    @Operation(summary = "Create new product")
    public ProductResponse createProduct(
            @Valid @RequestBody CreateProductRequest request) {
        
        log.info("Creating new product: {}", request.getName());
        
        Product product = productService.createProduct(request);
        return toResponse(product);
    }
    
    @ExceptionHandler(ProductNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(ProductNotFoundException ex) {
        return ErrorResponse.of(ex.getMessage());
    }
}
```

#### Service Layer

```java
@Service
@RequiredArgsConstructor
@Slf4j
@Transactional(readOnly = true)
public class OrderService {
    
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;
    private final EventPublisher eventPublisher;
    
    @Transactional
    @Retryable(value = {TransientException.class}, maxAttempts = 3)
    public Order createOrder(CreateOrderCommand command) {
        log.info("Processing order for customer: {}", command.getCustomerId());
        
        // Validate
        validateOrderCommand(command);
        
        // Create order
        Order order = Order.create(
            command.getCustomerId(),
            command.getItems()
        );
        
        // Process payment
        PaymentResult paymentResult = paymentService.processPayment(
            order.getTotalAmount(),
            command.getPaymentMethod()
        );
        
        order.confirmPayment(paymentResult.getTransactionId());
        
        // Save and publish event
        order = orderRepository.save(order);
        eventPublisher.publish(new OrderCreatedEvent(order));
        
        return order;
    }
    
    @Cacheable(value = "orders", key = "#id")
    public Optional<Order> findById(Long id) {
        return orderRepository.findById(id);
    }
}
```

#### Repository Layer

```java
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {
    
    @Query("SELECT p FROM Product p WHERE p.category = :category AND p.active = true")
    Page<Product> findActiveByCategory(@Param("category") String category, Pageable pageable);
    
    @Modifying
    @Query("UPDATE Product p SET p.inventory = p.inventory - :quantity WHERE p.id = :id AND p.inventory >= :quantity")
    int decrementInventory(@Param("id") Long id, @Param("quantity") int quantity);
    
    Optional<Product> findBySkuAndActive(String sku, boolean active);
    
    @EntityGraph(attributePaths = {"category", "images"})
    Optional<Product> findWithDetailsById(Long id);
}
```

#### DTOs and Validation

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class CreateProductRequest {
    
    @NotBlank(message = "Product name is required")
    @Size(min = 3, max = 100, message = "Product name must be between 3 and 100 characters")
    private String name;
    
    @Size(max = 500, message = "Description cannot exceed 500 characters")
    private String description;
    
    @NotNull(message = "Price is required")
    @DecimalMin(value = "0.01", message = "Price must be greater than 0")
    @Digits(integer = 10, fraction = 2, message = "Price format is invalid")
    private BigDecimal price;
    
    @NotNull(message = "Category is required")
    private Long categoryId;
    
    @Min(value = 0, message = "Initial inventory cannot be negative")
    private Integer initialInventory = 0;
    
    @Valid
    private List<ProductAttributeDto> attributes;
}
```

### Exception Handling

#### Custom Exceptions

```java
public class ProductNotFoundException extends BusinessException {
    
    public ProductNotFoundException(Long productId) {
        super(
            ErrorCode.PRODUCT_NOT_FOUND,
            String.format("Product with ID %d not found", productId)
        );
    }
}

@ResponseStatus(HttpStatus.BAD_REQUEST)
public abstract class BusinessException extends RuntimeException {
    
    private final ErrorCode errorCode;
    
    protected BusinessException(ErrorCode errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }
    
    public ErrorCode getErrorCode() {
        return errorCode;
    }
}
```

#### Global Exception Handler

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
    
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusinessException(BusinessException ex) {
        log.warn("Business exception: {}", ex.getMessage());
        
        ErrorResponse response = ErrorResponse.builder()
            .code(ex.getErrorCode().name())
            .message(ex.getMessage())
            .timestamp(Instant.now())
            .build();
            
        return ResponseEntity
            .status(ex.getErrorCode().getHttpStatus())
            .body(response);
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidationException(MethodArgumentNotValidException ex) {
        Map<String, String> errors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .collect(Collectors.toMap(
                FieldError::getField,
                FieldError::getDefaultMessage
            ));
            
        return ErrorResponse.builder()
            .code("VALIDATION_ERROR")
            .message("Validation failed")
            .details(errors)
            .timestamp(Instant.now())
            .build();
    }
}
```

### Testing Standards

#### Unit Tests

```java
@ExtendWith(MockitoExtension.class)
class ProductServiceTest {
    
    @Mock
    private ProductRepository productRepository;
    
    @Mock
    private EventPublisher eventPublisher;
    
    @InjectMocks
    private ProductService productService;
    
    @Test
    @DisplayName("Should create product successfully")
    void should_CreateProduct_When_ValidRequest() {
        // Given
        CreateProductRequest request = CreateProductRequest.builder()
            .name("Test Product")
            .price(new BigDecimal("99.99"))
            .categoryId(1L)
            .build();
            
        Product savedProduct = Product.builder()
            .id(1L)
            .name(request.getName())
            .price(request.getPrice())
            .build();
            
        when(productRepository.save(any(Product.class)))
            .thenReturn(savedProduct);
        
        // When
        Product result = productService.createProduct(request);
        
        // Then
        assertThat(result).isNotNull();
        assertThat(result.getName()).isEqualTo("Test Product");
        assertThat(result.getPrice()).isEqualByComparingTo(new BigDecimal("99.99"));
        
        verify(eventPublisher).publish(any(ProductCreatedEvent.class));
    }
    
    @Test
    @DisplayName("Should throw exception when product not found")
    void should_ThrowException_When_ProductNotFound() {
        // Given
        Long productId = 999L;
        when(productRepository.findById(productId))
            .thenReturn(Optional.empty());
        
        // When & Then
        assertThatThrownBy(() -> productService.findById(productId))
            .isInstanceOf(ProductNotFoundException.class)
            .hasMessageContaining("Product with ID 999 not found");
    }
}
```

#### Integration Tests

```java
@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles("test")
@TestPropertySource(locations = "classpath:application-test.properties")
@Sql(scripts = "classpath:test-data.sql")
class ProductControllerIT {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Autowired
    private ProductRepository productRepository;
    
    @Test
    @DisplayName("Should return product when exists")
    void should_ReturnProduct_When_ProductExists() throws Exception {
        // Given
        Product product = productRepository.save(
            Product.builder()
                .name("Integration Test Product")
                .price(new BigDecimal("49.99"))
                .build()
        );
        
        // When & Then
        mockMvc.perform(get("/api/v1/products/{id}", product.getId())
                .contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").value(product.getId()))
            .andExpect(jsonPath("$.name").value("Integration Test Product"))
            .andExpect(jsonPath("$.price").value(49.99));
    }
}
```

### Configuration Standards

#### Application Properties

```yaml
# application.yml
spring:
  application:
    name: product-catalog-service
  
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:products}
    username: ${DB_USER:postgres}
    password: ${DB_PASSWORD:password}
    hikari:
      maximum-pool-size: ${DB_POOL_SIZE:10}
      minimum-idle: ${DB_MIN_IDLE:5}
      connection-timeout: ${DB_TIMEOUT:30000}
  
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true
        jdbc:
          time_zone: UTC
  
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all
      retries: 3

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  metrics:
    tags:
      application: ${spring.application.name}

logging:
  level:
    com.firstviscount: ${LOG_LEVEL:INFO}
    org.springframework.web: INFO
    org.hibernate.SQL: ${SQL_LOG_LEVEL:WARN}
```

#### Configuration Classes

```java
@Configuration
@EnableKafka
@EnableCaching
@EnableAsync
@ConfigurationProperties(prefix = "app")
@Data
public class ApplicationConfig {
    
    private Retry retry = new Retry();
    private Cache cache = new Cache();
    private Security security = new Security();
    
    @Data
    public static class Retry {
        private int maxAttempts = 3;
        private long backoffDelay = 1000;
    }
    
    @Data
    public static class Cache {
        private int timeToLiveMinutes = 60;
        private int maxEntries = 1000;
    }
    
    @Data
    public static class Security {
        private String jwtSecret;
        private long jwtExpirationMs = 86400000; // 24 hours
    }
    
    @Bean
    public RestTemplate restTemplate(RestTemplateBuilder builder) {
        return builder
            .setConnectTimeout(Duration.ofSeconds(5))
            .setReadTimeout(Duration.ofSeconds(10))
            .interceptors(new LoggingRestTemplateInterceptor())
            .build();
    }
    
    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager();
        cacheManager.setCaffeine(Caffeine.newBuilder()
            .expireAfterWrite(cache.getTimeToLiveMinutes(), TimeUnit.MINUTES)
            .maximumSize(cache.getMaxEntries()));
        return cacheManager;
    }
}
```

### Documentation Standards

#### JavaDoc

```java
/**
 * Service responsible for managing product catalog operations.
 * This service handles CRUD operations for products and publishes
 * relevant events to the event bus.
 *
 * @author John Doe
 * @since 1.0.0
 */
@Service
public class ProductService {
    
    /**
     * Creates a new product in the catalog.
     *
     * @param request the product creation request containing product details
     * @return the created product entity
     * @throws InvalidProductException if the product data is invalid
     * @throws DuplicateProductException if a product with the same SKU already exists
     */
    public Product createProduct(CreateProductRequest request) {
        // Implementation
    }
    
    /**
     * Finds a product by its unique identifier.
     *
     * @param id the product ID
     * @return an Optional containing the product if found, empty otherwise
     */
    public Optional<Product> findById(Long id) {
        // Implementation
    }
}
```

#### API Documentation

```java
@Operation(
    summary = "Create a new product",
    description = "Creates a new product in the catalog with the provided details"
)
@ApiResponses(value = {
    @ApiResponse(
        responseCode = "201",
        description = "Product created successfully",
        content = @Content(schema = @Schema(implementation = ProductResponse.class))
    ),
    @ApiResponse(
        responseCode = "400",
        description = "Invalid product data",
        content = @Content(schema = @Schema(implementation = ErrorResponse.class))
    ),
    @ApiResponse(
        responseCode = "409",
        description = "Product with same SKU already exists"
    )
})
@PostMapping
public ResponseEntity<ProductResponse> createProduct(
    @RequestBody @Valid CreateProductRequest request) {
    // Implementation
}
```

### Security Standards

#### Authentication & Authorization

```java
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .cors().and()
            .csrf().disable()
            .sessionManagement()
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            .and()
            .authorizeHttpRequests()
                .requestMatchers("/api/v1/auth/**").permitAll()
                .requestMatchers("/api/v1/products/**").hasRole("USER")
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            .and()
            .oauth2ResourceServer()
                .jwt();
                
        return http.build();
    }
}
```

#### Input Validation

```java
@Component
public class InputSanitizer {
    
    private static final Pattern XSS_PATTERN = Pattern.compile(
        "(<script>|</script>|<iframe>|javascript:|onerror=)",
        Pattern.CASE_INSENSITIVE
    );
    
    public String sanitize(String input) {
        if (input == null) {
            return null;
        }
        
        // Remove potential XSS
        if (XSS_PATTERN.matcher(input).find()) {
            throw new SecurityException("Potentially malicious input detected");
        }
        
        // HTML encode
        return HtmlUtils.htmlEscape(input);
    }
}
```

### Performance Standards

#### Database Queries

```java
// Use projections for read-only operations
public interface ProductProjection {
    Long getId();
    String getName();
    BigDecimal getPrice();
}

@Query("SELECT p.id as id, p.name as name, p.price as price FROM Product p WHERE p.category = :category")
List<ProductProjection> findProductsByCategory(@Param("category") String category);

// Use batch operations
@Modifying
@Query("UPDATE Product p SET p.lastModified = :timestamp WHERE p.id IN :ids")
void updateLastModifiedBatch(@Param("ids") List<Long> ids, @Param("timestamp") Instant timestamp);

// Use pagination
Page<Product> findByActiveTrue(Pageable pageable);
```

#### Caching

```java
@Cacheable(value = "products", key = "#id", unless = "#result == null")
public Optional<Product> findById(Long id) {
    return productRepository.findById(id);
}

@CacheEvict(value = "products", key = "#product.id")
public Product updateProduct(Product product) {
    return productRepository.save(product);
}

@CacheEvict(value = "products", allEntries = true)
@Scheduled(fixedDelay = 3600000) // 1 hour
public void evictAllProductsCache() {
    log.info("Evicting all products cache");
}
```

### Code Quality Tools

#### Checkstyle Configuration

```xml
<!-- checkstyle.xml -->
<module name="Checker">
    <module name="TreeWalker">
        <module name="LineLength">
            <property name="max" value="120"/>
        </module>
        <module name="MethodLength">
            <property name="max" value="50"/>
        </module>
        <module name="ParameterNumber">
            <property name="max" value="5"/>
        </module>
        <module name="CyclomaticComplexity">
            <property name="max" value="10"/>
        </module>
    </module>
</module>
```

#### SonarQube Rules

- Code coverage: minimum 80%
- Duplicated lines: maximum 3%
- Technical debt ratio: maximum 5%
- Maintainability rating: A
- Reliability rating: A
- Security rating: A

---
*Last Updated: January 2025*
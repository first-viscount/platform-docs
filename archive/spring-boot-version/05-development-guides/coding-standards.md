# Coding Standards Guide

This document defines the coding standards and best practices for the First Viscount microservices platform. All developers must follow these guidelines to ensure code quality, maintainability, and consistency.

## Java Coding Standards

### 1. General Principles

- **Clean Code**: Write code that is easy to read, understand, and maintain
- **SOLID Principles**: Follow Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, and Dependency Inversion
- **DRY (Don't Repeat Yourself)**: Avoid code duplication
- **YAGNI (You Aren't Gonna Need It)**: Don't add functionality until it's needed
- **Boy Scout Rule**: Leave the code better than you found it

### 2. Naming Conventions

#### Classes and Interfaces
```java
// Classes - PascalCase, noun
public class ProductService { }
public class OrderRepository { }

// Interfaces - PascalCase, adjective or noun
public interface Cacheable { }
public interface ProductRepository { }

// Abstract classes - prefix with Abstract
public abstract class AbstractEntity { }

// Enums - PascalCase, singular
public enum OrderStatus {
    PENDING, CONFIRMED, SHIPPED, DELIVERED
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
private String productName;
private BigDecimal unitPrice;
private LocalDateTime createdAt;

// Constants - UPPER_SNAKE_CASE
public static final int MAX_RETRY_ATTEMPTS = 3;
private static final String DEFAULT_CURRENCY = "USD";

// Boolean variables - use is/has/can prefix
private boolean isActive;
private boolean hasInventory;
private boolean canShip;
```

#### Packages
```java
// Package names - lowercase, reverse domain
package com.firstviscount.product.service;
package com.firstviscount.order.repository;
package com.firstviscount.common.util;
```

### 3. Code Organization

#### Class Structure
```java
package com.firstviscount.product.service;

import java.util.List;  // Java imports first
import java.util.Optional;

import org.springframework.stereotype.Service;  // Third-party imports

import com.firstviscount.product.domain.Product;  // Project imports
import com.firstviscount.product.repository.ProductRepository;

/**
 * Service class for managing products.
 * Handles business logic related to product operations.
 */
@Service
@Slf4j
public class ProductService {
    
    // Constants
    private static final int DEFAULT_PAGE_SIZE = 20;
    
    // Fields
    private final ProductRepository productRepository;
    private final InventoryService inventoryService;
    private final CacheManager cacheManager;
    
    // Constructor
    public ProductService(ProductRepository productRepository,
                         InventoryService inventoryService,
                         CacheManager cacheManager) {
        this.productRepository = productRepository;
        this.inventoryService = inventoryService;
        this.cacheManager = cacheManager;
    }
    
    // Public methods
    public Product createProduct(CreateProductRequest request) {
        // Implementation
    }
    
    // Private methods
    private void validateProduct(Product product) {
        // Implementation
    }
    
    // Inner classes (if needed)
    @Data
    private static class ProductCacheKey {
        private final Long productId;
        private final String tenantId;
    }
}
```

### 4. Method Guidelines

#### Method Length
```java
// GOOD - Short, focused methods
public BigDecimal calculateTotalPrice(List<OrderItem> items) {
    return items.stream()
        .map(this::calculateItemPrice)
        .reduce(BigDecimal.ZERO, BigDecimal::add);
}

private BigDecimal calculateItemPrice(OrderItem item) {
    return item.getUnitPrice()
        .multiply(BigDecimal.valueOf(item.getQuantity()))
        .subtract(calculateDiscount(item));
}

// BAD - Long method doing too many things
public Order processOrder(OrderRequest request) {
    // 100+ lines of code doing validation, calculation, 
    // database updates, event publishing, etc.
}
```

#### Method Parameters
```java
// GOOD - Limited parameters
public Product updateProduct(Long id, UpdateProductRequest request) {
    // Implementation
}

// GOOD - Use parameter object for many parameters
public Order createOrder(CreateOrderRequest request) {
    // Implementation
}

// BAD - Too many parameters
public Order createOrder(Long customerId, List<OrderItem> items, 
                        Address shippingAddress, Address billingAddress,
                        PaymentMethod paymentMethod, String couponCode,
                        boolean expressShipping, String notes) {
    // Implementation
}
```

### 5. Exception Handling

```java
// Service layer exception handling
@Service
@Slf4j
public class ProductService {
    
    public Product findById(Long id) {
        return productRepository.findById(id)
            .orElseThrow(() -> new ProductNotFoundException(
                String.format("Product not found with id: %d", id)
            ));
    }
    
    @Transactional
    public Product updateProduct(Long id, UpdateProductRequest request) {
        try {
            Product product = findById(id);
            mapper.updateProduct(product, request);
            
            // Publish event
            eventPublisher.publish(new ProductUpdatedEvent(product));
            
            return productRepository.save(product);
        } catch (DataIntegrityViolationException e) {
            log.error("Database constraint violation updating product: {}", id, e);
            throw new BusinessException("Product update failed due to data conflict", e);
        } catch (Exception e) {
            log.error("Unexpected error updating product: {}", id, e);
            throw new SystemException("Failed to update product", e);
        }
    }
}

// Global exception handler
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
    
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException e) {
        log.warn("Resource not found: {}", e.getMessage());
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(ErrorResponse.of(e.getMessage(), "NOT_FOUND"));
    }
    
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusinessException(BusinessException e) {
        log.warn("Business exception: {}", e.getMessage());
        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
            .body(ErrorResponse.of(e.getMessage(), e.getErrorCode()));
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneral(Exception e) {
        log.error("Unexpected error", e);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ErrorResponse.of("An unexpected error occurred", "INTERNAL_ERROR"));
    }
}
```

### 6. Comments and Documentation

```java
/**
 * Service responsible for managing product catalog operations.
 * 
 * <p>This service handles CRUD operations for products, inventory
 * synchronization, and product search functionality.</p>
 * 
 * @author John Doe
 * @since 1.0.0
 */
@Service
public class ProductService {
    
    /**
     * Creates a new product in the catalog.
     * 
     * <p>This method validates the product data, checks for duplicates,
     * and publishes a ProductCreatedEvent upon successful creation.</p>
     * 
     * @param request the product creation request containing product details
     * @return the created product with generated ID
     * @throws ValidationException if the product data is invalid
     * @throws DuplicateProductException if a product with the same SKU exists
     */
    public Product createProduct(CreateProductRequest request) {
        // Validate request
        validateProductRequest(request);
        
        // Check for duplicate SKU
        if (productRepository.existsBySku(request.getSku())) {
            throw new DuplicateProductException(
                "Product with SKU already exists: " + request.getSku()
            );
        }
        
        // Create and save product
        Product product = mapper.toProduct(request);
        product = productRepository.save(product);
        
        // Publish event
        eventPublisher.publish(new ProductCreatedEvent(product));
        
        return product;
    }
    
    // Avoid obvious comments
    // BAD: 
    // Increment i by 1
    i++;
    
    // GOOD: Explain why, not what
    // Retry counter must not exceed max attempts to prevent infinite loops
    if (++retryCount > MAX_RETRY_ATTEMPTS) {
        throw new MaxRetriesExceededException();
    }
}
```

### 7. Spring Boot Best Practices

#### Dependency Injection
```java
// GOOD - Constructor injection
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;
    private final NotificationService notificationService;
}

// BAD - Field injection
@Service
public class OrderService {
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private PaymentService paymentService;
}
```

#### Configuration
```java
@Configuration
@ConfigurationProperties(prefix = "app.kafka")
@Validated
@Data
public class KafkaProperties {
    
    @NotBlank
    private String bootstrapServers;
    
    @NotNull
    @Valid
    private Producer producer = new Producer();
    
    @NotNull
    @Valid
    private Consumer consumer = new Consumer();
    
    @Data
    public static class Producer {
        @NotBlank
        private String clientId;
        
        @Min(1)
        @Max(5)
        private int retries = 3;
        
        private Duration requestTimeout = Duration.ofSeconds(30);
    }
    
    @Data
    public static class Consumer {
        @NotBlank
        private String groupId;
        
        @NotNull
        private AutoOffsetReset autoOffsetReset = AutoOffsetReset.EARLIEST;
        
        @Min(1)
        private int maxPollRecords = 500;
    }
    
    public enum AutoOffsetReset {
        EARLIEST, LATEST, NONE
    }
}
```

#### REST Controllers
```java
@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor
@Validated
@Tag(name = "Product Management", description = "APIs for managing products")
public class ProductController {
    
    private final ProductService productService;
    
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    @Operation(summary = "Create a new product")
    @ApiResponses({
        @ApiResponse(responseCode = "201", description = "Product created successfully"),
        @ApiResponse(responseCode = "400", description = "Invalid request data"),
        @ApiResponse(responseCode = "409", description = "Product already exists")
    })
    public ProductResponse createProduct(@Valid @RequestBody CreateProductRequest request) {
        Product product = productService.createProduct(request);
        return mapper.toResponse(product);
    }
    
    @GetMapping("/{id}")
    @Operation(summary = "Get product by ID")
    public ProductResponse getProduct(
            @PathVariable @Positive Long id,
            @RequestParam(defaultValue = "false") boolean includeInventory) {
        
        Product product = includeInventory 
            ? productService.findByIdWithInventory(id)
            : productService.findById(id);
            
        return mapper.toResponse(product);
    }
    
    @GetMapping
    @Operation(summary = "List products with pagination")
    public Page<ProductResponse> listProducts(
            @PageableDefault(size = 20, sort = "createdAt,desc") Pageable pageable,
            @RequestParam(required = false) String category,
            @RequestParam(required = false) @DecimalMin("0") BigDecimal minPrice,
            @RequestParam(required = false) @DecimalMax("999999") BigDecimal maxPrice) {
        
        ProductSearchCriteria criteria = ProductSearchCriteria.builder()
            .category(category)
            .minPrice(minPrice)
            .maxPrice(maxPrice)
            .build();
            
        return productService.searchProducts(criteria, pageable)
            .map(mapper::toResponse);
    }
}
```

### 8. Testing Standards

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
    
    @Nested
    @DisplayName("Create Product")
    class CreateProduct {
        
        @Test
        @DisplayName("Should create product successfully")
        void shouldCreateProductSuccessfully() {
            // Given
            CreateProductRequest request = CreateProductRequest.builder()
                .name("Test Product")
                .price(new BigDecimal("99.99"))
                .sku("TEST-001")
                .build();
                
            Product savedProduct = Product.builder()
                .id(1L)
                .name(request.getName())
                .price(request.getPrice())
                .sku(request.getSku())
                .build();
                
            when(productRepository.existsBySku(request.getSku())).thenReturn(false);
            when(productRepository.save(any(Product.class))).thenReturn(savedProduct);
            
            // When
            Product result = productService.createProduct(request);
            
            // Then
            assertThat(result).isNotNull();
            assertThat(result.getId()).isEqualTo(1L);
            assertThat(result.getName()).isEqualTo("Test Product");
            
            verify(eventPublisher).publish(any(ProductCreatedEvent.class));
        }
        
        @Test
        @DisplayName("Should throw exception when SKU already exists")
        void shouldThrowExceptionWhenSkuExists() {
            // Given
            CreateProductRequest request = CreateProductRequest.builder()
                .sku("EXISTING-SKU")
                .build();
                
            when(productRepository.existsBySku(request.getSku())).thenReturn(true);
            
            // When/Then
            assertThatThrownBy(() -> productService.createProduct(request))
                .isInstanceOf(DuplicateProductException.class)
                .hasMessageContaining("EXISTING-SKU");
                
            verify(productRepository, never()).save(any());
            verify(eventPublisher, never()).publish(any());
        }
    }
}
```

#### Integration Tests
```java
@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles("test")
@Sql(scripts = "/sql/test-data.sql", executionPhase = BEFORE_TEST_METHOD)
@Sql(scripts = "/sql/cleanup.sql", executionPhase = AFTER_TEST_METHOD)
class ProductControllerIntegrationTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    @Test
    @DisplayName("POST /api/v1/products - Should create product")
    void shouldCreateProduct() throws Exception {
        // Given
        CreateProductRequest request = CreateProductRequest.builder()
            .name("Integration Test Product")
            .price(new BigDecimal("149.99"))
            .sku("INT-TEST-001")
            .categoryId(1L)
            .build();
            
        // When & Then
        mockMvc.perform(post("/api/v1/products")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").exists())
            .andExpect(jsonPath("$.name").value("Integration Test Product"))
            .andExpect(jsonPath("$.price").value(149.99))
            .andExpect(jsonPath("$.sku").value("INT-TEST-001"));
    }
}
```

### 9. Database Access

#### JPA Entities
```java
@Entity
@Table(name = "products", indexes = {
    @Index(name = "idx_product_sku", columnList = "sku", unique = true),
    @Index(name = "idx_product_category", columnList = "category_id"),
    @Index(name = "idx_product_active_created", columnList = "is_active,created_at")
})
@EntityListeners(AuditingEntityListener.class)
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Product {
    
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "product_seq")
    @SequenceGenerator(name = "product_seq", sequenceName = "product_sequence", allocationSize = 1)
    private Long id;
    
    @Column(nullable = false, length = 200)
    private String name;
    
    @Column(unique = true, nullable = false, length = 50)
    private String sku;
    
    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal price;
    
    @Column(columnDefinition = "TEXT")
    private String description;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id")
    private Category category;
    
    @Column(name = "is_active", nullable = false)
    private boolean active = true;
    
    @CreatedDate
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;
    
    @LastModifiedDate
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
    
    @Version
    private Long version;
}
```

#### Repository Patterns
```java
@Repository
public interface ProductRepository extends JpaRepository<Product, Long>, ProductRepositoryCustom {
    
    Optional<Product> findBySku(String sku);
    
    boolean existsBySku(String sku);
    
    @Query("SELECT p FROM Product p WHERE p.category.id = :categoryId AND p.active = true")
    Page<Product> findByCategoryAndActive(@Param("categoryId") Long categoryId, Pageable pageable);
    
    @Modifying
    @Query("UPDATE Product p SET p.active = false WHERE p.id = :id")
    void deactivateProduct(@Param("id") Long id);
}

// Custom repository for complex queries
public interface ProductRepositoryCustom {
    Page<Product> searchProducts(ProductSearchCriteria criteria, Pageable pageable);
}

@Repository
@RequiredArgsConstructor
public class ProductRepositoryImpl implements ProductRepositoryCustom {
    
    private final EntityManager entityManager;
    
    @Override
    public Page<Product> searchProducts(ProductSearchCriteria criteria, Pageable pageable) {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<Product> query = cb.createQuery(Product.class);
        Root<Product> root = query.from(Product.class);
        
        List<Predicate> predicates = new ArrayList<>();
        
        if (criteria.getCategory() != null) {
            predicates.add(cb.equal(root.get("category").get("name"), criteria.getCategory()));
        }
        
        if (criteria.getMinPrice() != null) {
            predicates.add(cb.greaterThanOrEqualTo(root.get("price"), criteria.getMinPrice()));
        }
        
        if (criteria.getMaxPrice() != null) {
            predicates.add(cb.lessThanOrEqualTo(root.get("price"), criteria.getMaxPrice()));
        }
        
        query.where(predicates.toArray(new Predicate[0]));
        
        // Apply sorting
        Sort.Order order = pageable.getSort().iterator().next();
        if (order.isAscending()) {
            query.orderBy(cb.asc(root.get(order.getProperty())));
        } else {
            query.orderBy(cb.desc(root.get(order.getProperty())));
        }
        
        TypedQuery<Product> typedQuery = entityManager.createQuery(query);
        typedQuery.setFirstResult((int) pageable.getOffset());
        typedQuery.setMaxResults(pageable.getPageSize());
        
        List<Product> results = typedQuery.getResultList();
        long total = getTotalCount(criteria);
        
        return new PageImpl<>(results, pageable, total);
    }
}
```

### 10. Logging Standards

```java
@Slf4j
@Service
public class OrderService {
    
    private static final String ORDER_CREATED = "Order created successfully. OrderId: {}, CustomerId: {}, Total: {}";
    private static final String ORDER_PROCESSING_FAILED = "Failed to process order. OrderId: {}, Error: {}";
    
    public Order createOrder(CreateOrderRequest request) {
        log.debug("Creating order for customer: {}", request.getCustomerId());
        
        try {
            // Validate
            validateOrder(request);
            
            // Process
            Order order = processOrder(request);
            
            log.info(ORDER_CREATED, order.getId(), order.getCustomerId(), order.getTotalAmount());
            
            // Log with MDC for distributed tracing
            MDC.put("orderId", order.getId().toString());
            MDC.put("customerId", order.getCustomerId().toString());
            
            return order;
            
        } catch (ValidationException e) {
            log.warn("Order validation failed. CustomerId: {}, Reason: {}", 
                request.getCustomerId(), e.getMessage());
            throw e;
        } catch (Exception e) {
            log.error(ORDER_PROCESSING_FAILED, request.getCustomerId(), e.getMessage(), e);
            throw new OrderProcessingException("Failed to create order", e);
        } finally {
            MDC.clear();
        }
    }
    
    // Use structured logging
    public void updateInventory(Long productId, int quantity) {
        log.info("Updating inventory", 
            kv("productId", productId),
            kv("quantity", quantity),
            kv("operation", "UPDATE"));
    }
}
```

## Code Quality Tools

### 1. Checkstyle Configuration

```xml
<!-- checkstyle.xml -->
<!DOCTYPE module PUBLIC
    "-//Checkstyle//DTD Checkstyle Configuration 1.3//EN"
    "https://checkstyle.org/dtds/configuration_1_3.dtd">

<module name="Checker">
    <property name="severity" value="error"/>
    <property name="fileExtensions" value="java"/>
    
    <module name="TreeWalker">
        <!-- Naming Conventions -->
        <module name="TypeName"/>
        <module name="MethodName"/>
        <module name="PackageName"/>
        <module name="LocalFinalVariableName"/>
        <module name="LocalVariableName"/>
        <module name="MemberName"/>
        <module name="ParameterName"/>
        <module name="StaticVariableName"/>
        <module name="TypeName"/>
        
        <!-- Imports -->
        <module name="AvoidStarImport"/>
        <module name="IllegalImport"/>
        <module name="RedundantImport"/>
        <module name="UnusedImports"/>
        
        <!-- Size Violations -->
        <module name="LineLength">
            <property name="max" value="120"/>
        </module>
        <module name="MethodLength">
            <property name="max" value="50"/>
        </module>
        <module name="ParameterNumber">
            <property name="max" value="5"/>
        </module>
        
        <!-- Whitespace -->
        <module name="GenericWhitespace"/>
        <module name="EmptyForIteratorPad"/>
        <module name="MethodParamPad"/>
        <module name="NoWhitespaceAfter"/>
        <module name="NoWhitespaceBefore"/>
        <module name="OperatorWrap"/>
        <module name="ParenPad"/>
        <module name="TypecastParenPad"/>
        <module name="WhitespaceAfter"/>
        <module name="WhitespaceAround"/>
        
        <!-- Coding -->
        <module name="EmptyStatement"/>
        <module name="EqualsHashCode"/>
        <module name="IllegalInstantiation"/>
        <module name="InnerAssignment"/>
        <module name="MissingSwitchDefault"/>
        <module name="MultipleVariableDeclarations"/>
        <module name="SimplifyBooleanExpression"/>
        <module name="SimplifyBooleanReturn"/>
    </module>
    
    <!-- File Length -->
    <module name="FileLength">
        <property name="max" value="500"/>
    </module>
</module>
```

### 2. PMD Configuration

```xml
<!-- pmd-ruleset.xml -->
<?xml version="1.0"?>
<ruleset name="FirstViscount PMD Rules"
         xmlns="http://pmd.sourceforge.net/ruleset/2.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://pmd.sourceforge.net/ruleset/2.0.0 
                           https://pmd.sourceforge.io/ruleset_2_0_0.xsd">
    
    <description>PMD rules for First Viscount project</description>
    
    <rule ref="category/java/bestpractices.xml">
        <exclude name="JUnitTestsShouldIncludeAssert"/>
        <exclude name="JUnitTestContainsTooManyAsserts"/>
    </rule>
    
    <rule ref="category/java/codestyle.xml">
        <exclude name="AtLeastOneConstructor"/>
        <exclude name="OnlyOneReturn"/>
    </rule>
    
    <rule ref="category/java/design.xml">
        <exclude name="LawOfDemeter"/>
    </rule>
    
    <rule ref="category/java/errorprone.xml"/>
    
    <rule ref="category/java/multithreading.xml"/>
    
    <rule ref="category/java/performance.xml"/>
    
    <rule ref="category/java/security.xml"/>
</ruleset>
```

### 3. SpotBugs Configuration

```xml
<!-- spotbugs-exclude.xml -->
<FindBugsFilter>
    <Match>
        <Class name="~.*\.model\..*"/>
        <Bug pattern="EI_EXPOSE_REP,EI_EXPOSE_REP2"/>
    </Match>
    
    <Match>
        <Class name="~.*Test"/>
        <Bug pattern="DM_DEFAULT_ENCODING"/>
    </Match>
    
    <Match>
        <Package name="~com\.firstviscount\..*\.generated\..*"/>
    </Match>
</FindBugsFilter>
```

### 4. SonarQube Quality Gates

```yaml
# sonar-project.properties
sonar.projectKey=firstviscount
sonar.organization=firstviscount
sonar.sources=src/main/java
sonar.tests=src/test/java
sonar.java.binaries=target/classes
sonar.java.test.binaries=target/test-classes
sonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml

# Quality gate conditions
sonar.qualitygate.wait=true
```

## Git Commit Standards

### Commit Message Format
```
<type>(<scope>): <subject>

<body>

<footer>
```

### Types
- **feat**: New feature
- **fix**: Bug fix
- **docs**: Documentation changes
- **style**: Code style changes (formatting, semicolons, etc)
- **refactor**: Code refactoring
- **perf**: Performance improvements
- **test**: Adding or modifying tests
- **build**: Build system changes
- **ci**: CI configuration changes
- **chore**: Other changes that don't modify src or test files

### Examples
```bash
feat(product): add product search functionality

- Implement full-text search using PostgreSQL
- Add search filters for category and price range
- Include pagination support

Closes #123

---

fix(order): prevent duplicate order creation

Fix race condition in order creation process by adding
database-level unique constraint on order number.

Fixes #456

---

refactor(inventory): extract inventory validation logic

Move inventory validation to separate validator class
to improve testability and reusability.
```

## Code Review Checklist

- [ ] Code follows naming conventions
- [ ] Methods are small and focused (single responsibility)
- [ ] No code duplication
- [ ] Proper error handling and logging
- [ ] Unit tests cover new/modified code
- [ ] Integration tests for API endpoints
- [ ] Documentation updated (JavaDoc, README, etc.)
- [ ] No hardcoded values (use configuration)
- [ ] Security considerations addressed
- [ ] Performance impact considered
- [ ] Database queries optimized
- [ ] Transaction boundaries correct
- [ ] Thread safety considered
- [ ] Resource cleanup (try-with-resources)
- [ ] No commented-out code
- [ ] Dependencies up to date
- [ ] Code passes static analysis tools

---
*Last Updated: January 2025*
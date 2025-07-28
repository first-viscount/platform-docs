# Test Data Management

This document provides comprehensive strategies for managing test data in the First Viscount microservices platform, including test data factories, fixtures, and data generation approaches.

## Overview

Effective test data management ensures:
- Consistent and repeatable tests
- Realistic data scenarios
- Test isolation
- Performance test data generation
- GDPR compliance for test data

## Test Data Factory Pattern

### Base Test Factory

```java
// test-common/src/main/java/com/firstviscount/testing/factory/TestFactory.java
public abstract class TestFactory<T> {
    
    protected final Faker faker = new Faker();
    protected final Random random = new Random();
    
    public abstract T create();
    
    public T create(Consumer<T> customizer) {
        T instance = create();
        customizer.accept(instance);
        return instance;
    }
    
    public List<T> createList(int count) {
        return IntStream.range(0, count)
            .mapToObj(i -> create())
            .collect(Collectors.toList());
    }
    
    public List<T> createList(int count, Consumer<T> customizer) {
        return IntStream.range(0, count)
            .mapToObj(i -> create(customizer))
            .collect(Collectors.toList());
    }
    
    protected String randomEnum(Class<? extends Enum<?>> enumClass) {
        Object[] values = enumClass.getEnumConstants();
        return values[random.nextInt(values.length)].toString();
    }
    
    protected <E extends Enum<E>> E randomEnumValue(Class<E> enumClass) {
        E[] values = enumClass.getEnumConstants();
        return values[random.nextInt(values.length)];
    }
}
```

### Product Test Factory

```java
@Component
public class ProductTestFactory extends TestFactory<Product> {
    
    private final CategoryTestFactory categoryFactory;
    
    public ProductTestFactory(CategoryTestFactory categoryFactory) {
        this.categoryFactory = categoryFactory;
    }
    
    @Override
    public Product create() {
        return Product.builder()
            .id(faker.number().randomNumber())
            .sku(generateSku())
            .name(faker.commerce().productName())
            .description(faker.lorem().paragraph())
            .price(BigDecimal.valueOf(faker.number().randomDouble(2, 10, 1000)))
            .category(categoryFactory.create())
            .stockQuantity(faker.number().numberBetween(0, 1000))
            .active(faker.bool().bool())
            .attributes(generateAttributes())
            .images(generateImageUrls())
            .createdAt(faker.date().past(365, TimeUnit.DAYS).toInstant())
            .updatedAt(Instant.now())
            .build();
    }
    
    public Product createWithCategory(Category category) {
        return create(product -> product.setCategory(category));
    }
    
    public Product createInStock() {
        return create(product -> {
            product.setStockQuantity(faker.number().numberBetween(10, 1000));
            product.setActive(true);
        });
    }
    
    public Product createOutOfStock() {
        return create(product -> {
            product.setStockQuantity(0);
            product.setActive(true);
        });
    }
    
    public Product createOnSale(BigDecimal discountPercentage) {
        return create(product -> {
            BigDecimal originalPrice = product.getPrice();
            BigDecimal discount = originalPrice.multiply(discountPercentage.divide(BigDecimal.valueOf(100)));
            product.setPrice(originalPrice.subtract(discount));
            product.getAttributes().put("originalPrice", originalPrice.toString());
            product.getAttributes().put("discountPercentage", discountPercentage.toString());
        });
    }
    
    private String generateSku() {
        return String.format("SKU-%s-%04d", 
            faker.letterify("???").toUpperCase(),
            faker.number().numberBetween(1000, 9999)
        );
    }
    
    private Map<String, String> generateAttributes() {
        Map<String, String> attributes = new HashMap<>();
        attributes.put("brand", faker.company().name());
        attributes.put("weight", faker.number().randomDouble(2, 0.1, 50) + " kg");
        attributes.put("dimensions", String.format("%dx%dx%d cm",
            faker.number().numberBetween(10, 100),
            faker.number().numberBetween(10, 100),
            faker.number().numberBetween(10, 100)
        ));
        return attributes;
    }
    
    private List<String> generateImageUrls() {
        return IntStream.range(0, faker.number().numberBetween(1, 5))
            .mapToObj(i -> faker.internet().url() + "/image" + i + ".jpg")
            .collect(Collectors.toList());
    }
}
```

### Order Test Factory

```java
@Component
public class OrderTestFactory extends TestFactory<Order> {
    
    private final CustomerTestFactory customerFactory;
    private final ProductTestFactory productFactory;
    private final AddressTestFactory addressFactory;
    
    @Override
    public Order create() {
        List<OrderItem> items = generateOrderItems();
        BigDecimal totalAmount = calculateTotal(items);
        
        return Order.builder()
            .id(faker.number().randomNumber())
            .orderNumber(generateOrderNumber())
            .customerId(faker.number().randomNumber())
            .status(randomEnumValue(OrderStatus.class))
            .items(items)
            .totalAmount(totalAmount)
            .shippingAddress(addressFactory.create())
            .billingAddress(addressFactory.create())
            .paymentDetails(generatePaymentDetails())
            .shippingMethod(randomEnumValue(ShippingMethod.class))
            .notes(faker.lorem().sentence())
            .createdAt(faker.date().past(30, TimeUnit.DAYS).toInstant())
            .updatedAt(Instant.now())
            .build();
    }
    
    public Order createPending() {
        return create(order -> {
            order.setStatus(OrderStatus.PENDING);
            order.setPaymentDetails(null);
        });
    }
    
    public Order createConfirmed() {
        return create(order -> {
            order.setStatus(OrderStatus.CONFIRMED);
            order.setPaymentDetails(generateSuccessfulPayment());
            order.setConfirmedAt(Instant.now());
        });
    }
    
    public Order createWithItems(List<Product> products) {
        return create(order -> {
            List<OrderItem> items = products.stream()
                .map(product -> OrderItem.builder()
                    .productId(product.getId())
                    .productName(product.getName())
                    .quantity(faker.number().numberBetween(1, 5))
                    .unitPrice(product.getPrice())
                    .build())
                .collect(Collectors.toList());
            
            order.setItems(items);
            order.setTotalAmount(calculateTotal(items));
        });
    }
    
    private String generateOrderNumber() {
        return String.format("ORD-%s-%06d",
            LocalDate.now().format(DateTimeFormatter.ofPattern("yyyyMMdd")),
            faker.number().numberBetween(1, 999999)
        );
    }
    
    private List<OrderItem> generateOrderItems() {
        int itemCount = faker.number().numberBetween(1, 5);
        return IntStream.range(0, itemCount)
            .mapToObj(i -> {
                Product product = productFactory.createInStock();
                return OrderItem.builder()
                    .productId(product.getId())
                    .productName(product.getName())
                    .quantity(faker.number().numberBetween(1, 3))
                    .unitPrice(product.getPrice())
                    .build();
            })
            .collect(Collectors.toList());
    }
    
    private BigDecimal calculateTotal(List<OrderItem> items) {
        return items.stream()
            .map(item -> item.getUnitPrice().multiply(BigDecimal.valueOf(item.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
    
    private PaymentDetails generatePaymentDetails() {
        return PaymentDetails.builder()
            .method(randomEnumValue(PaymentMethod.class))
            .transactionId("TXN-" + faker.number().digits(10))
            .amount(faker.number().randomDouble(2, 10, 1000))
            .currency("USD")
            .status(PaymentStatus.COMPLETED)
            .processedAt(Instant.now())
            .build();
    }
    
    private PaymentDetails generateSuccessfulPayment() {
        PaymentDetails payment = generatePaymentDetails();
        payment.setStatus(PaymentStatus.COMPLETED);
        return payment;
    }
}
```

## Fluent Builder Pattern

### Product Builder

```java
public class ProductBuilder {
    
    private final Product product;
    
    private ProductBuilder() {
        this.product = new Product();
        // Set defaults
        this.product.setActive(true);
        this.product.setStockQuantity(100);
        this.product.setCreatedAt(Instant.now());
        this.product.setUpdatedAt(Instant.now());
    }
    
    public static ProductBuilder aProduct() {
        return new ProductBuilder();
    }
    
    public ProductBuilder withId(Long id) {
        product.setId(id);
        return this;
    }
    
    public ProductBuilder withSku(String sku) {
        product.setSku(sku);
        return this;
    }
    
    public ProductBuilder withName(String name) {
        product.setName(name);
        return this;
    }
    
    public ProductBuilder withPrice(BigDecimal price) {
        product.setPrice(price);
        return this;
    }
    
    public ProductBuilder withPrice(double price) {
        return withPrice(BigDecimal.valueOf(price));
    }
    
    public ProductBuilder inCategory(Category category) {
        product.setCategory(category);
        return this;
    }
    
    public ProductBuilder withStock(int quantity) {
        product.setStockQuantity(quantity);
        return this;
    }
    
    public ProductBuilder outOfStock() {
        return withStock(0);
    }
    
    public ProductBuilder inactive() {
        product.setActive(false);
        return this;
    }
    
    public ProductBuilder withAttribute(String key, String value) {
        if (product.getAttributes() == null) {
            product.setAttributes(new HashMap<>());
        }
        product.getAttributes().put(key, value);
        return this;
    }
    
    public Product build() {
        // Validate required fields
        if (product.getName() == null) {
            throw new IllegalStateException("Product name is required");
        }
        if (product.getPrice() == null) {
            throw new IllegalStateException("Product price is required");
        }
        return product;
    }
    
    // Convenience methods for common scenarios
    public Product buildElectronicsProduct() {
        return withName("Laptop")
            .withPrice(999.99)
            .inCategory(new Category("Electronics"))
            .withAttribute("brand", "TechBrand")
            .withAttribute("warranty", "2 years")
            .build();
    }
    
    public Product buildClothingProduct() {
        return withName("T-Shirt")
            .withPrice(29.99)
            .inCategory(new Category("Clothing"))
            .withAttribute("size", "M")
            .withAttribute("color", "Blue")
            .build();
    }
}
```

## Test Fixtures

### JSON Fixtures

```json
// test/resources/fixtures/products.json
{
  "electronics": {
    "laptop": {
      "name": "Premium Laptop",
      "sku": "LAPTOP-001",
      "price": 1299.99,
      "category": "Electronics",
      "stockQuantity": 50,
      "attributes": {
        "brand": "TechPro",
        "processor": "Intel i7",
        "ram": "16GB",
        "storage": "512GB SSD"
      }
    },
    "smartphone": {
      "name": "Smartphone Pro",
      "sku": "PHONE-001",
      "price": 799.99,
      "category": "Electronics",
      "stockQuantity": 100,
      "attributes": {
        "brand": "PhoneMaker",
        "screenSize": "6.5 inches",
        "storage": "128GB",
        "camera": "48MP"
      }
    }
  },
  "clothing": {
    "tshirt": {
      "name": "Cotton T-Shirt",
      "sku": "TSHIRT-001",
      "price": 19.99,
      "category": "Clothing",
      "stockQuantity": 200,
      "attributes": {
        "material": "100% Cotton",
        "sizes": ["S", "M", "L", "XL"],
        "colors": ["White", "Black", "Blue", "Red"]
      }
    }
  }
}
```

### YAML Fixtures

```yaml
# test/resources/fixtures/test-scenarios.yml
scenarios:
  happy_path_order:
    customer:
      id: 12345
      email: test@example.com
      name: John Doe
    
    products:
      - id: 1
        quantity: 2
        price: 50.00
      - id: 2
        quantity: 1
        price: 30.00
    
    shipping:
      method: STANDARD
      address:
        street: 123 Test St
        city: Seattle
        state: WA
        zipCode: "98101"
    
    payment:
      method: CREDIT_CARD
      cardNumber: "4111111111111111"
      expiryDate: "12/25"
      cvv: "123"
    
    expectedTotal: 130.00
    expectedStatus: CONFIRMED

  insufficient_inventory:
    customer:
      id: 12346
      email: test2@example.com
    
    products:
      - id: 3
        quantity: 1000  # More than available
        price: 25.00
    
    expectedError: INSUFFICIENT_INVENTORY
    expectedStatus: FAILED
```

### Fixture Loader

```java
@Component
public class FixtureLoader {
    
    private final ObjectMapper objectMapper;
    private final Yaml yaml;
    
    public FixtureLoader(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
        this.yaml = new Yaml();
    }
    
    public <T> T loadJson(String path, Class<T> type) {
        try (InputStream is = getClass().getResourceAsStream(path)) {
            return objectMapper.readValue(is, type);
        } catch (IOException e) {
            throw new RuntimeException("Failed to load fixture: " + path, e);
        }
    }
    
    public <T> T loadJson(String path, TypeReference<T> type) {
        try (InputStream is = getClass().getResourceAsStream(path)) {
            return objectMapper.readValue(is, type);
        } catch (IOException e) {
            throw new RuntimeException("Failed to load fixture: " + path, e);
        }
    }
    
    public Map<String, Object> loadYaml(String path) {
        try (InputStream is = getClass().getResourceAsStream(path)) {
            return yaml.load(is);
        } catch (Exception e) {
            throw new RuntimeException("Failed to load fixture: " + path, e);
        }
    }
    
    public <T> T loadYaml(String path, Class<T> type) {
        Map<String, Object> data = loadYaml(path);
        return objectMapper.convertValue(data, type);
    }
}
```

## Database Seeding

### Liquibase Test Data

```xml
<!-- test/resources/db/changelog/test-data/01-test-categories.xml -->
<databaseChangeLog>
    <changeSet id="test-categories" author="test" context="test">
        <insert tableName="categories">
            <column name="id" value="1"/>
            <column name="name" value="Electronics"/>
            <column name="description" value="Electronic devices and accessories"/>
        </insert>
        <insert tableName="categories">
            <column name="id" value="2"/>
            <column name="name" value="Clothing"/>
            <column name="description" value="Apparel and fashion items"/>
        </insert>
        <insert tableName="categories">
            <column name="id" value="3"/>
            <column name="name" value="Books"/>
            <column name="description" value="Books and publications"/>
        </insert>
    </changeSet>
</databaseChangeLog>
```

### SQL Test Data Scripts

```sql
-- test/resources/db/test-data/products.sql
INSERT INTO products (id, sku, name, description, price, category_id, stock_quantity, active)
VALUES 
    (1, 'LAPTOP-001', 'Premium Laptop', 'High-performance laptop', 1299.99, 1, 50, true),
    (2, 'PHONE-001', 'Smartphone Pro', 'Latest smartphone', 799.99, 1, 100, true),
    (3, 'TSHIRT-001', 'Cotton T-Shirt', 'Comfortable cotton t-shirt', 19.99, 2, 200, true),
    (4, 'BOOK-001', 'Programming Guide', 'Complete programming guide', 49.99, 3, 75, true),
    (5, 'LAPTOP-002', 'Budget Laptop', 'Affordable laptop', 599.99, 1, 30, true);

-- Test products with specific states
INSERT INTO products (id, sku, name, price, category_id, stock_quantity, active)
VALUES 
    (100, 'OUT-OF-STOCK', 'Out of Stock Product', 99.99, 1, 0, true),
    (101, 'INACTIVE-001', 'Inactive Product', 49.99, 1, 50, false),
    (102, 'HIGH-PRICE', 'Expensive Product', 9999.99, 1, 5, true);
```

### Database Seeder Service

```java
@Service
@Profile("test")
@Slf4j
public class TestDataSeeder {
    
    private final ProductRepository productRepository;
    private final CategoryRepository categoryRepository;
    private final CustomerRepository customerRepository;
    private final ProductTestFactory productFactory;
    private final CustomerTestFactory customerFactory;
    
    @EventListener(ApplicationReadyEvent.class)
    @Transactional
    public void seedDatabase() {
        if (isDatabaseEmpty()) {
            log.info("Seeding test database...");
            seedCategories();
            seedProducts();
            seedCustomers();
            seedOrders();
            log.info("Test data seeding completed");
        }
    }
    
    private boolean isDatabaseEmpty() {
        return categoryRepository.count() == 0;
    }
    
    private void seedCategories() {
        List<Category> categories = Arrays.asList(
            new Category("Electronics", "Electronic devices and accessories"),
            new Category("Clothing", "Apparel and fashion items"),
            new Category("Books", "Books and publications"),
            new Category("Home & Garden", "Home improvement and garden supplies"),
            new Category("Sports", "Sports equipment and accessories")
        );
        categoryRepository.saveAll(categories);
    }
    
    private void seedProducts() {
        List<Category> categories = categoryRepository.findAll();
        
        categories.forEach(category -> {
            // Create 20 products per category
            List<Product> products = productFactory.createList(20, 
                product -> product.setCategory(category)
            );
            productRepository.saveAll(products);
        });
        
        // Create specific test products
        createSpecificTestProducts();
    }
    
    private void createSpecificTestProducts() {
        // Out of stock product
        Product outOfStock = productFactory.createOutOfStock();
        outOfStock.setSku("TEST-OUT-OF-STOCK");
        productRepository.save(outOfStock);
        
        // Expensive product
        Product expensive = productFactory.create(p -> {
            p.setSku("TEST-EXPENSIVE");
            p.setPrice(new BigDecimal("9999.99"));
        });
        productRepository.save(expensive);
        
        // Product with many attributes
        Product detailed = productFactory.create(p -> {
            p.setSku("TEST-DETAILED");
            p.getAttributes().put("color", "Red");
            p.getAttributes().put("size", "Large");
            p.getAttributes().put("material", "Cotton");
            p.getAttributes().put("warranty", "2 years");
        });
        productRepository.save(detailed);
    }
    
    private void seedCustomers() {
        // Create test customers
        List<Customer> customers = customerFactory.createList(50);
        customerRepository.saveAll(customers);
        
        // Create specific test customer
        Customer testCustomer = Customer.builder()
            .email("test@example.com")
            .password("$2a$10$..." // bcrypt encoded "password"
            .firstName("Test")
            .lastName("User")
            .build();
        customerRepository.save(testCustomer);
    }
}
```

## Test Data Isolation

### Transaction Rollback

```java
@SpringBootTest
@Transactional
@Rollback
public class ProductServiceTest {
    
    @Autowired
    private ProductService productService;
    
    @Autowired
    private ProductTestFactory productFactory;
    
    @Test
    void testCreateProduct() {
        // Test data is automatically rolled back after test
        Product product = productFactory.create();
        Product saved = productService.save(product);
        
        assertThat(saved.getId()).isNotNull();
        // Database changes will be rolled back
    }
}
```

### Database Cleaner

```java
@Component
public class DatabaseCleaner {
    
    @Autowired
    private EntityManager entityManager;
    
    @Transactional
    public void clean() {
        entityManager.createNativeQuery("SET REFERENTIAL_INTEGRITY FALSE").executeUpdate();
        
        getTables().forEach(table -> {
            entityManager.createNativeQuery("TRUNCATE TABLE " + table).executeUpdate();
        });
        
        entityManager.createNativeQuery("SET REFERENTIAL_INTEGRITY TRUE").executeUpdate();
    }
    
    private List<String> getTables() {
        return entityManager.createNativeQuery(
            "SELECT TABLE_NAME FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA = 'PUBLIC'"
        ).getResultList();
    }
}

@TestConfiguration
public class DatabaseCleanerConfig {
    
    @Bean
    public TestExecutionListener databaseCleanerListener(DatabaseCleaner cleaner) {
        return new TestExecutionListener() {
            @Override
            public void beforeTestMethod(TestContext testContext) {
                cleaner.clean();
            }
        };
    }
}
```

### Test Containers Data Isolation

```java
@TestConfiguration
public class TestContainersConfig {
    
    @Bean
    @ServiceConnection
    public PostgreSQLContainer<?> postgresContainer() {
        return new PostgreSQLContainer<>("postgres:15-alpine")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test")
            .withInitScript("db/init-test.sql");
    }
    
    @Bean
    public DataSource dataSource(PostgreSQLContainer<?> container) {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(container.getJdbcUrl());
        config.setUsername(container.getUsername());
        config.setPassword(container.getPassword());
        config.setDriverClassName(container.getDriverClassName());
        return new HikariDataSource(config);
    }
}
```

## Edge Case Data Generation

### Boundary Value Testing

```java
public class EdgeCaseDataGenerator {
    
    public static class StringEdgeCases {
        public static final String EMPTY = "";
        public static final String SINGLE_CHAR = "a";
        public static final String MAX_LENGTH = "a".repeat(255);
        public static final String WITH_SPACES = "  test  ";
        public static final String SPECIAL_CHARS = "!@#$%^&*()_+-=[]{}|;':\",./<>?";
        public static final String UNICODE = "Hello 世界 🌍 مرحبا мир";
        public static final String SQL_INJECTION = "'; DROP TABLE users; --";
        public static final String XSS_ATTEMPT = "<script>alert('XSS')</script>";
        public static final String NULL_CHAR = "test\0test";
    }
    
    public static class NumericEdgeCases {
        public static final BigDecimal ZERO = BigDecimal.ZERO;
        public static final BigDecimal NEGATIVE = new BigDecimal("-1");
        public static final BigDecimal MIN_POSITIVE = new BigDecimal("0.01");
        public static final BigDecimal MAX_VALUE = new BigDecimal("999999999.99");
        public static final BigDecimal PRECISION_TEST = new BigDecimal("123.456789");
    }
    
    public static class DateEdgeCases {
        public static final Instant FAR_PAST = Instant.parse("1900-01-01T00:00:00Z");
        public static final Instant FAR_FUTURE = Instant.parse("2100-12-31T23:59:59Z");
        public static final Instant EPOCH = Instant.EPOCH;
        public static final Instant NOW = Instant.now();
        public static final Instant LEAP_YEAR = Instant.parse("2024-02-29T00:00:00Z");
    }
    
    public static Product createProductWithEdgeCases() {
        return ProductBuilder.aProduct()
            .withName(StringEdgeCases.MAX_LENGTH)
            .withPrice(NumericEdgeCases.MIN_POSITIVE)
            .withStock(Integer.MAX_VALUE)
            .build();
    }
}
```

### Combinatorial Testing

```java
@ParameterizedTest
@MethodSource("productCombinations")
void testProductValidation(String name, BigDecimal price, Integer stock, boolean expectedValid) {
    Product product = new Product();
    product.setName(name);
    product.setPrice(price);
    product.setStockQuantity(stock);
    
    Set<ConstraintViolation<Product>> violations = validator.validate(product);
    
    if (expectedValid) {
        assertThat(violations).isEmpty();
    } else {
        assertThat(violations).isNotEmpty();
    }
}

static Stream<Arguments> productCombinations() {
    return Stream.of(
        // name, price, stock, expectedValid
        Arguments.of("Valid Product", new BigDecimal("10.00"), 100, true),
        Arguments.of("", new BigDecimal("10.00"), 100, false),
        Arguments.of("Valid Product", new BigDecimal("-10.00"), 100, false),
        Arguments.of("Valid Product", new BigDecimal("10.00"), -1, false),
        Arguments.of(null, new BigDecimal("10.00"), 100, false),
        Arguments.of("Valid Product", null, 100, false),
        Arguments.of("Valid Product", new BigDecimal("10.00"), null, false)
    );
}
```

## Performance Test Data Generation

### Bulk Data Generator

```java
@Component
public class BulkDataGenerator {
    
    private final JdbcTemplate jdbcTemplate;
    private final Faker faker = new Faker();
    
    public void generateProducts(int count) {
        String sql = "INSERT INTO products (sku, name, price, category_id, stock_quantity) VALUES (?, ?, ?, ?, ?)";
        
        jdbcTemplate.batchUpdate(sql, new BatchPreparedStatementSetter() {
            @Override
            public void setValues(PreparedStatement ps, int i) throws SQLException {
                ps.setString(1, "SKU-" + UUID.randomUUID());
                ps.setString(2, faker.commerce().productName());
                ps.setBigDecimal(3, BigDecimal.valueOf(faker.number().randomDouble(2, 10, 1000)));
                ps.setLong(4, faker.number().numberBetween(1, 5));
                ps.setInt(5, faker.number().numberBetween(0, 1000));
            }
            
            @Override
            public int getBatchSize() {
                return count;
            }
        });
    }
    
    public void generateOrders(int customerCount, int ordersPerCustomer) {
        List<Long> productIds = jdbcTemplate.queryForList(
            "SELECT id FROM products ORDER BY RANDOM() LIMIT 100", 
            Long.class
        );
        
        String orderSql = "INSERT INTO orders (order_number, customer_id, total_amount, status) VALUES (?, ?, ?, ?)";
        String itemSql = "INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES (?, ?, ?, ?)";
        
        for (int customerId = 1; customerId <= customerCount; customerId++) {
            for (int orderNum = 0; orderNum < ordersPerCustomer; orderNum++) {
                String orderNumber = generateOrderNumber();
                BigDecimal totalAmount = BigDecimal.ZERO;
                
                // Insert order
                KeyHolder keyHolder = new GeneratedKeyHolder();
                jdbcTemplate.update(connection -> {
                    PreparedStatement ps = connection.prepareStatement(orderSql, Statement.RETURN_GENERATED_KEYS);
                    ps.setString(1, orderNumber);
                    ps.setLong(2, customerId);
                    ps.setBigDecimal(3, totalAmount);
                    ps.setString(4, "PENDING");
                    return ps;
                }, keyHolder);
                
                Long orderId = keyHolder.getKey().longValue();
                
                // Insert order items
                int itemCount = faker.number().numberBetween(1, 5);
                for (int i = 0; i < itemCount; i++) {
                    Long productId = productIds.get(faker.number().numberBetween(0, productIds.size()));
                    int quantity = faker.number().numberBetween(1, 3);
                    BigDecimal unitPrice = BigDecimal.valueOf(faker.number().randomDouble(2, 10, 500));
                    
                    jdbcTemplate.update(itemSql, orderId, productId, quantity, unitPrice);
                    totalAmount = totalAmount.add(unitPrice.multiply(BigDecimal.valueOf(quantity)));
                }
                
                // Update order total
                jdbcTemplate.update("UPDATE orders SET total_amount = ? WHERE id = ?", totalAmount, orderId);
            }
        }
    }
    
    private String generateOrderNumber() {
        return String.format("ORD-%s-%06d",
            LocalDate.now().format(DateTimeFormatter.BASIC_ISO_DATE),
            faker.number().numberBetween(1, 999999)
        );
    }
}
```

### CSV Data Generator

```java
@Component
public class CsvDataGenerator {
    
    private final CSVPrinter csvPrinter;
    
    public void generateProductsCsv(String filename, int count) throws IOException {
        try (Writer writer = Files.newBufferedWriter(Paths.get(filename));
             CSVPrinter csvPrinter = new CSVPrinter(writer, CSVFormat.DEFAULT
                 .withHeader("sku", "name", "price", "category", "stock"))) {
            
            Faker faker = new Faker();
            for (int i = 0; i < count; i++) {
                csvPrinter.printRecord(
                    "SKU-" + faker.number().digits(10),
                    faker.commerce().productName(),
                    faker.number().randomDouble(2, 10, 1000),
                    faker.commerce().department(),
                    faker.number().numberBetween(0, 1000)
                );
            }
        }
    }
    
    public void generateCustomersCsv(String filename, int count) throws IOException {
        try (Writer writer = Files.newBufferedWriter(Paths.get(filename));
             CSVPrinter csvPrinter = new CSVPrinter(writer, CSVFormat.DEFAULT
                 .withHeader("email", "firstName", "lastName", "phone", "address"))) {
            
            Faker faker = new Faker();
            for (int i = 0; i < count; i++) {
                csvPrinter.printRecord(
                    faker.internet().emailAddress(),
                    faker.name().firstName(),
                    faker.name().lastName(),
                    faker.phoneNumber().phoneNumber(),
                    faker.address().fullAddress()
                );
            }
        }
    }
}
```

## Test Data Management Best Practices

### 1. Data Privacy and GDPR Compliance

```java
@Component
public class TestDataAnonymizer {
    
    private final Faker faker = new Faker();
    
    public void anonymizeCustomerData() {
        String sql = """
            UPDATE customers SET
                email = CONCAT('user', id, '@example.com'),
                first_name = ?,
                last_name = ?,
                phone = ?,
                address = ?
            WHERE id = ?
            """;
        
        List<Customer> customers = customerRepository.findAll();
        
        jdbcTemplate.batchUpdate(sql, new BatchPreparedStatementSetter() {
            @Override
            public void setValues(PreparedStatement ps, int i) throws SQLException {
                ps.setString(1, faker.name().firstName());
                ps.setString(2, faker.name().lastName());
                ps.setString(3, faker.phoneNumber().phoneNumber());
                ps.setString(4, faker.address().fullAddress());
                ps.setLong(5, customers.get(i).getId());
            }
            
            @Override
            public int getBatchSize() {
                return customers.size();
            }
        });
    }
}
```

### 2. Test Data Cleanup

```java
@Component
public class TestDataCleanupService {
    
    @Scheduled(cron = "0 0 2 * * ?") // Run at 2 AM daily
    public void cleanupOldTestData() {
        // Delete test orders older than 30 days
        jdbcTemplate.update(
            "DELETE FROM orders WHERE created_at < ? AND order_number LIKE 'TEST-%'",
            Instant.now().minus(30, ChronoUnit.DAYS)
        );
        
        // Delete orphaned order items
        jdbcTemplate.update(
            "DELETE FROM order_items WHERE order_id NOT IN (SELECT id FROM orders)"
        );
        
        // Archive old test data
        archiveOldTestData();
    }
    
    private void archiveOldTestData() {
        // Move old test data to archive tables
        jdbcTemplate.update("""
            INSERT INTO archived_orders 
            SELECT * FROM orders 
            WHERE created_at < ? AND order_number LIKE 'TEST-%'
            """, Instant.now().minus(90, ChronoUnit.DAYS));
    }
}
```

### 3. Test Data Versioning

```java
@Entity
@Table(name = "test_data_versions")
public class TestDataVersion {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String version;
    private String description;
    private Instant appliedAt;
    private String appliedBy;
    
    // Getters and setters
}

@Component
public class TestDataVersionManager {
    
    public void applyTestDataSet(String version, Runnable dataSetup) {
        if (!isVersionApplied(version)) {
            dataSetup.run();
            recordVersion(version);
        }
    }
    
    private boolean isVersionApplied(String version) {
        return testDataVersionRepository.existsByVersion(version);
    }
    
    private void recordVersion(String version) {
        TestDataVersion tdv = new TestDataVersion();
        tdv.setVersion(version);
        tdv.setAppliedAt(Instant.now());
        tdv.setAppliedBy(System.getProperty("user.name"));
        testDataVersionRepository.save(tdv);
    }
}
```

## Best Practices

1. **Use Factories**: Centralize test data creation logic
2. **Isolation**: Each test should have its own data
3. **Repeatability**: Tests should produce same results
4. **Performance**: Use bulk operations for large datasets
5. **Cleanup**: Always clean up test data
6. **Realistic Data**: Use realistic values and scenarios
7. **Edge Cases**: Test boundary conditions
8. **Documentation**: Document test data requirements

---
*Last Updated: January 2025*
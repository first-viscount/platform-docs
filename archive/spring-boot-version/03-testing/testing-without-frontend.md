# Testing Without Frontend

This document provides comprehensive strategies and tools for testing the First Viscount backend services without requiring a frontend application.

## Overview

Testing backend services independently allows for:
- Faster development cycles
- Earlier bug detection
- Parallel frontend/backend development
- Better API contract validation
- Automated regression testing

## Testing Tools and Approaches

### 1. REST API Testing Tools

#### Postman/Newman

**Interactive Testing with Postman**
```javascript
// Postman Collection for Product Service
{
  "info": {
    "name": "Product Catalog API",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
  },
  "item": [
    {
      "name": "Create Product",
      "request": {
        "method": "POST",
        "header": [
          {
            "key": "Content-Type",
            "value": "application/json"
          },
          {
            "key": "Authorization",
            "value": "Bearer {{access_token}}"
          }
        ],
        "body": {
          "mode": "raw",
          "raw": "{\n  \"name\": \"{{productName}}\",\n  \"price\": {{price}},\n  \"categoryId\": {{categoryId}}\n}"
        },
        "url": {
          "raw": "{{baseUrl}}/api/v1/products",
          "host": ["{{baseUrl}}"],
          "path": ["api", "v1", "products"]
        }
      },
      "event": [
        {
          "listen": "test",
          "script": {
            "exec": [
              "pm.test(\"Status code is 201\", function () {",
              "    pm.response.to.have.status(201);",
              "});",
              "",
              "pm.test(\"Response has product ID\", function () {",
              "    var jsonData = pm.response.json();",
              "    pm.expect(jsonData).to.have.property('id');",
              "    pm.environment.set(\"productId\", jsonData.id);",
              "});"
            ]
          }
        }
      ]
    }
  ]
}
```

**Automated Testing with Newman**
```bash
# Run Postman collection from CLI
newman run product-catalog-api.json \
  -e dev-environment.json \
  --reporters cli,junit \
  --reporter-junit-export results.xml

# Run with data file for parameterized testing
newman run product-catalog-api.json \
  -d test-data.csv \
  -n 100 \
  --delay-request 100
```

#### HTTPie

**Command-line Testing**
```bash
# Simple GET request
http GET localhost:8082/api/v1/products

# POST with JSON data
http POST localhost:8082/api/v1/products \
  name="Premium Coffee Maker" \
  price=299.99 \
  categoryId=5 \
  Authorization:"Bearer $TOKEN"

# PUT with file input
http PUT localhost:8082/api/v1/products/123 < product-update.json

# Complex query parameters
http GET localhost:8082/api/v1/products \
  category==electronics \
  'price[gte]'==100 \
  'price[lte]'==500 \
  sort==-price
```

#### cURL Scripts

**Automated Test Scripts**
```bash
#!/bin/bash
# test-product-api.sh

BASE_URL="http://localhost:8082/api/v1"
TOKEN="your-jwt-token"

# Function to test endpoint
test_endpoint() {
    local method=$1
    local endpoint=$2
    local data=$3
    local expected_status=$4
    
    response=$(curl -s -w "\n%{http_code}" -X $method \
        -H "Authorization: Bearer $TOKEN" \
        -H "Content-Type: application/json" \
        -d "$data" \
        "$BASE_URL$endpoint")
    
    status=$(echo "$response" | tail -n 1)
    body=$(echo "$response" | sed '$d')
    
    if [ "$status" -eq "$expected_status" ]; then
        echo "✅ $method $endpoint - Status: $status"
        echo "Response: $body"
    else
        echo "❌ $method $endpoint - Expected: $expected_status, Got: $status"
        echo "Response: $body"
        exit 1
    fi
}

# Test cases
echo "Testing Product API..."

# Create product
test_endpoint "POST" "/products" \
    '{"name":"Test Product","price":99.99,"categoryId":1}' \
    201

# Get product
test_endpoint "GET" "/products/1" "" 200

# Update product
test_endpoint "PUT" "/products/1" \
    '{"name":"Updated Product","price":149.99}' \
    200
```

### 2. Spring Boot Test Framework

#### MockMvc for REST Testing

```java
@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles("test")
public class ProductControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Autowired
    private ProductRepository productRepository;
    
    @MockBean
    private InventoryClient inventoryClient;
    
    @Test
    @DisplayName("Should create product successfully")
    void createProduct() throws Exception {
        // Given
        CreateProductRequest request = CreateProductRequest.builder()
            .name("Test Product")
            .price(new BigDecimal("99.99"))
            .categoryId(1L)
            .build();
            
        when(inventoryClient.checkAvailability(anyLong()))
            .thenReturn(true);
        
        // When & Then
        mockMvc.perform(post("/api/v1/products")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request))
                .header("Authorization", "Bearer " + getTestToken()))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").exists())
            .andExpect(jsonPath("$.name").value("Test Product"))
            .andExpect(jsonPath("$.price").value(99.99))
            .andDo(document("create-product",
                requestHeaders(
                    headerWithName("Authorization").description("JWT Bearer token")
                ),
                requestFields(
                    fieldWithPath("name").description("Product name"),
                    fieldWithPath("price").description("Product price"),
                    fieldWithPath("categoryId").description("Category ID")
                ),
                responseFields(
                    fieldWithPath("id").description("Product ID"),
                    fieldWithPath("name").description("Product name"),
                    fieldWithPath("price").description("Product price")
                )
            ));
        
        // Verify
        assertThat(productRepository.count()).isEqualTo(1);
    }
    
    @Test
    @DisplayName("Should return paginated products")
    void listProducts() throws Exception {
        // Given
        IntStream.range(0, 25).forEach(i -> 
            productRepository.save(Product.builder()
                .name("Product " + i)
                .price(new BigDecimal(i * 10))
                .build())
        );
        
        // When & Then
        mockMvc.perform(get("/api/v1/products")
                .param("page", "0")
                .param("size", "10")
                .param("sort", "price,desc"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.content").isArray())
            .andExpect(jsonPath("$.content.length()").value(10))
            .andExpect(jsonPath("$.totalElements").value(25))
            .andExpect(jsonPath("$.totalPages").value(3))
            .andExpect(jsonPath("$.content[0].price").value(240));
    }
}
```

#### WebTestClient for Reactive Testing

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureWebTestClient
public class ProductControllerReactiveTest {
    
    @Autowired
    private WebTestClient webTestClient;
    
    @Test
    void testProductStream() {
        webTestClient.get()
            .uri("/api/v1/products/stream")
            .accept(MediaType.TEXT_EVENT_STREAM)
            .exchange()
            .expectStatus().isOk()
            .expectHeader().contentType(MediaType.TEXT_EVENT_STREAM)
            .expectBodyList(Product.class)
            .hasSize(10)
            .consumeWith(response -> {
                List<Product> products = response.getResponseBody();
                assertThat(products).allMatch(p -> p.getPrice().compareTo(BigDecimal.ZERO) > 0);
            });
    }
}
```

### 3. REST Assured Testing

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class ProductApiTest {
    
    @LocalServerPort
    private int port;
    
    @BeforeEach
    void setUp() {
        RestAssured.port = port;
        RestAssured.basePath = "/api/v1";
        RestAssured.enableLoggingOfRequestAndResponseIfValidationFails();
    }
    
    @Test
    void testCompleteProductWorkflow() {
        // Create product
        String productId = given()
            .contentType(ContentType.JSON)
            .body(new CreateProductRequest("Test Product", new BigDecimal("99.99"), 1L))
            .when()
            .post("/products")
            .then()
            .statusCode(201)
            .body("name", equalTo("Test Product"))
            .body("price", equalTo(99.99f))
            .extract()
            .path("id");
        
        // Get product
        given()
            .pathParam("id", productId)
            .when()
            .get("/products/{id}")
            .then()
            .statusCode(200)
            .body("id", equalTo(Integer.parseInt(productId)))
            .body("name", equalTo("Test Product"));
        
        // Update product
        given()
            .pathParam("id", productId)
            .contentType(ContentType.JSON)
            .body(Map.of("price", 149.99))
            .when()
            .patch("/products/{id}")
            .then()
            .statusCode(200)
            .body("price", equalTo(149.99f));
        
        // Delete product
        given()
            .pathParam("id", productId)
            .when()
            .delete("/products/{id}")
            .then()
            .statusCode(204);
    }
    
    @Test
    void testProductSearch() {
        given()
            .queryParam("q", "coffee")
            .queryParam("minPrice", 50)
            .queryParam("maxPrice", 200)
            .when()
            .get("/products/search")
            .then()
            .statusCode(200)
            .body("content", hasSize(greaterThan(0)))
            .body("content[0].name", containsString("coffee"))
            .body("content[0].price", both(greaterThanOrEqualTo(50f)).and(lessThanOrEqualTo(200f)));
    }
}
```

### 4. API Documentation Testing

#### Swagger UI

```yaml
# application.yml
springdoc:
  api-docs:
    path: /api-docs
  swagger-ui:
    path: /swagger-ui.html
    operationsSorter: method
    tagsSorter: alpha
    tryItOutEnabled: true
    filter: true
    deepLinking: true
```

Access Swagger UI at: `http://localhost:8082/swagger-ui.html`

#### Spring REST Docs

```java
@ExtendWith(RestDocumentationExtension.class)
@AutoConfigureMockMvc
public class ApiDocumentationTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Test
    void documentProductApi() throws Exception {
        mockMvc.perform(get("/api/v1/products/{id}", 1)
                .accept(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andDo(document("get-product",
                pathParameters(
                    parameterWithName("id").description("Product ID")
                ),
                responseFields(
                    fieldWithPath("id").description("Product ID"),
                    fieldWithPath("name").description("Product name"),
                    fieldWithPath("price").description("Product price"),
                    fieldWithPath("categoryId").description("Category ID"),
                    fieldWithPath("createdAt").description("Creation timestamp"),
                    fieldWithPath("updatedAt").description("Last update timestamp")
                ),
                links(
                    linkWithRel("self").description("Link to this product"),
                    linkWithRel("category").description("Link to product category")
                )
            ));
    }
}
```

### 5. Contract Testing

#### Spring Cloud Contract

**Contract Definition**
```groovy
// contracts/products/shouldReturnProduct.groovy
Contract.make {
    description "should return product by ID"
    request {
        method GET()
        url("/api/v1/products/1")
        headers {
            accept(applicationJson())
        }
    }
    response {
        status OK()
        headers {
            contentType(applicationJson())
        }
        body([
            id: 1,
            name: "Premium Coffee Maker",
            price: 299.99,
            categoryId: 5
        ])
    }
}
```

**Generated Test**
```java
@SpringBootTest
@AutoConfigureMockMvc
public class ContractVerifierTest extends ProductContractBase {
    
    @Test
    public void validate_shouldReturnProduct() throws Exception {
        // given:
        MockMvcRequestSpecification request = given();
        
        // when:
        ResponseOptions response = given().spec(request)
            .get("/api/v1/products/1");
        
        // then:
        assertThat(response.statusCode()).isEqualTo(200);
        DocumentContext parsedJson = JsonPath.parse(response.getBody().asString());
        assertThat(parsedJson.read("$.id", Integer.class)).isEqualTo(1);
        assertThat(parsedJson.read("$.name", String.class)).isEqualTo("Premium Coffee Maker");
    }
}
```

### 6. Event-Driven Testing

#### Kafka Testing

```java
@SpringBootTest
@EmbeddedKafka(
    partitions = 1,
    topics = {"product-events", "order-events"},
    brokerProperties = {
        "listeners=PLAINTEXT://localhost:9092",
        "port=9092"
    }
)
public class EventDrivenTest {
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    @Autowired
    private ProductEventHandler productEventHandler;
    
    @Test
    void testProductCreatedEvent() {
        // Given
        ProductCreatedEvent event = ProductCreatedEvent.builder()
            .productId(123L)
            .name("Test Product")
            .price(new BigDecimal("99.99"))
            .timestamp(Instant.now())
            .build();
        
        // When
        kafkaTemplate.send("product-events", event.getProductId().toString(), event);
        
        // Then
        await().atMost(5, TimeUnit.SECONDS).untilAsserted(() -> {
            verify(productEventHandler).handleProductCreated(any(ProductCreatedEvent.class));
        });
    }
}
```

### 7. GraphQL Testing

```java
@SpringBootTest
@AutoConfigureMockMvc
public class GraphQLTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Test
    void testProductQuery() throws Exception {
        String query = """
            {
                product(id: 1) {
                    id
                    name
                    price
                    category {
                        name
                    }
                }
            }
            """;
        
        mockMvc.perform(post("/graphql")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"query\": \"" + query.replace("\n", " ") + "\"}"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.data.product.id").value(1))
            .andExpect(jsonPath("$.data.product.name").exists())
            .andExpect(jsonPath("$.data.product.category.name").exists());
    }
}
```

## Test Data Management

### Test Fixtures

```java
public class ProductFixtures {
    
    public static Product createProduct() {
        return Product.builder()
            .name("Test Product " + UUID.randomUUID())
            .price(BigDecimal.valueOf(99.99))
            .categoryId(1L)
            .active(true)
            .build();
    }
    
    public static CreateProductRequest createProductRequest() {
        return CreateProductRequest.builder()
            .name("Test Product " + UUID.randomUUID())
            .price(BigDecimal.valueOf(99.99))
            .categoryId(1L)
            .build();
    }
    
    public static List<Product> createProducts(int count) {
        return IntStream.range(0, count)
            .mapToObj(i -> Product.builder()
                .name("Product " + i)
                .price(BigDecimal.valueOf(10 + i * 5))
                .categoryId((long) (i % 5 + 1))
                .build())
            .collect(Collectors.toList());
    }
}
```

### Database Test Data

```sql
-- test-data.sql
INSERT INTO categories (id, name) VALUES 
    (1, 'Electronics'),
    (2, 'Clothing'),
    (3, 'Food'),
    (4, 'Books'),
    (5, 'Home');

INSERT INTO products (name, price, category_id, active) VALUES
    ('Laptop', 999.99, 1, true),
    ('T-Shirt', 29.99, 2, true),
    ('Coffee', 14.99, 3, true),
    ('Programming Book', 49.99, 4, true),
    ('Desk Lamp', 39.99, 5, true);
```

## Automated Test Execution

### Maven Configuration

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
        
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <configuration>
                <includes>
                    <include>**/*Test.java</include>
                </includes>
                <excludes>
                    <exclude>**/*IT.java</exclude>
                </excludes>
            </configuration>
        </plugin>
        
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-failsafe-plugin</artifactId>
            <configuration>
                <includes>
                    <include>**/*IT.java</include>
                </includes>
            </configuration>
        </plugin>
    </plugins>
</build>
```

### Test Execution Script

```bash
#!/bin/bash
# run-backend-tests.sh

echo "Starting backend tests..."

# Start infrastructure
docker-compose -f docker-compose-test.yml up -d

# Wait for services
./scripts/wait-for-it.sh localhost:5432 -t 30
./scripts/wait-for-it.sh localhost:9092 -t 30

# Run unit tests
echo "Running unit tests..."
mvn test

# Run integration tests
echo "Running integration tests..."
mvn verify

# Run contract tests
echo "Running contract tests..."
mvn clean test -Pcontract-tests

# Run API tests with Newman
echo "Running Postman tests..."
newman run postman/product-api.json -e postman/test-env.json

# Generate test report
mvn surefire-report:report

# Cleanup
docker-compose -f docker-compose-test.yml down

echo "Tests completed!"
```

## CI/CD Integration

### GitHub Actions

```yaml
name: Backend Tests

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      
      kafka:
        image: confluentinc/cp-kafka:latest
        ports:
          - 9092:9092
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up JDK 21
      uses: actions/setup-java@v3
      with:
        java-version: '21'
        distribution: 'temurin'
    
    - name: Cache Maven packages
      uses: actions/cache@v3
      with:
        path: ~/.m2
        key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}
    
    - name: Run tests
      run: |
        mvn clean test
        mvn verify -P integration-tests
    
    - name: Run API tests
      run: |
        npm install -g newman
        newman run postman/api-tests.json
    
    - name: Generate test report
      run: mvn surefire-report:report
    
    - name: Upload test results
      uses: actions/upload-artifact@v3
      with:
        name: test-results
        path: target/surefire-reports/
```

## Best Practices

1. **Test Independence**: Each test should be independent and not rely on other tests
2. **Test Data Isolation**: Use separate test data for each test to avoid conflicts
3. **Mock External Services**: Mock external dependencies for unit tests
4. **Use TestContainers**: For integration tests that need real databases
5. **Automate Everything**: All tests should run automatically in CI/CD
6. **Document API**: Use tools like Swagger or Spring REST Docs
7. **Version Your API**: Always version your API for backward compatibility
8. **Test Error Scenarios**: Don't just test happy paths

---
*Last Updated: January 2025*
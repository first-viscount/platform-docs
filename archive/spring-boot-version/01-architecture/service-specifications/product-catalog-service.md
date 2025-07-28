# Product Catalog Service Specification

## Overview

The Product Catalog Service manages all product-related information including products, categories, pricing, and attributes. It serves as the single source of truth for product data and provides high-performance search and filtering capabilities.

### Key Responsibilities
- Product lifecycle management (CRUD operations)
- Category hierarchy management
- Price management and history tracking
- Product search and filtering
- Inventory availability integration
- Product recommendations (future)

## API Specification

### Base URL
```
http://product-catalog:8082/api/v1
```

### Endpoints

#### List Products
```http
GET /products?page=0&size=20&sort=name,asc&category=electronics&minPrice=10&maxPrice=100
Authorization: Bearer {token}

Response:
{
  "content": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "sku": "LAPTOP-001",
      "name": "Premium Laptop Pro 15",
      "description": "High-performance laptop with 16GB RAM",
      "price": {
        "amount": 1299.99,
        "currency": "USD"
      },
      "category": {
        "id": "123e4567-e89b-12d3-a456-426614174000",
        "name": "Laptops",
        "path": "Electronics/Computers/Laptops"
      },
      "brand": "TechBrand",
      "attributes": {
        "processor": "Intel i7",
        "ram": "16GB",
        "storage": "512GB SSD",
        "display": "15.6 inch"
      },
      "images": [
        {
          "url": "https://cdn.example.com/products/laptop-001-main.jpg",
          "type": "main",
          "alt": "Premium Laptop Pro 15 - Front View"
        }
      ],
      "availability": {
        "inStock": true,
        "quantity": 45
      },
      "status": "ACTIVE",
      "createdAt": "2024-01-01T10:00:00Z",
      "updatedAt": "2024-01-15T14:30:00Z"
    }
  ],
  "totalElements": 156,
  "totalPages": 8,
  "number": 0,
  "size": 20
}
```

#### Get Product by ID
```http
GET /products/{productId}
Authorization: Bearer {token}

Response:
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "sku": "LAPTOP-001",
  "name": "Premium Laptop Pro 15",
  "description": "High-performance laptop with 16GB RAM and dedicated graphics",
  "longDescription": "The Premium Laptop Pro 15 combines power and portability...",
  "price": {
    "amount": 1299.99,
    "currency": "USD",
    "originalPrice": 1499.99,
    "discount": {
      "amount": 200.00,
      "percentage": 13.33,
      "validUntil": "2024-01-31T23:59:59Z"
    }
  },
  "priceHistory": [
    {
      "amount": 1499.99,
      "effectiveFrom": "2024-01-01T00:00:00Z",
      "effectiveTo": "2024-01-10T00:00:00Z"
    }
  ],
  "category": {
    "id": "123e4567-e89b-12d3-a456-426614174000",
    "name": "Laptops",
    "path": "Electronics/Computers/Laptops",
    "breadcrumb": [
      {"id": "aaa", "name": "Electronics"},
      {"id": "bbb", "name": "Computers"},
      {"id": "123e4567-e89b-12d3-a456-426614174000", "name": "Laptops"}
    ]
  },
  "specifications": {
    "general": {
      "brand": "TechBrand",
      "model": "Pro 15",
      "year": "2024"
    },
    "technical": {
      "processor": "Intel Core i7-12700H",
      "ram": "16GB DDR5",
      "storage": "512GB NVMe SSD",
      "graphics": "NVIDIA RTX 3060",
      "display": {
        "size": "15.6 inch",
        "resolution": "1920x1080",
        "type": "IPS",
        "refreshRate": "144Hz"
      }
    },
    "physical": {
      "weight": "1.8kg",
      "dimensions": "35.5 x 23.5 x 1.8 cm",
      "color": "Space Gray"
    }
  }
}
```

#### Create Product (Admin)
```http
POST /products
Content-Type: application/json
Authorization: Bearer {admin-token}

{
  "sku": "LAPTOP-002",
  "name": "Business Laptop Essential",
  "description": "Reliable laptop for business use",
  "categoryId": "123e4567-e89b-12d3-a456-426614174000",
  "price": {
    "amount": 799.99,
    "currency": "USD"
  },
  "brand": "TechBrand",
  "attributes": {
    "processor": "Intel i5",
    "ram": "8GB",
    "storage": "256GB SSD"
  }
}

Response:
{
  "id": "660e8400-e29b-41d4-a716-446655440000",
  "sku": "LAPTOP-002",
  "name": "Business Laptop Essential",
  "status": "DRAFT",
  "createdAt": "2024-01-15T15:00:00Z"
}
```

#### Update Product (Admin)
```http
PUT /products/{productId}
Content-Type: application/json
Authorization: Bearer {admin-token}

{
  "name": "Business Laptop Essential Plus",
  "price": {
    "amount": 849.99,
    "currency": "USD"
  },
  "status": "ACTIVE"
}

Response:
{
  "id": "660e8400-e29b-41d4-a716-446655440000",
  "sku": "LAPTOP-002",
  "name": "Business Laptop Essential Plus",
  "price": {
    "amount": 849.99,
    "currency": "USD"
  },
  "status": "ACTIVE",
  "updatedAt": "2024-01-15T15:30:00Z"
}
```

#### Search Products
```http
POST /products/search
Content-Type: application/json
Authorization: Bearer {token}

{
  "query": "gaming laptop",
  "filters": {
    "categories": ["123e4567-e89b-12d3-a456-426614174000"],
    "priceRange": {
      "min": 1000,
      "max": 2000
    },
    "brands": ["TechBrand", "GameBrand"],
    "attributes": {
      "ram": ["16GB", "32GB"],
      "graphics": ["RTX 3060", "RTX 3070"]
    }
  },
  "sort": {
    "field": "price",
    "direction": "ASC"
  },
  "page": 0,
  "size": 20
}

Response:
{
  "results": [...],
  "facets": {
    "categories": [
      {"id": "123", "name": "Gaming Laptops", "count": 45},
      {"id": "124", "name": "Professional Laptops", "count": 23}
    ],
    "brands": [
      {"name": "TechBrand", "count": 32},
      {"name": "GameBrand", "count": 28}
    ],
    "priceRanges": [
      {"range": "1000-1500", "count": 35},
      {"range": "1500-2000", "count": 25}
    ]
  },
  "totalResults": 68,
  "page": 0,
  "totalPages": 4
}
```

#### Get Categories
```http
GET /categories
Authorization: Bearer {token}

Response:
{
  "categories": [
    {
      "id": "root-1",
      "name": "Electronics",
      "slug": "electronics",
      "level": 0,
      "children": [
        {
          "id": "cat-1",
          "name": "Computers",
          "slug": "computers",
          "level": 1,
          "children": [
            {
              "id": "123e4567-e89b-12d3-a456-426614174000",
              "name": "Laptops",
              "slug": "laptops",
              "level": 2,
              "productCount": 156
            }
          ]
        }
      ]
    }
  ]
}
```

## Event Contracts

### Published Events

#### ProductCreatedEvent
```json
{
  "eventId": "770e8400-e29b-41d4-a716-446655440000",
  "productId": "550e8400-e29b-41d4-a716-446655440000",
  "sku": "LAPTOP-001",
  "name": "Premium Laptop Pro 15",
  "categoryId": "123e4567-e89b-12d3-a456-426614174000",
  "price": {
    "amount": 1299.99,
    "currency": "USD"
  },
  "status": "ACTIVE",
  "timestamp": "2024-01-15T15:00:00Z"
}
```

#### ProductUpdatedEvent
```json
{
  "eventId": "880e8400-e29b-41d4-a716-446655440000",
  "productId": "550e8400-e29b-41d4-a716-446655440000",
  "changes": {
    "price": {
      "old": 1299.99,
      "new": 1199.99
    },
    "name": {
      "old": "Premium Laptop Pro 15",
      "new": "Premium Laptop Pro 15 - Sale"
    }
  },
  "updatedBy": "admin-user-id",
  "timestamp": "2024-01-15T16:00:00Z"
}
```

#### PriceChangedEvent
```json
{
  "eventId": "990e8400-e29b-41d4-a716-446655440000",
  "productId": "550e8400-e29b-41d4-a716-446655440000",
  "sku": "LAPTOP-001",
  "oldPrice": 1299.99,
  "newPrice": 1199.99,
  "currency": "USD",
  "reason": "PROMOTION",
  "effectiveFrom": "2024-01-16T00:00:00Z",
  "effectiveTo": "2024-01-31T23:59:59Z",
  "timestamp": "2024-01-15T16:00:00Z"
}
```

#### ProductDeactivatedEvent
```json
{
  "eventId": "aa0e8400-e29b-41d4-a716-446655440000",
  "productId": "550e8400-e29b-41d4-a716-446655440000",
  "sku": "LAPTOP-001",
  "reason": "OUT_OF_STOCK",
  "deactivatedBy": "system",
  "timestamp": "2024-01-15T17:00:00Z"
}
```

### Consumed Events

- `InventoryUpdatedEvent` - Updates product availability
- `OrderCreatedEvent` - Updates product popularity metrics

## Data Model

### Database Schema

```sql
-- Products table
CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sku VARCHAR(100) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    long_description TEXT,
    category_id UUID REFERENCES categories(id),
    brand VARCHAR(100),
    status VARCHAR(50) NOT NULL DEFAULT 'DRAFT',
    attributes JSONB,
    specifications JSONB,
    seo_metadata JSONB,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    created_by UUID,
    updated_by UUID,
    version INT NOT NULL DEFAULT 1,
    
    INDEX idx_sku (sku),
    INDEX idx_category_id (category_id),
    INDEX idx_status (status),
    INDEX idx_brand (brand),
    INDEX idx_created_at (created_at)
);

-- Categories table with hierarchical structure
CREATE TABLE categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_id UUID REFERENCES categories(id),
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    description TEXT,
    path LTREE NOT NULL,
    level INT NOT NULL DEFAULT 0,
    display_order INT DEFAULT 0,
    attributes_schema JSONB,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_path ON categories USING GIST (path),
    INDEX idx_parent_id (parent_id),
    INDEX idx_slug (slug),
    INDEX idx_is_active (is_active)
);

-- Prices table for price history
CREATE TABLE prices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID NOT NULL REFERENCES products(id),
    amount DECIMAL(10,2) NOT NULL,
    currency VARCHAR(3) NOT NULL DEFAULT 'USD',
    price_type VARCHAR(50) NOT NULL DEFAULT 'REGULAR',
    effective_from TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    effective_to TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    created_by UUID,
    
    INDEX idx_product_id (product_id),
    INDEX idx_effective_dates (effective_from, effective_to),
    CONSTRAINT valid_price CHECK (amount >= 0)
);

-- Product images
CREATE TABLE product_images (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID NOT NULL REFERENCES products(id),
    url VARCHAR(500) NOT NULL,
    type VARCHAR(50) NOT NULL DEFAULT 'additional',
    display_order INT DEFAULT 0,
    alt_text VARCHAR(255),
    metadata JSONB,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_product_id (product_id),
    INDEX idx_type (type)
);

-- Product variants (future)
CREATE TABLE product_variants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID NOT NULL REFERENCES products(id),
    sku VARCHAR(100) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    attributes JSONB NOT NULL,
    price_adjustment DECIMAL(10,2) DEFAULT 0,
    stock_tracking BOOLEAN DEFAULT true,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_product_id (product_id),
    INDEX idx_sku (sku)
);

-- Full-text search
CREATE INDEX idx_product_search ON products 
USING GIN (to_tsvector('english', name || ' ' || COALESCE(description, '') || ' ' || COALESCE(brand, '')));

-- JSONB indexes for attributes
CREATE INDEX idx_product_attributes ON products USING GIN (attributes);
CREATE INDEX idx_product_specifications ON products USING GIN (specifications);
```

### Entity Models

```java
@Entity
@Table(name = "products")
@EntityListeners(AuditingEntityListener.class)
public class Product {
    @Id
    @GeneratedValue
    private UUID id;
    
    @Column(unique = true, nullable = false)
    private String sku;
    
    @Column(nullable = false)
    private String name;
    
    @Column(columnDefinition = "TEXT")
    private String description;
    
    @Column(name = "long_description", columnDefinition = "TEXT")
    private String longDescription;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id")
    private Category category;
    
    private String brand;
    
    @Enumerated(EnumType.STRING)
    private ProductStatus status = ProductStatus.DRAFT;
    
    @Type(type = "jsonb")
    @Column(columnDefinition = "jsonb")
    private Map<String, Object> attributes;
    
    @Type(type = "jsonb")
    @Column(columnDefinition = "jsonb")
    private Map<String, Object> specifications;
    
    @OneToMany(mappedBy = "product", cascade = CascadeType.ALL)
    @OrderBy("displayOrder ASC")
    private List<ProductImage> images;
    
    @OneToMany(mappedBy = "product")
    @OrderBy("effectiveFrom DESC")
    private List<Price> priceHistory;
    
    @Transient
    public Price getCurrentPrice() {
        return priceHistory.stream()
            .filter(p -> p.isEffective())
            .findFirst()
            .orElse(null);
    }
    
    @Version
    private Long version;
    
    @CreatedDate
    private Instant createdAt;
    
    @LastModifiedDate
    private Instant updatedAt;
    
    @CreatedBy
    private UUID createdBy;
    
    @LastModifiedBy
    private UUID updatedBy;
}
```

## Configuration

### Application Properties

```yaml
spring:
  application:
    name: product-catalog-service
  
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:5432/product_db
    username: ${DB_USER:product_user}
    password: ${DB_PASSWORD:password}
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
  
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        jdbc:
          batch_size: 25
        order_inserts: true
        order_updates: true
  
  redis:
    host: ${REDIS_HOST:localhost}
    port: 6379
    timeout: 2000ms
    lettuce:
      pool:
        max-active: 8
        max-idle: 8
  
  kafka:
    bootstrap-servers: ${KAFKA_BROKERS:localhost:9092}
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all
      retries: 3

cache:
  products:
    ttl: PT1H
    max-size: 10000
  categories:
    ttl: PT24H
    max-size: 1000

search:
  max-results: 1000
  default-page-size: 20
  max-page-size: 100
```

### Environment Variables

```bash
# Database
DB_HOST=postgres
DB_USER=product_user
DB_PASSWORD=secure_password

# Redis
REDIS_HOST=redis
REDIS_PASSWORD=redis_password

# Kafka
KAFKA_BROKERS=kafka1:9092,kafka2:9092,kafka3:9092

# Service URLs
INVENTORY_SERVICE_URL=http://inventory-service:8083
IMAGE_CDN_URL=https://cdn.firstviscount.com

# Security
JWT_SECRET=your-secret-key
API_KEY=service-api-key
```

## Dependencies

### External Services
- **Inventory Service** - Real-time stock availability
- **Image CDN** - Product image storage and delivery
- **Search Service** (future) - Advanced search capabilities

### Libraries
```xml
<dependencies>
    <!-- Spring Boot -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    
    <!-- PostgreSQL -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
    </dependency>
    
    <!-- Redis -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-cache</artifactId>
    </dependency>
    
    <!-- Kafka -->
    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka</artifactId>
    </dependency>
    
    <!-- Validation -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
    
    <!-- Monitoring -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>
</dependencies>
```

## Testing

### Unit Tests
```java
@Test
void shouldCreateProduct() {
    // Given
    CreateProductRequest request = CreateProductRequest.builder()
        .sku("TEST-001")
        .name("Test Product")
        .categoryId(categoryId)
        .price(new Money(99.99, "USD"))
        .build();
    
    // When
    Product product = productService.createProduct(request);
    
    // Then
    assertThat(product.getSku()).isEqualTo("TEST-001");
    assertThat(product.getStatus()).isEqualTo(ProductStatus.DRAFT);
    verify(eventPublisher).publish(any(ProductCreatedEvent.class));
}
```

### Integration Tests
```java
@SpringBootTest
@AutoConfigureMockMvc
class ProductControllerIntegrationTest {
    
    @Test
    void shouldSearchProducts() throws Exception {
        // Given products in database
        
        // When
        mockMvc.perform(post("/api/v1/products/search")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {
                        "query": "laptop",
                        "filters": {
                            "priceRange": {"min": 500, "max": 1500}
                        }
                    }
                    """))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.results").isArray())
                .andExpect(jsonPath("$.facets").exists());
    }
}
```

## Deployment

### Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-catalog-service
  namespace: ecommerce
spec:
  replicas: 3
  selector:
    matchLabels:
      app: product-catalog-service
  template:
    metadata:
      labels:
        app: product-catalog-service
    spec:
      containers:
      - name: product-catalog
        image: firstviscount/product-catalog-service:latest
        ports:
        - containerPort: 8082
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "production"
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: host
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8082
          initialDelaySeconds: 30
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8082
          initialDelaySeconds: 20
```

## Monitoring

### Health Checks

```java
@Component
public class ProductDatabaseHealthIndicator implements HealthIndicator {
    
    @Override
    public Health health() {
        try {
            long productCount = productRepository.count();
            long activeProducts = productRepository.countByStatus(ProductStatus.ACTIVE);
            
            return Health.up()
                .withDetail("total_products", productCount)
                .withDetail("active_products", activeProducts)
                .withDetail("database", "Connected")
                .build();
                
        } catch (Exception e) {
            return Health.down()
                .withException(e)
                .build();
        }
    }
}
```

### Metrics

Key metrics exposed:
- `products.created.total` - Total products created
- `products.updated.total` - Total products updated
- `products.search.duration` - Search query execution time
- `products.cache.hit.ratio` - Cache hit ratio
- `products.api.requests` - API request count by endpoint

### Alerts

```yaml
groups:
  - name: product-catalog
    rules:
      - alert: HighSearchLatency
        expr: histogram_quantile(0.95, products_search_duration_seconds) > 1
        for: 5m
        annotations:
          summary: "Product search latency is high"
          
      - alert: LowCacheHitRatio
        expr: products_cache_hit_ratio < 0.7
        for: 10m
        annotations:
          summary: "Product cache hit ratio is low"
```

## Best Practices

1. **Caching** - Cache frequently accessed products
2. **Pagination** - Always paginate list endpoints
3. **Indexing** - Create appropriate database indexes
4. **Validation** - Validate all input data
5. **Versioning** - Use optimistic locking for updates
6. **Search** - Implement faceted search for better UX
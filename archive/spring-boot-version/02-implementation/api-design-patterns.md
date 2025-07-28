# API Design Patterns

This document defines RESTful API conventions and patterns for the First Viscount platform microservices.

## RESTful API Principles

### Core REST Principles

1. **Client-Server Architecture**: Clear separation between client and server
2. **Statelessness**: Each request contains all information needed
3. **Cacheability**: Responses indicate if they can be cached
4. **Uniform Interface**: Consistent patterns across all APIs
5. **Layered System**: APIs can work through intermediaries
6. **Code on Demand** (optional): Server can extend client functionality

### Resource-Oriented Design

APIs are designed around resources (nouns), not actions (verbs):

```
✅ Good: GET /api/v1/products/123
❌ Bad:  GET /api/v1/getProduct?id=123

✅ Good: POST /api/v1/orders
❌ Bad:  POST /api/v1/createOrder

✅ Good: PUT /api/v1/inventory/456/reserve
❌ Bad:  POST /api/v1/reserveInventory
```

## URL Structure

### Base URL Pattern

```
https://{service}.firstviscount.com/api/{version}/{resource}
```

Examples:
```
https://catalog.firstviscount.com/api/v1/products
https://order.firstviscount.com/api/v1/orders
https://inventory.firstviscount.com/api/v1/stock
```

### API Versioning

Version in URL path for major versions:
```
/api/v1/products  # Version 1
/api/v2/products  # Version 2 (breaking changes)
```

### Resource Naming Conventions

1. **Use plural nouns**: `/products`, `/orders`, `/customers`
2. **Use kebab-case**: `/order-items`, `/product-categories`
3. **Avoid deep nesting**: Maximum 2 levels deep
4. **Use clear relationships**: `/orders/123/items`

### URL Examples

```
# Collection Resources
GET    /api/v1/products              # List products
POST   /api/v1/products              # Create product

# Individual Resources
GET    /api/v1/products/123          # Get product
PUT    /api/v1/products/123          # Update product
PATCH  /api/v1/products/123          # Partial update
DELETE /api/v1/products/123          # Delete product

# Nested Resources
GET    /api/v1/orders/456/items      # Order items
POST   /api/v1/orders/456/items      # Add order item

# Actions (when necessary)
POST   /api/v1/orders/456/cancel     # Cancel order
POST   /api/v1/inventory/789/reserve # Reserve inventory
```

## HTTP Methods

### Method Usage

| Method | Purpose | Idempotent | Safe | Request Body | Success Status |
|--------|---------|------------|------|--------------|----------------|
| GET | Retrieve resource(s) | Yes | Yes | No | 200 OK |
| POST | Create resource | No | No | Yes | 201 Created |
| PUT | Replace resource | Yes | No | Yes | 200 OK |
| PATCH | Update partially | No | No | Yes | 200 OK |
| DELETE | Remove resource | Yes | No | No | 204 No Content |
| HEAD | Get headers only | Yes | Yes | No | 200 OK |
| OPTIONS | Get allowed methods | Yes | Yes | No | 200 OK |

### Method Examples

#### GET - Retrieve Resources

```http
# Get single resource
GET /api/v1/products/123
Accept: application/json

Response: 200 OK
{
  "id": 123,
  "name": "Premium Coffee Maker",
  "price": 299.99,
  "currency": "USD"
}

# Get collection with filtering
GET /api/v1/products?category=electronics&minPrice=100&maxPrice=500
Accept: application/json

Response: 200 OK
{
  "items": [...],
  "pagination": {
    "page": 1,
    "perPage": 20,
    "total": 150,
    "totalPages": 8
  }
}
```

#### POST - Create Resource

```http
POST /api/v1/products
Content-Type: application/json

{
  "name": "Smart Watch Pro",
  "description": "Advanced fitness tracking",
  "price": 399.99,
  "categoryId": 5
}

Response: 201 Created
Location: /api/v1/products/124
{
  "id": 124,
  "name": "Smart Watch Pro",
  "price": 399.99,
  "createdAt": "2024-01-15T10:30:00Z"
}
```

#### PUT - Replace Resource

```http
PUT /api/v1/products/123
Content-Type: application/json

{
  "name": "Premium Coffee Maker v2",
  "description": "Updated model",
  "price": 349.99,
  "categoryId": 3
}

Response: 200 OK
{
  "id": 123,
  "name": "Premium Coffee Maker v2",
  "price": 349.99,
  "updatedAt": "2024-01-15T11:00:00Z"
}
```

#### PATCH - Partial Update

```http
PATCH /api/v1/products/123
Content-Type: application/json-patch+json

[
  { "op": "replace", "path": "/price", "value": 279.99 },
  { "op": "add", "path": "/tags", "value": ["sale", "featured"] }
]

Response: 200 OK
{
  "id": 123,
  "name": "Premium Coffee Maker",
  "price": 279.99,
  "tags": ["sale", "featured"]
}
```

#### DELETE - Remove Resource

```http
DELETE /api/v1/products/123

Response: 204 No Content
```

## Request/Response Patterns

### Request Headers

```http
# Required Headers
Content-Type: application/json
Accept: application/json

# Authentication
Authorization: Bearer {jwt-token}

# Request Tracking
X-Request-ID: 550e8400-e29b-41d4-a716-446655440000
X-Client-ID: mobile-app-v2.1

# Conditional Requests
If-Match: "etag-value"
If-Modified-Since: Wed, 21 Oct 2023 07:28:00 GMT
```

### Response Headers

```http
# Standard Headers
Content-Type: application/json
Content-Length: 1234

# Caching
Cache-Control: public, max-age=3600
ETag: "33a64df551425fcc55e4d42a148795d9f25f89d4"
Last-Modified: Wed, 21 Oct 2023 07:28:00 GMT

# Rate Limiting
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1672531200

# Custom Headers
X-Request-ID: 550e8400-e29b-41d4-a716-446655440000
X-Response-Time: 125ms
```

### Standard Response Format

#### Success Response

```json
{
  "data": {
    "id": 123,
    "type": "product",
    "attributes": {
      "name": "Premium Coffee Maker",
      "price": 299.99,
      "inStock": true
    },
    "relationships": {
      "category": {
        "id": 5,
        "type": "category"
      }
    }
  },
  "meta": {
    "requestId": "550e8400-e29b-41d4-a716-446655440000",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

#### Error Response

```json
{
  "error": {
    "code": "PRODUCT_NOT_FOUND",
    "message": "Product with ID 999 not found",
    "details": {
      "productId": 999,
      "searchedAt": "2024-01-15T10:30:00Z"
    },
    "traceId": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

#### Collection Response

```json
{
  "data": [
    {
      "id": 1,
      "type": "product",
      "attributes": {...}
    },
    {
      "id": 2,
      "type": "product",
      "attributes": {...}
    }
  ],
  "pagination": {
    "page": 1,
    "perPage": 20,
    "total": 150,
    "totalPages": 8,
    "hasNext": true,
    "hasPrev": false
  },
  "links": {
    "self": "/api/v1/products?page=1",
    "next": "/api/v1/products?page=2",
    "last": "/api/v1/products?page=8"
  }
}
```

## Query Parameters

### Filtering

```
# Exact match
GET /api/v1/products?category=electronics

# Multiple values (OR)
GET /api/v1/products?status=active,pending

# Range queries
GET /api/v1/products?price[gte]=100&price[lte]=500
GET /api/v1/orders?createdAt[after]=2024-01-01

# Search
GET /api/v1/products?q=coffee+maker

# Complex filters
GET /api/v1/products?filter[category]=electronics&filter[brand]=samsung
```

### Sorting

```
# Single field
GET /api/v1/products?sort=price

# Descending order
GET /api/v1/products?sort=-price

# Multiple fields
GET /api/v1/products?sort=-price,name

# Nested fields
GET /api/v1/products?sort=category.name
```

### Pagination

```
# Page-based
GET /api/v1/products?page=2&perPage=50

# Offset-based
GET /api/v1/products?offset=100&limit=50

# Cursor-based
GET /api/v1/products?cursor=eyJpZCI6MTAwfQ&limit=50
```

### Field Selection

```
# Include specific fields
GET /api/v1/products/123?fields=id,name,price

# Include related resources
GET /api/v1/products/123?include=category,reviews

# Exclude fields
GET /api/v1/products/123?exclude=description,metadata
```

## Status Codes

### Success Codes

| Code | Meaning | Usage |
|------|---------|-------|
| 200 | OK | Successful GET, PUT, PATCH |
| 201 | Created | Successful POST |
| 202 | Accepted | Request accepted for async processing |
| 204 | No Content | Successful DELETE |
| 206 | Partial Content | Partial response (range requests) |

### Client Error Codes

| Code | Meaning | Usage |
|------|---------|-------|
| 400 | Bad Request | Invalid request format/parameters |
| 401 | Unauthorized | Missing or invalid authentication |
| 403 | Forbidden | Authenticated but not authorized |
| 404 | Not Found | Resource doesn't exist |
| 405 | Method Not Allowed | HTTP method not supported |
| 409 | Conflict | Resource conflict (e.g., duplicate) |
| 422 | Unprocessable Entity | Validation errors |
| 429 | Too Many Requests | Rate limit exceeded |

### Server Error Codes

| Code | Meaning | Usage |
|------|---------|-------|
| 500 | Internal Server Error | Generic server error |
| 502 | Bad Gateway | Invalid response from upstream |
| 503 | Service Unavailable | Service temporarily down |
| 504 | Gateway Timeout | Upstream timeout |

## Error Handling

### Error Response Structure

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed for the request",
    "details": [
      {
        "field": "price",
        "code": "INVALID_FORMAT",
        "message": "Price must be a positive number"
      },
      {
        "field": "categoryId",
        "code": "NOT_FOUND",
        "message": "Category with ID 999 does not exist"
      }
    ],
    "helpUrl": "https://docs.firstviscount.com/errors/VALIDATION_ERROR",
    "traceId": "550e8400-e29b-41d4-a716-446655440000",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

### Error Codes

```java
public enum ErrorCode {
    // Client errors
    INVALID_REQUEST("Invalid request format"),
    AUTHENTICATION_REQUIRED("Authentication is required"),
    INSUFFICIENT_PERMISSIONS("Insufficient permissions"),
    RESOURCE_NOT_FOUND("Resource not found"),
    DUPLICATE_RESOURCE("Resource already exists"),
    VALIDATION_ERROR("Validation failed"),
    RATE_LIMIT_EXCEEDED("Rate limit exceeded"),
    
    // Business errors
    INSUFFICIENT_INVENTORY("Insufficient inventory"),
    PAYMENT_FAILED("Payment processing failed"),
    ORDER_CANCELLED("Order has been cancelled"),
    
    // Server errors
    INTERNAL_ERROR("Internal server error"),
    SERVICE_UNAVAILABLE("Service temporarily unavailable"),
    UPSTREAM_ERROR("Upstream service error");
}
```

## API Security

### Authentication

```http
# JWT Bearer Token
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

# API Key (for service-to-service)
X-API-Key: 550e8400-e29b-41d4-a716-446655440000

# OAuth 2.0
Authorization: Bearer {access_token}
```

### Rate Limiting

```java
@RestController
@RateLimiter(name = "api", fallbackMethod = "rateLimitFallback")
public class ProductController {
    
    @GetMapping("/products")
    @RateLimit(value = 100, duration = 1, unit = TimeUnit.HOURS)
    public ResponseEntity<List<Product>> getProducts() {
        // Implementation
    }
    
    public ResponseEntity<ErrorResponse> rateLimitFallback(Exception ex) {
        return ResponseEntity
            .status(429)
            .header("Retry-After", "3600")
            .body(ErrorResponse.of("Rate limit exceeded"));
    }
}
```

### CORS Configuration

```java
@Configuration
public class CorsConfig {
    
    @Bean
    public WebMvcConfigurer corsConfigurer() {
        return new WebMvcConfigurer() {
            @Override
            public void addCorsMappings(CorsRegistry registry) {
                registry.addMapping("/api/**")
                    .allowedOrigins("https://app.firstviscount.com")
                    .allowedMethods("GET", "POST", "PUT", "PATCH", "DELETE")
                    .allowedHeaders("*")
                    .exposedHeaders("X-Total-Count", "X-Page-Number")
                    .allowCredentials(true)
                    .maxAge(3600);
            }
        };
    }
}
```

## Hypermedia (HATEOAS)

### Resource Links

```json
{
  "id": 123,
  "name": "Premium Coffee Maker",
  "price": 299.99,
  "_links": {
    "self": {
      "href": "/api/v1/products/123"
    },
    "category": {
      "href": "/api/v1/categories/5"
    },
    "reviews": {
      "href": "/api/v1/products/123/reviews"
    },
    "add-to-cart": {
      "href": "/api/v1/cart/items",
      "method": "POST"
    }
  }
}
```

### Collection Links

```json
{
  "items": [...],
  "_links": {
    "self": {
      "href": "/api/v1/products?page=2"
    },
    "first": {
      "href": "/api/v1/products?page=1"
    },
    "prev": {
      "href": "/api/v1/products?page=1"
    },
    "next": {
      "href": "/api/v1/products?page=3"
    },
    "last": {
      "href": "/api/v1/products?page=10"
    }
  }
}
```

## Asynchronous Operations

### Long-Running Operations

```http
# 1. Initiate operation
POST /api/v1/reports/generate
Content-Type: application/json

{
  "type": "sales",
  "startDate": "2024-01-01",
  "endDate": "2024-01-31"
}

Response: 202 Accepted
Location: /api/v1/reports/jobs/789
{
  "jobId": "789",
  "status": "processing",
  "estimatedCompletionTime": "2024-01-15T10:35:00Z"
}

# 2. Check status
GET /api/v1/reports/jobs/789

Response: 200 OK
{
  "jobId": "789",
  "status": "completed",
  "result": {
    "reportUrl": "/api/v1/reports/789",
    "expiresAt": "2024-01-16T10:35:00Z"
  }
}
```

### Webhooks

```json
{
  "event": "order.completed",
  "timestamp": "2024-01-15T10:30:00Z",
  "data": {
    "orderId": "ORD-12345",
    "customerId": "CUST-67890",
    "totalAmount": 599.99
  },
  "signature": "sha256=5d41402abc4b2a76b9719d911017c592"
}
```

## API Documentation

### OpenAPI Specification

```yaml
openapi: 3.0.0
info:
  title: Product Catalog API
  version: 1.0.0
  description: API for managing products in the First Viscount platform

paths:
  /api/v1/products:
    get:
      summary: List products
      operationId: listProducts
      parameters:
        - name: category
          in: query
          schema:
            type: string
        - name: page
          in: query
          schema:
            type: integer
            default: 1
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ProductList'

components:
  schemas:
    Product:
      type: object
      required:
        - id
        - name
        - price
      properties:
        id:
          type: integer
          format: int64
        name:
          type: string
          maxLength: 100
        price:
          type: number
          format: decimal
          minimum: 0
```

### API Client Examples

#### JavaScript/TypeScript

```typescript
class ProductAPI {
  private baseURL = 'https://api.firstviscount.com';
  
  async getProduct(id: number): Promise<Product> {
    const response = await fetch(`${this.baseURL}/api/v1/products/${id}`, {
      headers: {
        'Accept': 'application/json',
        'Authorization': `Bearer ${this.token}`
      }
    });
    
    if (!response.ok) {
      throw new APIError(response.status, await response.json());
    }
    
    return response.json();
  }
  
  async createProduct(product: CreateProductRequest): Promise<Product> {
    const response = await fetch(`${this.baseURL}/api/v1/products`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
        'Authorization': `Bearer ${this.token}`
      },
      body: JSON.stringify(product)
    });
    
    if (!response.ok) {
      throw new APIError(response.status, await response.json());
    }
    
    return response.json();
  }
}
```

#### Java/Spring

```java
@Component
public class ProductClient {
    
    private final WebClient webClient;
    
    public ProductClient(WebClient.Builder builder) {
        this.webClient = builder
            .baseUrl("https://api.firstviscount.com")
            .defaultHeader("Accept", "application/json")
            .filter(new AuthenticationFilter())
            .build();
    }
    
    public Mono<Product> getProduct(Long id) {
        return webClient.get()
            .uri("/api/v1/products/{id}", id)
            .retrieve()
            .onStatus(HttpStatus::is4xxClientError, this::handle4xxError)
            .onStatus(HttpStatus::is5xxServerError, this::handle5xxError)
            .bodyToMono(Product.class);
    }
    
    public Mono<Product> createProduct(CreateProductRequest request) {
        return webClient.post()
            .uri("/api/v1/products")
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue(request)
            .retrieve()
            .bodyToMono(Product.class);
    }
}
```

## Best Practices

1. **Consistency**: Use the same patterns across all APIs
2. **Versioning**: Version APIs to maintain backward compatibility
3. **Documentation**: Keep OpenAPI specs up to date
4. **Error Handling**: Provide meaningful error messages
5. **Security**: Always use HTTPS and proper authentication
6. **Performance**: Implement caching and pagination
7. **Monitoring**: Track API usage and performance metrics
8. **Testing**: Comprehensive API testing including contract tests

---
*Last Updated: January 2025*
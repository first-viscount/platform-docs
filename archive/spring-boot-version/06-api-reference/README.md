# First Viscount API Reference

## Overview

This section provides comprehensive API documentation for all First Viscount platform services. Each service exposes RESTful APIs following standard conventions with JSON payloads.

## Service APIs

### Core Services

1. **[Platform Coordination API](./platform-coordination-api.md)**
   - Workflow orchestration and saga management
   - Port: 8081
   - Base Path: `/api/v1`

2. **[Product Catalog API](./product-catalog-api.md)**
   - Product management and search capabilities
   - Port: 8082
   - Base Path: `/api/v1`

3. **[Inventory API](./inventory-api.md)**
   - Stock management and reservation system
   - Port: 8083
   - Base Path: `/api/v1`

4. **[Order Management API](./order-management-api.md)**
   - Order processing and lifecycle management
   - Port: 8084
   - Base Path: `/api/v1`

5. **[Delivery API](./delivery-api.md)**
   - Shipping coordination and tracking
   - Port: 8085
   - Base Path: `/api/v1`

6. **[Notification API](./notification-api.md)**
   - Multi-channel customer communications
   - Port: 8086
   - Base Path: `/api/v1`

## Common Standards

### Authentication

All APIs use JWT (JSON Web Token) authentication:

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### Request Headers

Standard headers for all API requests:

```http
Content-Type: application/json
Accept: application/json
Authorization: Bearer {token}
X-Request-ID: {unique-request-id}
X-Client-Version: {client-version}
```

### Response Format

All APIs return JSON responses with consistent structure:

**Success Response:**
```json
{
  "data": {
    // Response payload
  },
  "metadata": {
    "timestamp": "2024-01-15T10:30:00Z",
    "version": "1.0",
    "requestId": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

**Error Response:**
```json
{
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "Product not found",
    "details": {
      "productId": "123e4567-e89b-12d3-a456-426614174000"
    }
  },
  "metadata": {
    "timestamp": "2024-01-15T10:30:00Z",
    "requestId": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

### HTTP Status Codes

| Status Code | Description |
|------------|-------------|
| 200 | OK - Request succeeded |
| 201 | Created - Resource created successfully |
| 204 | No Content - Request succeeded with no response body |
| 400 | Bad Request - Invalid request format or parameters |
| 401 | Unauthorized - Missing or invalid authentication |
| 403 | Forbidden - Insufficient permissions |
| 404 | Not Found - Resource does not exist |
| 409 | Conflict - Resource conflict (e.g., duplicate) |
| 422 | Unprocessable Entity - Business rule violation |
| 429 | Too Many Requests - Rate limit exceeded |
| 500 | Internal Server Error - Server error |
| 503 | Service Unavailable - Service temporarily unavailable |

### Pagination

List endpoints support pagination with standard parameters:

```http
GET /api/v1/products?page=0&size=20&sort=name,asc
```

**Parameters:**
- `page` - Page number (0-based)
- `size` - Page size (default: 20, max: 100)
- `sort` - Sort criteria (format: `field,direction`)

**Response:**
```json
{
  "content": [...],
  "pageable": {
    "pageNumber": 0,
    "pageSize": 20,
    "sort": {
      "sorted": true,
      "ascending": true
    }
  },
  "totalElements": 1234,
  "totalPages": 62,
  "first": true,
  "last": false
}
```

### Rate Limiting

All APIs implement rate limiting:

- **Anonymous:** 100 requests per hour
- **Authenticated:** 1000 requests per hour
- **Service-to-Service:** 10000 requests per hour

Rate limit headers:
```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1642252800
```

### Versioning

APIs use URL path versioning:

- Current version: `v1`
- Format: `/api/v1/resource`
- Backward compatibility maintained for 6 months
- Deprecation notices provided via headers:
  ```http
  Sunset: Sat, 1 Jul 2024 00:00:00 GMT
  Deprecation: true
  Link: </api/v2/resource>; rel="successor-version"
  ```

### API Security

1. **TLS/HTTPS** - All API traffic must use HTTPS
2. **CORS** - Configured for allowed origins only
3. **Input Validation** - All inputs validated and sanitized
4. **SQL Injection Protection** - Parameterized queries only
5. **XSS Protection** - Output encoding enforced
6. **CSRF Protection** - Token-based for state-changing operations

### OpenAPI Specifications

Each service provides OpenAPI 3.0 specifications:

- **Endpoint:** `GET /api/docs/openapi.json`
- **Swagger UI:** `GET /api/docs/swagger-ui`
- **ReDoc:** `GET /api/docs/redoc`

### Service Discovery

Services register with Eureka for dynamic discovery:

- **Eureka Dashboard:** http://eureka:8761
- **Service Names:**
  - `platform-coordination-service`
  - `product-catalog-service`
  - `inventory-service`
  - `order-management-service`
  - `delivery-service`
  - `notification-service`

### Health Checks

All services expose health endpoints:

- **Liveness:** `GET /actuator/health/liveness`
- **Readiness:** `GET /actuator/health/readiness`
- **Full Health:** `GET /actuator/health`

### Monitoring

Services expose Prometheus metrics:

- **Endpoint:** `GET /actuator/prometheus`
- **Key Metrics:**
  - Request rate
  - Response time
  - Error rate
  - Business metrics

## Getting Started

1. **Obtain JWT Token** - Authenticate via the auth service
2. **Include Token** - Add to Authorization header
3. **Make Requests** - Use service endpoints as documented
4. **Handle Responses** - Parse JSON responses
5. **Implement Retry** - Use exponential backoff for failures

## API Testing

### Using cURL

```bash
# Get products
curl -X GET "http://localhost:8082/api/v1/products" \
  -H "Authorization: Bearer your-jwt-token" \
  -H "Content-Type: application/json"

# Create order
curl -X POST "http://localhost:8084/api/v1/orders" \
  -H "Authorization: Bearer your-jwt-token" \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": "123e4567-e89b-12d3-a456-426614174000",
    "items": [
      {
        "productId": "789e0123-e89b-12d3-a456-426614174000",
        "quantity": 2
      }
    ]
  }'
```

### Using Postman

Import the OpenAPI specifications into Postman for interactive testing:

1. Open Postman
2. Import → Link → Enter OpenAPI URL
3. Configure environment variables
4. Set authentication token
5. Execute requests

## Support

For API support and questions:

- **Documentation Issues:** Create GitHub issue
- **Service Issues:** Check service health endpoints
- **Security Issues:** Contact security team immediately
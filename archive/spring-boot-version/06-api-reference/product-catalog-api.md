# Product Catalog API Reference

## Overview

The Product Catalog API provides comprehensive product management capabilities including CRUD operations, search functionality, category management, and pricing. This API serves as the single source of truth for all product-related data in the First Viscount platform.

**Base URL:** `http://product-catalog:8082/api/v1`  
**Port:** 8082

## Authentication

All endpoints require JWT authentication unless otherwise specified:

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

## OpenAPI Specification

```yaml
openapi: 3.0.3
info:
  title: Product Catalog API
  version: 1.0.0
  description: Product management and search API for First Viscount platform
servers:
  - url: http://product-catalog:8082/api/v1
    description: Internal service endpoint
  - url: https://api.firstviscount.com/catalog/v1
    description: Production API gateway
```

## Endpoints

### Product Management

#### List Products

Retrieve a paginated list of products with optional filtering.

```http
GET /products
```

**Query Parameters:**

| Parameter | Type | Required | Description | Default |
|-----------|------|----------|-------------|---------|
| page | integer | No | Page number (0-based) | 0 |
| size | integer | No | Page size (max 100) | 20 |
| sort | string | No | Sort criteria (field,direction) | name,asc |
| category | string | No | Filter by category ID | - |
| minPrice | number | No | Minimum price filter | - |
| maxPrice | number | No | Maximum price filter | - |
| brand | string | No | Filter by brand name | - |
| status | string | No | Filter by status (ACTIVE, DRAFT, ARCHIVED) | ACTIVE |

**Success Response (200 OK):**

```json
{
  "content": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "sku": "LAPTOP-001",
      "name": "Premium Laptop Pro 15",
      "description": "High-performance laptop with 16GB RAM",
      "price": {
        "amount": 1299.99,
        "currency": "USD",
        "formattedPrice": "$1,299.99"
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
          "url": "https://cdn.firstviscount.com/products/laptop-001-main.jpg",
          "type": "main",
          "alt": "Premium Laptop Pro 15 - Front View"
        }
      ],
      "availability": {
        "inStock": true,
        "quantity": 45,
        "warehouse": "MAIN"
      },
      "rating": {
        "average": 4.5,
        "count": 234
      },
      "status": "ACTIVE",
      "createdAt": "2024-01-01T10:00:00Z",
      "updatedAt": "2024-01-15T14:30:00Z"
    }
  ],
  "pageable": {
    "pageNumber": 0,
    "pageSize": 20,
    "sort": {
      "sorted": true,
      "ascending": true,
      "property": "name"
    }
  },
  "totalElements": 156,
  "totalPages": 8,
  "first": true,
  "last": false,
  "numberOfElements": 20
}
```

**Error Responses:**

- **400 Bad Request** - Invalid query parameters
- **401 Unauthorized** - Missing or invalid authentication
- **500 Internal Server Error** - Server error

#### Get Product by ID

Retrieve detailed information about a specific product.

```http
GET /products/{productId}
```

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| productId | string (UUID) | Yes | Product identifier |

**Success Response (200 OK):**

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "sku": "LAPTOP-001",
  "name": "Premium Laptop Pro 15",
  "description": "High-performance laptop with 16GB RAM and dedicated graphics",
  "longDescription": "The Premium Laptop Pro 15 combines power and portability in a sleek aluminum design...",
  "price": {
    "amount": 1299.99,
    "currency": "USD",
    "formattedPrice": "$1,299.99",
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
      "currency": "USD",
      "effectiveFrom": "2024-01-01T00:00:00Z",
      "effectiveTo": "2024-01-10T00:00:00Z"
    }
  ],
  "category": {
    "id": "123e4567-e89b-12d3-a456-426614174000",
    "name": "Laptops",
    "path": "Electronics/Computers/Laptops",
    "breadcrumb": [
      {"id": "aaa", "name": "Electronics", "slug": "electronics"},
      {"id": "bbb", "name": "Computers", "slug": "computers"},
      {"id": "123e4567-e89b-12d3-a456-426614174000", "name": "Laptops", "slug": "laptops"}
    ]
  },
  "brand": "TechBrand",
  "manufacturer": {
    "name": "Tech Manufacturing Co.",
    "country": "USA"
  },
  "specifications": {
    "general": {
      "brand": "TechBrand",
      "model": "Pro 15",
      "year": "2024",
      "warranty": "2 years"
    },
    "technical": {
      "processor": {
        "brand": "Intel",
        "model": "Core i7-12700H",
        "cores": 14,
        "threads": 20,
        "baseSpeed": "2.3 GHz",
        "turboSpeed": "4.7 GHz"
      },
      "memory": {
        "size": "16GB",
        "type": "DDR5",
        "speed": "4800 MHz",
        "expandable": true,
        "maxSize": "64GB"
      },
      "storage": {
        "primary": {
          "type": "NVMe SSD",
          "capacity": "512GB",
          "interface": "PCIe 4.0"
        }
      },
      "display": {
        "size": "15.6 inch",
        "resolution": "1920x1080",
        "type": "IPS",
        "refreshRate": "144Hz",
        "brightness": "400 nits",
        "colorGamut": "100% sRGB"
      },
      "graphics": {
        "brand": "NVIDIA",
        "model": "RTX 3060",
        "memory": "6GB GDDR6"
      }
    },
    "physical": {
      "weight": "1.8kg",
      "dimensions": {
        "length": "35.5cm",
        "width": "23.5cm",
        "height": "1.8cm"
      },
      "color": "Space Gray",
      "material": "Aluminum"
    },
    "connectivity": {
      "wifi": "Wi-Fi 6E",
      "bluetooth": "5.2",
      "ports": [
        "2x USB-C Thunderbolt 4",
        "2x USB-A 3.2",
        "1x HDMI 2.1",
        "1x Audio Jack"
      ]
    }
  },
  "images": [
    {
      "id": "img-001",
      "url": "https://cdn.firstviscount.com/products/laptop-001-main.jpg",
      "type": "main",
      "displayOrder": 0,
      "alt": "Premium Laptop Pro 15 - Front View"
    },
    {
      "id": "img-002",
      "url": "https://cdn.firstviscount.com/products/laptop-001-side.jpg",
      "type": "additional",
      "displayOrder": 1,
      "alt": "Premium Laptop Pro 15 - Side View"
    }
  ],
  "attributes": {
    "processor": "Intel i7",
    "ram": "16GB",
    "storage": "512GB SSD",
    "display": "15.6 inch",
    "weight": "1.8kg"
  },
  "tags": ["gaming", "professional", "high-performance", "portable"],
  "seo": {
    "metaTitle": "Premium Laptop Pro 15 - High-Performance Gaming & Professional Laptop",
    "metaDescription": "Experience ultimate performance with the Premium Laptop Pro 15...",
    "slug": "premium-laptop-pro-15"
  },
  "availability": {
    "inStock": true,
    "quantity": 45,
    "warehouse": "MAIN",
    "backorderable": true,
    "preorderable": false
  },
  "shipping": {
    "weight": 2.5,
    "dimensions": {
      "length": 45,
      "width": 30,
      "height": 8
    },
    "freeShipping": true,
    "shippingClass": "STANDARD"
  },
  "rating": {
    "average": 4.5,
    "count": 234,
    "distribution": {
      "5": 150,
      "4": 60,
      "3": 15,
      "2": 5,
      "1": 4
    }
  },
  "relatedProducts": [
    "660e8400-e29b-41d4-a716-446655440000",
    "770e8400-e29b-41d4-a716-446655440000"
  ],
  "status": "ACTIVE",
  "createdAt": "2024-01-01T10:00:00Z",
  "updatedAt": "2024-01-15T14:30:00Z",
  "version": 3
}
```

**Error Responses:**

- **404 Not Found** - Product does not exist
- **401 Unauthorized** - Missing or invalid authentication

#### Create Product (Admin Only)

Create a new product in the catalog.

```http
POST /products
```

**Request Headers:**

```http
Content-Type: application/json
Authorization: Bearer {admin-token}
```

**Request Body:**

```json
{
  "sku": "LAPTOP-002",
  "name": "Business Laptop Essential",
  "description": "Reliable laptop for business use",
  "longDescription": "The Business Laptop Essential is designed for professionals...",
  "categoryId": "123e4567-e89b-12d3-a456-426614174000",
  "price": {
    "amount": 799.99,
    "currency": "USD"
  },
  "brand": "TechBrand",
  "attributes": {
    "processor": "Intel i5",
    "ram": "8GB",
    "storage": "256GB SSD",
    "display": "14 inch"
  },
  "specifications": {
    "general": {
      "model": "Essential 14",
      "year": "2024"
    },
    "technical": {
      "processor": {
        "brand": "Intel",
        "model": "Core i5-1235U"
      }
    }
  },
  "images": [
    {
      "url": "https://cdn.firstviscount.com/products/laptop-002-main.jpg",
      "type": "main",
      "alt": "Business Laptop Essential"
    }
  ],
  "tags": ["business", "portable", "affordable"],
  "status": "DRAFT"
}
```

**Success Response (201 Created):**

```json
{
  "id": "660e8400-e29b-41d4-a716-446655440000",
  "sku": "LAPTOP-002",
  "name": "Business Laptop Essential",
  "status": "DRAFT",
  "createdAt": "2024-01-15T15:00:00Z",
  "createdBy": "admin-123",
  "version": 1
}
```

**Error Responses:**

- **400 Bad Request** - Invalid request data
- **401 Unauthorized** - Missing or invalid authentication
- **403 Forbidden** - Insufficient permissions
- **409 Conflict** - SKU already exists

#### Update Product (Admin Only)

Update an existing product's information.

```http
PUT /products/{productId}
```

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| productId | string (UUID) | Yes | Product identifier |

**Request Body:**

```json
{
  "name": "Business Laptop Essential Plus",
  "price": {
    "amount": 849.99,
    "currency": "USD"
  },
  "attributes": {
    "processor": "Intel i5",
    "ram": "16GB",
    "storage": "512GB SSD",
    "display": "14 inch"
  },
  "status": "ACTIVE"
}
```

**Success Response (200 OK):**

```json
{
  "id": "660e8400-e29b-41d4-a716-446655440000",
  "sku": "LAPTOP-002",
  "name": "Business Laptop Essential Plus",
  "price": {
    "amount": 849.99,
    "currency": "USD"
  },
  "status": "ACTIVE",
  "updatedAt": "2024-01-15T15:30:00Z",
  "updatedBy": "admin-123",
  "version": 2
}
```

**Error Responses:**

- **400 Bad Request** - Invalid request data
- **401 Unauthorized** - Missing or invalid authentication
- **403 Forbidden** - Insufficient permissions
- **404 Not Found** - Product does not exist
- **409 Conflict** - Version conflict (optimistic locking)

#### Delete Product (Admin Only)

Delete a product from the catalog.

```http
DELETE /products/{productId}
```

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| productId | string (UUID) | Yes | Product identifier |

**Success Response (204 No Content)**

**Error Responses:**

- **401 Unauthorized** - Missing or invalid authentication
- **403 Forbidden** - Insufficient permissions
- **404 Not Found** - Product does not exist
- **409 Conflict** - Product has active orders

### Product Search

#### Advanced Search

Perform advanced product search with filters and facets.

```http
POST /products/search
```

**Request Body:**

```json
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
      "graphics": ["RTX 3060", "RTX 3070"],
      "processor": ["Intel i7", "Intel i9"]
    },
    "inStock": true,
    "rating": {
      "min": 4.0
    }
  },
  "sort": {
    "field": "price",
    "direction": "ASC"
  },
  "facets": ["category", "brand", "price", "attributes"],
  "page": 0,
  "size": 20
}
```

**Success Response (200 OK):**

```json
{
  "results": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "sku": "LAPTOP-001",
      "name": "Premium Laptop Pro 15",
      "description": "High-performance laptop with 16GB RAM",
      "price": {
        "amount": 1299.99,
        "currency": "USD"
      },
      "score": 0.95,
      "highlights": {
        "name": "Premium <em>Laptop</em> Pro 15",
        "description": "High-performance <em>laptop</em> with 16GB RAM"
      }
    }
  ],
  "facets": {
    "categories": [
      {
        "id": "123",
        "name": "Gaming Laptops",
        "count": 45,
        "path": "Electronics/Computers/Gaming Laptops"
      },
      {
        "id": "124",
        "name": "Professional Laptops",
        "count": 23,
        "path": "Electronics/Computers/Professional Laptops"
      }
    ],
    "brands": [
      {"name": "TechBrand", "count": 32},
      {"name": "GameBrand", "count": 28}
    ],
    "priceRanges": [
      {"range": "1000-1250", "count": 15, "min": 1000, "max": 1250},
      {"range": "1250-1500", "count": 20, "min": 1250, "max": 1500},
      {"range": "1500-1750", "count": 18, "min": 1500, "max": 1750},
      {"range": "1750-2000", "count": 12, "min": 1750, "max": 2000}
    ],
    "attributes": {
      "ram": [
        {"value": "16GB", "count": 40},
        {"value": "32GB", "count": 25}
      ],
      "graphics": [
        {"value": "RTX 3060", "count": 35},
        {"value": "RTX 3070", "count": 30}
      ]
    }
  },
  "totalResults": 65,
  "page": 0,
  "totalPages": 4,
  "searchTime": 125
}
```

#### Quick Search

Perform a quick text-based search across products.

```http
GET /products/search?q={query}
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| q | string | Yes | Search query |
| limit | integer | No | Result limit (default: 10, max: 50) |

**Success Response (200 OK):**

```json
{
  "results": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "sku": "LAPTOP-001",
      "name": "Premium Laptop Pro 15",
      "price": 1299.99,
      "image": "https://cdn.firstviscount.com/products/laptop-001-thumb.jpg"
    }
  ],
  "total": 5
}
```

### Category Management

#### List Categories

Retrieve the category hierarchy.

```http
GET /categories
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| parent | string | No | Parent category ID (null for root) |
| depth | integer | No | Tree depth to return (default: all) |
| includeEmpty | boolean | No | Include categories with no products |

**Success Response (200 OK):**

```json
{
  "categories": [
    {
      "id": "root-1",
      "name": "Electronics",
      "slug": "electronics",
      "description": "Electronic devices and accessories",
      "image": "https://cdn.firstviscount.com/categories/electronics.jpg",
      "level": 0,
      "productCount": 1250,
      "children": [
        {
          "id": "cat-1",
          "name": "Computers",
          "slug": "computers",
          "level": 1,
          "productCount": 456,
          "children": [
            {
              "id": "123e4567-e89b-12d3-a456-426614174000",
              "name": "Laptops",
              "slug": "laptops",
              "level": 2,
              "productCount": 156,
              "attributes": ["processor", "ram", "storage", "display"]
            },
            {
              "id": "223e4567-e89b-12d3-a456-426614174000",
              "name": "Desktops",
              "slug": "desktops",
              "level": 2,
              "productCount": 89
            }
          ]
        }
      ]
    }
  ],
  "total": 45
}
```

#### Get Category

Get detailed information about a specific category.

```http
GET /categories/{categoryId}
```

**Success Response (200 OK):**

```json
{
  "id": "123e4567-e89b-12d3-a456-426614174000",
  "name": "Laptops",
  "slug": "laptops",
  "description": "Portable computers for work and gaming",
  "path": "Electronics/Computers/Laptops",
  "breadcrumb": [
    {"id": "root-1", "name": "Electronics", "slug": "electronics"},
    {"id": "cat-1", "name": "Computers", "slug": "computers"},
    {"id": "123e4567-e89b-12d3-a456-426614174000", "name": "Laptops", "slug": "laptops"}
  ],
  "parent": {
    "id": "cat-1",
    "name": "Computers"
  },
  "children": [],
  "attributesSchema": {
    "processor": {
      "type": "select",
      "required": true,
      "options": ["Intel i5", "Intel i7", "Intel i9", "AMD Ryzen 5", "AMD Ryzen 7"]
    },
    "ram": {
      "type": "select",
      "required": true,
      "options": ["4GB", "8GB", "16GB", "32GB", "64GB"]
    }
  },
  "seo": {
    "metaTitle": "Laptops - Shop Portable Computers",
    "metaDescription": "Find the perfect laptop for your needs..."
  },
  "productCount": 156,
  "isActive": true
}
```

### Price Management

#### Get Price History

Retrieve price history for a product.

```http
GET /products/{productId}/prices
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| from | datetime | No | Start date filter |
| to | datetime | No | End date filter |

**Success Response (200 OK):**

```json
{
  "productId": "550e8400-e29b-41d4-a716-446655440000",
  "currentPrice": {
    "amount": 1299.99,
    "currency": "USD",
    "effectiveFrom": "2024-01-10T00:00:00Z"
  },
  "history": [
    {
      "id": "price-001",
      "amount": 1499.99,
      "currency": "USD",
      "effectiveFrom": "2024-01-01T00:00:00Z",
      "effectiveTo": "2024-01-10T00:00:00Z",
      "reason": "INITIAL_PRICE"
    },
    {
      "id": "price-002",
      "amount": 1299.99,
      "currency": "USD",
      "effectiveFrom": "2024-01-10T00:00:00Z",
      "effectiveTo": null,
      "reason": "PROMOTION",
      "createdBy": "admin-123"
    }
  ]
}
```

#### Update Product Price (Admin Only)

Update the price of a product.

```http
POST /products/{productId}/prices
```

**Request Body:**

```json
{
  "amount": 1199.99,
  "currency": "USD",
  "reason": "SEASONAL_SALE",
  "effectiveFrom": "2024-02-01T00:00:00Z",
  "effectiveTo": "2024-02-28T23:59:59Z"
}
```

**Success Response (201 Created):**

```json
{
  "id": "price-003",
  "productId": "550e8400-e29b-41d4-a716-446655440000",
  "amount": 1199.99,
  "currency": "USD",
  "reason": "SEASONAL_SALE",
  "effectiveFrom": "2024-02-01T00:00:00Z",
  "effectiveTo": "2024-02-28T23:59:59Z",
  "createdAt": "2024-01-20T10:00:00Z",
  "createdBy": "admin-123"
}
```

## Error Handling

All error responses follow a consistent format:

```json
{
  "error": {
    "code": "PRODUCT_NOT_FOUND",
    "message": "Product with ID 550e8400-e29b-41d4-a716-446655440000 not found",
    "details": {
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "timestamp": "2024-01-15T10:30:00Z"
    },
    "traceId": "b9d7c9a8-3e2f-4d1e-8c7b-6a5d4c3b2a1f"
  }
}
```

### Error Codes

| Code | Description | HTTP Status |
|------|-------------|-------------|
| INVALID_REQUEST | Request validation failed | 400 |
| AUTHENTICATION_REQUIRED | Missing authentication | 401 |
| INSUFFICIENT_PERMISSIONS | User lacks required permissions | 403 |
| PRODUCT_NOT_FOUND | Product does not exist | 404 |
| SKU_ALREADY_EXISTS | SKU conflict | 409 |
| VERSION_CONFLICT | Optimistic locking failure | 409 |
| INVALID_PRICE | Price validation failed | 422 |
| CATEGORY_NOT_FOUND | Category does not exist | 404 |
| SEARCH_QUERY_TOO_BROAD | Search requires more filters | 400 |
| RATE_LIMIT_EXCEEDED | Too many requests | 429 |
| INTERNAL_ERROR | Server error | 500 |

## Rate Limiting

The API implements rate limiting to ensure fair usage:

- **Standard Users:** 1000 requests per hour
- **Admin Users:** 5000 requests per hour
- **Service Accounts:** 10000 requests per hour

Rate limit information is included in response headers:

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1642252800
```

When rate limit is exceeded, the API returns:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 3600

{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Rate limit exceeded. Please retry after 3600 seconds.",
    "retryAfter": 3600
  }
}
```

## Webhooks

The Product Catalog service can send webhooks for product events:

### Event Types

- `product.created` - New product created
- `product.updated` - Product information updated
- `product.deleted` - Product deleted
- `price.changed` - Product price changed
- `stock.updated` - Stock level changed

### Webhook Payload

```json
{
  "id": "webhook-001",
  "timestamp": "2024-01-15T10:30:00Z",
  "event": "product.updated",
  "data": {
    "productId": "550e8400-e29b-41d4-a716-446655440000",
    "sku": "LAPTOP-001",
    "changes": {
      "price": {
        "old": 1299.99,
        "new": 1199.99
      }
    }
  },
  "signature": "sha256=abcdef123456..."
}
```

## SDK Examples

### JavaScript/TypeScript

```typescript
import { ProductCatalogClient } from '@firstviscount/product-catalog-sdk';

const client = new ProductCatalogClient({
  baseUrl: 'https://api.firstviscount.com/catalog/v1',
  apiKey: process.env.API_KEY
});

// Search products
const results = await client.products.search({
  query: 'laptop',
  filters: {
    priceRange: { min: 1000, max: 2000 },
    brands: ['TechBrand']
  }
});

// Get product details
const product = await client.products.get('550e8400-e29b-41d4-a716-446655440000');

// Create product (admin)
const newProduct = await client.products.create({
  sku: 'NEW-001',
  name: 'New Product',
  price: { amount: 99.99, currency: 'USD' }
});
```

### Python

```python
from firstviscount import ProductCatalogClient

client = ProductCatalogClient(
    base_url='https://api.firstviscount.com/catalog/v1',
    api_key=os.environ['API_KEY']
)

# Search products
results = client.products.search(
    query='laptop',
    filters={
        'price_range': {'min': 1000, 'max': 2000},
        'brands': ['TechBrand']
    }
)

# Get product details
product = client.products.get('550e8400-e29b-41d4-a716-446655440000')

# Update price (admin)
client.products.update_price(
    product_id='550e8400-e29b-41d4-a716-446655440000',
    amount=1199.99,
    reason='SEASONAL_SALE'
)
```

### Java

```java
import com.firstviscount.catalog.ProductCatalogClient;
import com.firstviscount.catalog.model.*;

ProductCatalogClient client = ProductCatalogClient.builder()
    .baseUrl("https://api.firstviscount.com/catalog/v1")
    .apiKey(System.getenv("API_KEY"))
    .build();

// Search products
SearchRequest request = SearchRequest.builder()
    .query("laptop")
    .addFilter("brand", "TechBrand")
    .priceRange(1000, 2000)
    .build();

SearchResults results = client.products().search(request);

// Get product
Product product = client.products()
    .get("550e8400-e29b-41d4-a716-446655440000");

// Create product (admin)
CreateProductRequest createRequest = CreateProductRequest.builder()
    .sku("NEW-001")
    .name("New Product")
    .price(Money.of(99.99, "USD"))
    .build();

Product newProduct = client.products().create(createRequest);
```

## Testing

### Test Environment

```
Base URL: https://api-test.firstviscount.com/catalog/v1
```

### Test Credentials

```
# Standard user
Authorization: Bearer test_user_token_123

# Admin user
Authorization: Bearer test_admin_token_456

# Service account
Authorization: Bearer test_service_token_789
```

### Sample Test Data

| Product ID | SKU | Name | Price |
|------------|-----|------|-------|
| 550e8400-e29b-41d4-a716-446655440000 | TEST-001 | Test Laptop | $999.99 |
| 660e8400-e29b-41d4-a716-446655440000 | TEST-002 | Test Desktop | $1299.99 |
| 770e8400-e29b-41d4-a716-446655440000 | TEST-003 | Test Monitor | $399.99 |

## Changelog

### Version 1.0.0 (2024-01-15)

- Initial release
- Product CRUD operations
- Advanced search with facets
- Category management
- Price history tracking
- Webhook support
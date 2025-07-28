# Order Management API Reference

## Overview

The Order Management API handles the complete order lifecycle from creation through fulfillment. This API manages order state transitions, payment processing integration, order tracking, refunds, and cancellations.

**Base URL:** `http://order-service:8084/api/v1`  
**Port:** 8084

## Authentication

All endpoints require JWT authentication:

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

## OpenAPI Specification

```yaml
openapi: 3.0.3
info:
  title: Order Management API
  version: 1.0.0
  description: Complete order lifecycle management for First Viscount platform
servers:
  - url: http://order-service:8084/api/v1
    description: Internal service endpoint
  - url: https://api.firstviscount.com/orders/v1
    description: Production API gateway
```

## Endpoints

### Order Creation and Management

#### Create Order

Create a new order with items, shipping details, and payment method.

```http
POST /orders
```

**Request Headers:**

```http
Content-Type: application/json
Authorization: Bearer {token}
X-Idempotency-Key: {unique-key}
```

**Request Body:**

```json
{
  "customerId": "123e4567-e89b-12d3-a456-426614174000",
  "items": [
    {
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "quantity": 2,
      "unitPrice": 599.99,
      "metadata": {
        "giftWrap": true,
        "giftMessage": "Happy Birthday!"
      }
    },
    {
      "productId": "660e8400-e29b-41d4-a716-446655440000",
      "quantity": 1,
      "unitPrice": 299.99
    }
  ],
  "shippingAddress": {
    "name": "John Doe",
    "company": "Acme Corp",
    "street": "123 Main St",
    "streetLine2": "Apt 4B",
    "city": "New York",
    "state": "NY",
    "zipCode": "10001",
    "country": "US",
    "phone": "+1-234-567-8900",
    "email": "john.doe@example.com",
    "instructions": "Leave at door"
  },
  "billingAddress": {
    "sameAsShipping": true
  },
  "paymentMethod": {
    "type": "CREDIT_CARD",
    "token": "tok_visa_4242",
    "lastFourDigits": "4242",
    "brand": "visa",
    "saveForFuture": true
  },
  "shippingMethod": "STANDARD",
  "promotionCodes": ["SAVE10", "FREESHIP"],
  "metadata": {
    "source": "mobile_app",
    "campaign": "summer_sale"
  }
}
```

**Success Response (201 Created):**

```json
{
  "orderId": "ord-770e8400-e29b-41d4-a716-446655440000",
  "orderNumber": "ORD-2024-001234",
  "status": "PENDING_PAYMENT",
  "customerId": "123e4567-e89b-12d3-a456-426614174000",
  "items": [
    {
      "orderItemId": "item-880e8400-e29b-41d4-a716-446655440000",
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "productName": "Premium Laptop Pro 15",
      "productSku": "LAPTOP-001",
      "quantity": 2,
      "unitPrice": 599.99,
      "subtotal": 1199.98,
      "tax": 96.00,
      "total": 1295.98
    },
    {
      "orderItemId": "item-990e8400-e29b-41d4-a716-446655440000",
      "productId": "660e8400-e29b-41d4-a716-446655440000",
      "productName": "Wireless Mouse",
      "productSku": "MOUSE-001",
      "quantity": 1,
      "unitPrice": 299.99,
      "subtotal": 299.99,
      "tax": 24.00,
      "total": 323.99
    }
  ],
  "pricing": {
    "subtotal": 1499.97,
    "shippingCost": 0.00,
    "shippingDiscount": 9.99,
    "tax": 120.00,
    "productDiscount": 149.99,
    "total": 1469.98,
    "currency": "USD"
  },
  "promotionsApplied": [
    {
      "code": "SAVE10",
      "type": "PERCENTAGE",
      "discount": 149.99,
      "description": "10% off entire order"
    },
    {
      "code": "FREESHIP",
      "type": "FREE_SHIPPING",
      "discount": 9.99,
      "description": "Free standard shipping"
    }
  ],
  "shippingMethod": "STANDARD",
  "estimatedDelivery": {
    "earliest": "2024-01-20",
    "latest": "2024-01-22"
  },
  "paymentStatus": "PENDING",
  "createdAt": "2024-01-15T10:00:00Z",
  "expiresAt": "2024-01-15T10:30:00Z"
}
```

**Error Responses:**

- **400 Bad Request** - Invalid order data
- **401 Unauthorized** - Missing or invalid authentication
- **402 Payment Required** - Payment method validation failed
- **409 Conflict** - Duplicate order (idempotency key conflict)
- **422 Unprocessable Entity** - Business rule violation (e.g., out of stock)

#### Get Order by ID

Retrieve detailed information about a specific order.

```http
GET /orders/{orderId}
```

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| orderId | string (UUID) | Yes | Order identifier |

**Success Response (200 OK):**

```json
{
  "orderId": "ord-770e8400-e29b-41d4-a716-446655440000",
  "orderNumber": "ORD-2024-001234",
  "status": "PROCESSING",
  "statusHistory": [
    {
      "status": "CREATED",
      "timestamp": "2024-01-15T10:00:00Z",
      "description": "Order created"
    },
    {
      "status": "PENDING_PAYMENT",
      "timestamp": "2024-01-15T10:00:01Z",
      "description": "Awaiting payment"
    },
    {
      "status": "PAID",
      "timestamp": "2024-01-15T10:00:30Z",
      "description": "Payment received",
      "metadata": {
        "transactionId": "ch_1234567890",
        "paymentMethod": "visa_4242"
      }
    },
    {
      "status": "PROCESSING",
      "timestamp": "2024-01-15T10:01:00Z",
      "description": "Order sent to fulfillment center",
      "metadata": {
        "fulfillmentCenter": "NYC-01",
        "estimatedProcessingTime": "PT2H"
      }
    }
  ],
  "customer": {
    "customerId": "123e4567-e89b-12d3-a456-426614174000",
    "email": "john.doe@example.com",
    "name": "John Doe",
    "phone": "+1-234-567-8900",
    "loyaltyTier": "GOLD"
  },
  "items": [
    {
      "orderItemId": "item-880e8400-e29b-41d4-a716-446655440000",
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "productName": "Premium Laptop Pro 15",
      "productSku": "LAPTOP-001",
      "quantity": 2,
      "unitPrice": 599.99,
      "subtotal": 1199.98,
      "tax": 96.00,
      "total": 1295.98,
      "status": "PROCESSING",
      "serialNumbers": []
    }
  ],
  "pricing": {
    "subtotal": 1499.97,
    "shippingCost": 0.00,
    "tax": 120.00,
    "discount": 159.98,
    "total": 1469.98,
    "currency": "USD",
    "exchangeRate": 1.0
  },
  "payment": {
    "method": "CREDIT_CARD",
    "status": "CAPTURED",
    "transactionId": "ch_1234567890",
    "amount": 1469.98,
    "currency": "USD",
    "processedAt": "2024-01-15T10:00:30Z",
    "card": {
      "brand": "visa",
      "lastFour": "4242",
      "expiryMonth": 12,
      "expiryYear": 2025
    },
    "billingAddress": {
      "name": "John Doe",
      "street": "123 Main St",
      "city": "New York",
      "state": "NY",
      "zipCode": "10001",
      "country": "US"
    }
  },
  "shipping": {
    "method": "STANDARD",
    "cost": 0.00,
    "carrier": null,
    "trackingNumber": null,
    "estimatedDelivery": {
      "earliest": "2024-01-20",
      "latest": "2024-01-22"
    },
    "address": {
      "name": "John Doe",
      "company": "Acme Corp",
      "street": "123 Main St",
      "streetLine2": "Apt 4B",
      "city": "New York",
      "state": "NY",
      "zipCode": "10001",
      "country": "US",
      "phone": "+1-234-567-8900"
    }
  },
  "timeline": {
    "created": "2024-01-15T10:00:00Z",
    "paid": "2024-01-15T10:00:30Z",
    "processing": "2024-01-15T10:01:00Z",
    "shipped": null,
    "delivered": null,
    "completed": null
  },
  "notes": [],
  "tags": ["vip_customer", "expedite"],
  "metadata": {
    "source": "mobile_app",
    "campaign": "summer_sale",
    "referrer": "email_campaign_123"
  },
  "createdAt": "2024-01-15T10:00:00Z",
  "updatedAt": "2024-01-15T10:01:00Z",
  "version": 3
}
```

**Error Responses:**

- **401 Unauthorized** - Missing or invalid authentication
- **403 Forbidden** - Not authorized to view this order
- **404 Not Found** - Order does not exist

#### List Customer Orders

Retrieve a paginated list of orders for a specific customer.

```http
GET /orders/customer/{customerId}
```

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| customerId | string (UUID) | Yes | Customer identifier |

**Query Parameters:**

| Parameter | Type | Required | Description | Default |
|-----------|------|----------|-------------|---------|
| status | string | No | Filter by order status | all |
| dateFrom | date | No | Start date filter | - |
| dateTo | date | No | End date filter | - |
| page | integer | No | Page number (0-based) | 0 |
| size | integer | No | Page size (max 100) | 20 |
| sort | string | No | Sort criteria | createdAt,desc |

**Success Response (200 OK):**

```json
{
  "content": [
    {
      "orderId": "ord-770e8400-e29b-41d4-a716-446655440000",
      "orderNumber": "ORD-2024-001234",
      "status": "DELIVERED",
      "itemCount": 3,
      "totalItems": 5,
      "total": 1469.98,
      "currency": "USD",
      "createdAt": "2024-01-15T10:00:00Z",
      "deliveredAt": "2024-01-21T14:30:00Z",
      "summary": {
        "items": [
          {
            "name": "Premium Laptop Pro 15",
            "quantity": 2,
            "image": "https://cdn.firstviscount.com/products/laptop-001-thumb.jpg"
          }
        ]
      }
    }
  ],
  "pageable": {
    "pageNumber": 0,
    "pageSize": 20,
    "sort": {
      "sorted": true,
      "descending": true,
      "property": "createdAt"
    }
  },
  "totalElements": 45,
  "totalPages": 3,
  "first": true,
  "last": false,
  "statistics": {
    "totalOrders": 45,
    "totalSpent": 23456.78,
    "averageOrderValue": 521.26,
    "ordersByStatus": {
      "COMPLETED": 40,
      "PROCESSING": 3,
      "DELIVERED": 2
    }
  }
}
```

### Order Status Management

#### Update Order Status (Admin Only)

Update the status of an order (internal use).

```http
PUT /orders/{orderId}/status
```

**Request Body:**

```json
{
  "status": "SHIPPED",
  "reason": "Order dispatched from fulfillment center",
  "metadata": {
    "trackingNumber": "1Z999AA10123456784",
    "carrier": "UPS",
    "service": "GROUND",
    "weight": 2.5,
    "dimensions": {
      "length": 45,
      "width": 30,
      "height": 15
    },
    "fulfillmentCenter": "NYC-01",
    "shippedAt": "2024-01-16T09:00:00Z"
  },
  "notifyCustomer": true
}
```

**Success Response (200 OK):**

```json
{
  "orderId": "ord-770e8400-e29b-41d4-a716-446655440000",
  "previousStatus": "PROCESSING",
  "newStatus": "SHIPPED",
  "transitionValid": true,
  "updatedAt": "2024-01-16T09:00:00Z",
  "notification": {
    "sent": true,
    "type": "ORDER_SHIPPED",
    "channels": ["email", "sms"]
  }
}
```

**Error Responses:**

- **400 Bad Request** - Invalid status transition
- **401 Unauthorized** - Missing or invalid authentication
- **403 Forbidden** - Insufficient permissions
- **404 Not Found** - Order does not exist

### Order Cancellation

#### Cancel Order

Request cancellation of an order.

```http
POST /orders/{orderId}/cancel
```

**Request Body:**

```json
{
  "reason": "CUSTOMER_REQUEST",
  "reasonDetails": "Found a better price elsewhere",
  "refundMethod": "ORIGINAL_PAYMENT_METHOD",
  "notifyCustomer": true,
  "metadata": {
    "source": "customer_service",
    "agentId": "agent-123"
  }
}
```

**Success Response (202 Accepted):**

```json
{
  "orderId": "ord-770e8400-e29b-41d4-a716-446655440000",
  "status": "CANCELLING",
  "cancellationId": "cancel-990e8400-e29b-41d4-a716-446655440000",
  "reason": "CUSTOMER_REQUEST",
  "refund": {
    "amount": 1469.98,
    "currency": "USD",
    "status": "PENDING",
    "method": "ORIGINAL_PAYMENT_METHOD",
    "estimatedDate": "2024-01-18",
    "breakdown": {
      "productTotal": 1499.97,
      "tax": 120.00,
      "shipping": 0.00,
      "discount": -159.98,
      "refundable": 1469.98
    }
  },
  "cancellationFee": 0.00,
  "message": "Your order is being cancelled. Refund will be processed within 3-5 business days.",
  "actions": [
    "RELEASE_INVENTORY",
    "PROCESS_REFUND",
    "CANCEL_SHIPMENT",
    "NOTIFY_CUSTOMER"
  ]
}
```

**Error Responses:**

- **400 Bad Request** - Order cannot be cancelled (e.g., already shipped)
- **401 Unauthorized** - Missing or invalid authentication
- **404 Not Found** - Order does not exist
- **409 Conflict** - Cancellation already in progress

### Refund Management

#### Process Refund

Process a full or partial refund for an order.

```http
POST /orders/{orderId}/refund
```

**Request Body:**

```json
{
  "type": "PARTIAL",
  "items": [
    {
      "orderItemId": "item-880e8400-e29b-41d4-a716-446655440000",
      "quantity": 1,
      "reason": "DEFECTIVE",
      "reasonDetails": "Screen has dead pixels",
      "returnRequired": true
    }
  ],
  "refundShipping": false,
  "refundTax": true,
  "adjustmentAmount": 0.00,
  "adjustmentReason": null,
  "customerNote": "Item was defective upon arrival",
  "internalNote": "Customer provided photos of defect",
  "processImmediately": true
}
```

**Success Response (201 Created):**

```json
{
  "refundId": "ref-aa0e8400-e29b-41d4-a716-446655440000",
  "orderId": "ord-770e8400-e29b-41d4-a716-446655440000",
  "type": "PARTIAL",
  "status": "PROCESSING",
  "amount": {
    "subtotal": 599.99,
    "tax": 48.00,
    "shipping": 0.00,
    "adjustment": 0.00,
    "total": 647.99,
    "currency": "USD"
  },
  "items": [
    {
      "orderItemId": "item-880e8400-e29b-41d4-a716-446655440000",
      "productName": "Premium Laptop Pro 15",
      "productSku": "LAPTOP-001",
      "quantity": 1,
      "unitPrice": 599.99,
      "refundAmount": 599.99,
      "reason": "DEFECTIVE",
      "returnRequired": true,
      "returnLabel": {
        "trackingNumber": "RL123456789",
        "carrier": "UPS",
        "url": "https://returns.firstviscount.com/label/RL123456789"
      }
    }
  ],
  "paymentMethod": "ORIGINAL_PAYMENT_METHOD",
  "transactionId": "rf_1234567890",
  "estimatedCompletionDate": "2024-01-20",
  "processedAt": "2024-01-17T10:00:00Z",
  "processedBy": "admin-123"
}
```

**Error Responses:**

- **400 Bad Request** - Invalid refund request
- **401 Unauthorized** - Missing or invalid authentication
- **403 Forbidden** - Insufficient permissions
- **404 Not Found** - Order or items not found
- **409 Conflict** - Refund already processed
- **422 Unprocessable Entity** - Refund amount exceeds order total

#### Get Refund Status

Check the status of a refund.

```http
GET /orders/{orderId}/refunds/{refundId}
```

**Success Response (200 OK):**

```json
{
  "refundId": "ref-aa0e8400-e29b-41d4-a716-446655440000",
  "orderId": "ord-770e8400-e29b-41d4-a716-446655440000",
  "status": "COMPLETED",
  "amount": {
    "requested": 647.99,
    "processed": 647.99,
    "currency": "USD"
  },
  "timeline": {
    "requested": "2024-01-17T10:00:00Z",
    "approved": "2024-01-17T10:05:00Z",
    "processing": "2024-01-17T10:10:00Z",
    "completed": "2024-01-19T14:00:00Z"
  },
  "paymentDetails": {
    "method": "CREDIT_CARD",
    "lastFour": "4242",
    "transactionId": "rf_1234567890",
    "gatewayResponse": {
      "status": "succeeded",
      "processedAt": "2024-01-19T14:00:00Z"
    }
  }
}
```

### Search and Analytics

#### Search Orders (Admin Only)

Search orders with advanced filtering.

```http
POST /orders/search
```

**Request Body:**

```json
{
  "filters": {
    "status": ["PROCESSING", "SHIPPED"],
    "dateRange": {
      "from": "2024-01-01",
      "to": "2024-01-31"
    },
    "totalRange": {
      "min": 100,
      "max": 1000
    },
    "customerEmail": "john*",
    "productIds": ["550e8400-e29b-41d4-a716-446655440000"],
    "shippingCountry": "US",
    "paymentMethod": "CREDIT_CARD",
    "hasRefunds": false,
    "tags": ["vip_customer"]
  },
  "page": 0,
  "size": 50,
  "sort": {
    "field": "createdAt",
    "direction": "DESC"
  }
}
```

**Success Response (200 OK):**

```json
{
  "results": [
    {
      "orderId": "ord-770e8400-e29b-41d4-a716-446655440000",
      "orderNumber": "ORD-2024-001234",
      "status": "PROCESSING",
      "customer": {
        "id": "123e4567-e89b-12d3-a456-426614174000",
        "name": "John Doe",
        "email": "john.doe@example.com"
      },
      "total": 1469.98,
      "itemCount": 2,
      "createdAt": "2024-01-15T10:00:00Z",
      "tags": ["vip_customer", "expedite"]
    }
  ],
  "pagination": {
    "page": 0,
    "size": 50,
    "totalElements": 234,
    "totalPages": 5
  },
  "aggregations": {
    "totalRevenue": 125430.50,
    "averageOrderValue": 536.00,
    "orderCount": 234,
    "statusBreakdown": {
      "PROCESSING": 45,
      "SHIPPED": 189
    },
    "topProducts": [
      {
        "productId": "550e8400-e29b-41d4-a716-446655440000",
        "name": "Premium Laptop Pro 15",
        "orderCount": 67,
        "revenue": 40193.33
      }
    ],
    "revenueByDay": [
      {
        "date": "2024-01-15",
        "revenue": 12340.50,
        "orderCount": 23
      }
    ]
  }
}
```

#### Get Order Statistics

Get statistical data about orders.

```http
GET /orders/statistics
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| period | string | No | Time period (TODAY, WEEK, MONTH, YEAR) |
| groupBy | string | No | Group results by (day, week, month) |

**Success Response (200 OK):**

```json
{
  "period": "MONTH",
  "summary": {
    "totalOrders": 1234,
    "totalRevenue": 567890.12,
    "averageOrderValue": 460.29,
    "conversionRate": 3.45,
    "repeatCustomerRate": 23.5
  },
  "trends": {
    "ordersGrowth": 12.5,
    "revenueGrowth": 15.8,
    "aovGrowth": 3.2
  },
  "breakdown": {
    "byStatus": {
      "COMPLETED": 1100,
      "PROCESSING": 89,
      "SHIPPED": 45
    },
    "byPaymentMethod": {
      "CREDIT_CARD": 890,
      "PAYPAL": 234,
      "APPLE_PAY": 110
    },
    "byShippingMethod": {
      "STANDARD": 678,
      "EXPRESS": 456,
      "OVERNIGHT": 100
    }
  },
  "topMetrics": {
    "peakOrderHour": 14,
    "peakOrderDay": "FRIDAY",
    "mostPopularCategory": "Electronics",
    "highestRevenueCountry": "US"
  }
}
```

## Order Status Flow

### Status Transitions

```mermaid
graph LR
    CREATED --> PENDING_PAYMENT
    PENDING_PAYMENT --> PAID
    PENDING_PAYMENT --> CANCELLED
    PAID --> PROCESSING
    PAID --> CANCELLED
    PAID --> REFUNDED
    PROCESSING --> SHIPPED
    PROCESSING --> CANCELLED
    SHIPPED --> DELIVERED
    SHIPPED --> RETURNED
    DELIVERED --> COMPLETED
    DELIVERED --> REFUNDED
    COMPLETED --> REFUNDED
```

### Status Descriptions

| Status | Description | Customer Visible |
|--------|-------------|------------------|
| CREATED | Order created but not yet submitted | No |
| PENDING_PAYMENT | Awaiting payment processing | Yes |
| PAID | Payment successfully processed | Yes |
| PROCESSING | Order being prepared for shipment | Yes |
| SHIPPED | Order dispatched from warehouse | Yes |
| DELIVERED | Order delivered to customer | Yes |
| COMPLETED | Order finalized (after return window) | Yes |
| CANCELLED | Order cancelled | Yes |
| REFUNDED | Order refunded (full or partial) | Yes |
| RETURNED | Order returned by customer | Yes |

## Error Handling

### Error Response Format

```json
{
  "error": {
    "code": "INSUFFICIENT_STOCK",
    "message": "Product LAPTOP-001 has insufficient stock. Available: 5, Requested: 10",
    "details": {
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "availableQuantity": 5,
      "requestedQuantity": 10
    },
    "traceId": "b9d7c9a8-3e2f-4d1e-8c7b-6a5d4c3b2a1f",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

### Error Codes

| Code | Description | HTTP Status |
|------|-------------|-------------|
| INVALID_ORDER_DATA | Order validation failed | 400 |
| AUTHENTICATION_REQUIRED | Missing authentication | 401 |
| PAYMENT_REQUIRED | Payment validation failed | 402 |
| INSUFFICIENT_PERMISSIONS | User lacks permissions | 403 |
| ORDER_NOT_FOUND | Order does not exist | 404 |
| DUPLICATE_ORDER | Idempotency key conflict | 409 |
| INVALID_STATUS_TRANSITION | Invalid status change | 400 |
| INSUFFICIENT_STOCK | Product out of stock | 422 |
| PRICE_MISMATCH | Product price changed | 422 |
| PAYMENT_FAILED | Payment processing failed | 422 |
| CANCELLATION_NOT_ALLOWED | Cannot cancel order | 400 |
| REFUND_EXCEEDS_TOTAL | Refund amount too high | 422 |
| RATE_LIMIT_EXCEEDED | Too many requests | 429 |
| INTERNAL_ERROR | Server error | 500 |

## Webhooks

The Order Management service sends webhooks for order events:

### Event Types

- `order.created` - New order created
- `order.paid` - Payment successfully processed
- `order.processing` - Order sent to fulfillment
- `order.shipped` - Order shipped
- `order.delivered` - Order delivered
- `order.completed` - Order completed
- `order.cancelled` - Order cancelled
- `order.refunded` - Refund processed

### Webhook Payload

```json
{
  "id": "webhook-001",
  "timestamp": "2024-01-15T10:30:00Z",
  "event": "order.shipped",
  "data": {
    "orderId": "ord-770e8400-e29b-41d4-a716-446655440000",
    "orderNumber": "ORD-2024-001234",
    "customerId": "123e4567-e89b-12d3-a456-426614174000",
    "status": "SHIPPED",
    "trackingNumber": "1Z999AA10123456784",
    "carrier": "UPS",
    "estimatedDelivery": "2024-01-22"
  },
  "signature": "sha256=abcdef123456..."
}
```

### Webhook Security

Verify webhook signatures using HMAC-SHA256:

```javascript
const crypto = require('crypto');

function verifyWebhook(payload, signature, secret) {
  const expectedSignature = crypto
    .createHmac('sha256', secret)
    .update(payload)
    .digest('hex');
  
  return `sha256=${expectedSignature}` === signature;
}
```

## Rate Limiting

- **Standard Users:** 1000 requests per hour
- **Admin Users:** 5000 requests per hour
- **Service Accounts:** 10000 requests per hour

Rate limit headers:

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1642252800
```

## SDK Examples

### JavaScript/TypeScript

```typescript
import { OrderManagementClient } from '@firstviscount/order-sdk';

const client = new OrderManagementClient({
  baseUrl: 'https://api.firstviscount.com/orders/v1',
  apiKey: process.env.API_KEY
});

// Create order
const order = await client.orders.create({
  customerId: '123e4567-e89b-12d3-a456-426614174000',
  items: [
    {
      productId: '550e8400-e29b-41d4-a716-446655440000',
      quantity: 2
    }
  ],
  shippingAddress: { /* ... */ },
  paymentMethod: { /* ... */ }
});

// Track order
const status = await client.orders.get(order.orderId);

// Cancel order
await client.orders.cancel(order.orderId, {
  reason: 'CUSTOMER_REQUEST',
  reasonDetails: 'Changed mind'
});
```

### Python

```python
from firstviscount import OrderManagementClient

client = OrderManagementClient(
    base_url='https://api.firstviscount.com/orders/v1',
    api_key=os.environ['API_KEY']
)

# Create order
order = client.orders.create(
    customer_id='123e4567-e89b-12d3-a456-426614174000',
    items=[
        {'product_id': '550e8400-e29b-41d4-a716-446655440000', 'quantity': 2}
    ],
    shipping_address={...},
    payment_method={...}
)

# Get customer orders
orders = client.orders.list_customer_orders(
    customer_id='123e4567-e89b-12d3-a456-426614174000',
    status='COMPLETED'
)

# Process refund
refund = client.orders.refund(
    order_id=order.order_id,
    items=[
        {
            'order_item_id': 'item-880e8400-e29b-41d4-a716-446655440000',
            'quantity': 1,
            'reason': 'DEFECTIVE'
        }
    ]
)
```

### Java

```java
import com.firstviscount.orders.OrderManagementClient;
import com.firstviscount.orders.model.*;

OrderManagementClient client = OrderManagementClient.builder()
    .baseUrl("https://api.firstviscount.com/orders/v1")
    .apiKey(System.getenv("API_KEY"))
    .build();

// Create order
CreateOrderRequest request = CreateOrderRequest.builder()
    .customerId("123e4567-e89b-12d3-a456-426614174000")
    .addItem(OrderItem.builder()
        .productId("550e8400-e29b-41d4-a716-446655440000")
        .quantity(2)
        .build())
    .shippingAddress(/* ... */)
    .paymentMethod(/* ... */)
    .build();

Order order = client.orders().create(request);

// Update order status (admin)
client.orders().updateStatus(order.getOrderId(), 
    UpdateStatusRequest.builder()
        .status(OrderStatus.SHIPPED)
        .trackingNumber("1Z999AA10123456784")
        .carrier("UPS")
        .build());
```

## Testing

### Test Environment

```
Base URL: https://api-test.firstviscount.com/orders/v1
```

### Test Cards

| Card Number | Result |
|-------------|---------|
| 4242 4242 4242 4242 | Success |
| 4000 0000 0000 0002 | Declined |
| 4000 0000 0000 9995 | Insufficient funds |

### Sample Test Data

| Order ID | Status | Customer ID |
|----------|--------|-------------|
| ord-test-001 | COMPLETED | cust-test-001 |
| ord-test-002 | PROCESSING | cust-test-001 |
| ord-test-003 | CANCELLED | cust-test-002 |

## Changelog

### Version 1.0.0 (2024-01-15)

- Initial release
- Order CRUD operations
- Payment integration
- Refund management
- Order search and analytics
- Webhook support
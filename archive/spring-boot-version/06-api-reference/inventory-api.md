# Inventory API Reference

## Overview

The Inventory API manages real-time inventory levels, stock reservations, and inventory movements across all warehouses. It provides critical functionality for preventing overselling through distributed locking mechanisms and maintains accurate stock tracking.

**Base URL:** `http://inventory-service:8083/api/v1`  
**Port:** 8083

## Authentication

All endpoints require JWT authentication:

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

## OpenAPI Specification

```yaml
openapi: 3.0.3
info:
  title: Inventory Management API
  version: 1.0.0
  description: Real-time inventory tracking and reservation system
servers:
  - url: http://inventory-service:8083/api/v1
    description: Internal service endpoint
  - url: https://api.firstviscount.com/inventory/v1
    description: Production API gateway
```

## Endpoints

### Inventory Availability

#### Check Inventory Availability

Check if products are available in the requested quantities.

```http
POST /inventory/check-availability
```

**Request Headers:**

```http
Content-Type: application/json
Authorization: Bearer {token}
```

**Request Body:**

```json
{
  "items": [
    {
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "quantity": 2,
      "warehouseId": "wh-001"
    },
    {
      "productId": "660e8400-e29b-41d4-a716-446655440000",
      "quantity": 1,
      "warehousePreference": ["wh-001", "wh-002"]
    }
  ],
  "checkType": "AVAILABLE",
  "includeIncoming": false
}
```

**Request Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| items | array | Yes | Products to check |
| items[].productId | string (UUID) | Yes | Product identifier |
| items[].quantity | integer | Yes | Requested quantity |
| items[].warehouseId | string | No | Specific warehouse |
| items[].warehousePreference | array | No | Preferred warehouses in order |
| checkType | string | No | AVAILABLE or RESERVABLE |
| includeIncoming | boolean | No | Include incoming stock |

**Success Response (200 OK):**

```json
{
  "available": true,
  "items": [
    {
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "productSku": "LAPTOP-001",
      "requestedQuantity": 2,
      "availableQuantity": 45,
      "status": "AVAILABLE",
      "warehouses": [
        {
          "warehouseId": "wh-001",
          "warehouseName": "Main Warehouse",
          "availableQuantity": 20,
          "reservedQuantity": 5,
          "incomingQuantity": 0,
          "location": {
            "city": "New York",
            "state": "NY"
          }
        },
        {
          "warehouseId": "wh-002",
          "warehouseName": "Secondary Warehouse",
          "availableQuantity": 25,
          "reservedQuantity": 3,
          "incomingQuantity": 50,
          "incomingDate": "2024-01-20"
        }
      ],
      "fulfillmentOptions": [
        {
          "warehouseId": "wh-001",
          "quantity": 2,
          "shippingMethod": "STANDARD",
          "estimatedDelivery": "2024-01-18"
        }
      ]
    },
    {
      "productId": "660e8400-e29b-41d4-a716-446655440000",
      "productSku": "MOUSE-001",
      "requestedQuantity": 1,
      "availableQuantity": 0,
      "status": "OUT_OF_STOCK",
      "warehouses": [],
      "nextAvailableDate": "2024-01-25",
      "alternativeProducts": [
        {
          "productId": "770e8400-e29b-41d4-a716-446655440000",
          "productSku": "MOUSE-002",
          "availableQuantity": 15
        }
      ]
    }
  ],
  "summary": {
    "allAvailable": false,
    "partiallyAvailable": true,
    "totalRequestedItems": 2,
    "totalAvailableItems": 1
  }
}
```

**Error Responses:**

- **400 Bad Request** - Invalid request data
- **401 Unauthorized** - Missing or invalid authentication

### Inventory Reservation

#### Reserve Inventory

Create a reservation for specified products.

```http
POST /inventory/reserve
```

**Request Body:**

```json
{
  "orderId": "ord-123e4567-e89b-12d3-a456-426614174000",
  "customerId": "cust-123e4567-e89b-12d3-a456-426614174000",
  "items": [
    {
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "quantity": 2,
      "warehouseId": "wh-001",
      "unitPrice": 599.99
    }
  ],
  "timeoutMinutes": 15,
  "priority": "NORMAL",
  "metadata": {
    "source": "web_checkout",
    "sessionId": "session-123"
  }
}
```

**Request Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| orderId | string (UUID) | Yes | Associated order ID |
| customerId | string (UUID) | No | Customer identifier |
| items | array | Yes | Products to reserve |
| timeoutMinutes | integer | No | Reservation timeout (default: 15) |
| priority | string | No | NORMAL, HIGH, URGENT |
| metadata | object | No | Additional metadata |

**Success Response (201 Created):**

```json
{
  "reservationId": "res-770e8400-e29b-41d4-a716-446655440000",
  "reservationNumber": "RES-2024-001234",
  "orderId": "ord-123e4567-e89b-12d3-a456-426614174000",
  "status": "RESERVED",
  "items": [
    {
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "productSku": "LAPTOP-001",
      "quantity": 2,
      "warehouseId": "wh-001",
      "warehouseName": "Main Warehouse",
      "reserved": true,
      "location": {
        "zone": "A",
        "aisle": "12",
        "shelf": "B3"
      }
    }
  ],
  "expiresAt": "2024-01-15T10:15:00Z",
  "createdAt": "2024-01-15T10:00:00Z",
  "totalValue": 1199.98,
  "confirmationCode": "RSVP-XYZ123"
}
```

**Error Responses:**

- **400 Bad Request** - Invalid reservation data
- **409 Conflict** - Insufficient inventory
- **422 Unprocessable Entity** - Business rule violation

#### Release Inventory

Release a previously created reservation.

```http
POST /inventory/release
```

**Request Body:**

```json
{
  "reservationId": "res-770e8400-e29b-41d4-a716-446655440000",
  "reason": "ORDER_CANCELLED",
  "reasonDetails": "Customer cancelled order",
  "releaseItems": [
    {
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "quantity": 1
    }
  ]
}
```

**Request Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| reservationId | string (UUID) | Yes | Reservation to release |
| reason | string | Yes | Release reason code |
| reasonDetails | string | No | Additional details |
| releaseItems | array | No | Partial release items |

**Success Response (200 OK):**

```json
{
  "reservationId": "res-770e8400-e29b-41d4-a716-446655440000",
  "status": "PARTIALLY_RELEASED",
  "items": [
    {
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "originalQuantity": 2,
      "releasedQuantity": 1,
      "remainingQuantity": 1,
      "warehouseId": "wh-001"
    }
  ],
  "releasedAt": "2024-01-15T10:05:00Z",
  "releasedBy": "system",
  "remainingValue": 599.99
}
```

#### Confirm Reservation

Confirm a reservation when order is completed.

```http
POST /inventory/reservations/{reservationId}/confirm
```

**Success Response (200 OK):**

```json
{
  "reservationId": "res-770e8400-e29b-41d4-a716-446655440000",
  "status": "CONFIRMED",
  "confirmedAt": "2024-01-15T10:10:00Z",
  "items": [
    {
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "quantity": 2,
      "warehouseId": "wh-001",
      "allocated": true
    }
  ]
}
```

### Inventory Levels

#### Get Inventory Levels

Retrieve current inventory levels for a product.

```http
GET /inventory/{productId}
```

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| productId | string (UUID) | Yes | Product identifier |

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| warehouseId | string | No | Filter by warehouse |
| includeReserved | boolean | No | Include reserved quantity details |
| includeIncoming | boolean | No | Include incoming stock |

**Success Response (200 OK):**

```json
{
  "productId": "550e8400-e29b-41d4-a716-446655440000",
  "productSku": "LAPTOP-001",
  "productName": "Premium Laptop Pro 15",
  "summary": {
    "totalQuantity": 45,
    "totalAvailable": 40,
    "totalReserved": 5,
    "totalIncoming": 100,
    "nextIncomingDate": "2024-01-20"
  },
  "warehouses": [
    {
      "warehouseId": "wh-001",
      "warehouseName": "Main Warehouse",
      "type": "MAIN",
      "stock": {
        "quantityOnHand": 20,
        "quantityReserved": 2,
        "quantityAvailable": 18,
        "quantityIncoming": 50,
        "quantityDamaged": 0
      },
      "location": {
        "zone": "A",
        "aisle": "12",
        "shelf": "B3",
        "bin": "001"
      },
      "reorderInfo": {
        "reorderPoint": 10,
        "reorderQuantity": 50,
        "status": "ABOVE_REORDER_POINT"
      },
      "lastActivity": {
        "type": "RECEIPT",
        "quantity": 20,
        "timestamp": "2024-01-10T09:00:00Z"
      }
    },
    {
      "warehouseId": "wh-002",
      "warehouseName": "Secondary Warehouse",
      "type": "SECONDARY",
      "stock": {
        "quantityOnHand": 25,
        "quantityReserved": 3,
        "quantityAvailable": 22,
        "quantityIncoming": 50,
        "quantityDamaged": 0
      }
    }
  ],
  "movements": {
    "last24Hours": {
      "receipts": 0,
      "shipments": 5,
      "adjustments": 0,
      "transfers": 0,
      "returns": 1
    },
    "last7Days": {
      "receipts": 20,
      "shipments": 35,
      "adjustments": 2,
      "transfers": 5,
      "returns": 3
    }
  },
  "reservations": [
    {
      "reservationId": "res-123",
      "orderId": "ord-456",
      "quantity": 2,
      "expiresAt": "2024-01-15T11:00:00Z"
    }
  ],
  "analytics": {
    "averageDailySales": 5.2,
    "daysOfSupply": 8.7,
    "turnoverRate": 42.1,
    "stockoutRisk": "LOW"
  },
  "lastUpdated": "2024-01-15T09:45:00Z"
}
```

#### Get Multiple Product Inventory

Retrieve inventory levels for multiple products.

```http
POST /inventory/batch
```

**Request Body:**

```json
{
  "productIds": [
    "550e8400-e29b-41d4-a716-446655440000",
    "660e8400-e29b-41d4-a716-446655440000"
  ],
  "warehouseIds": ["wh-001"],
  "includeZeroStock": false
}
```

**Success Response (200 OK):**

```json
{
  "products": [
    {
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "totalAvailable": 18,
      "warehouses": [
        {
          "warehouseId": "wh-001",
          "available": 18
        }
      ]
    }
  ]
}
```

### Inventory Management

#### Adjust Inventory (Admin Only)

Make inventory adjustments for receipts, counts, or corrections.

```http
PUT /inventory/adjust
```

**Request Body:**

```json
{
  "adjustments": [
    {
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "warehouseId": "wh-001",
      "quantity": 10,
      "type": "RECEIPT",
      "reason": "PURCHASE_ORDER",
      "reasonDetails": "New shipment received from supplier",
      "reference": "PO-12345",
      "cost": 499.99,
      "lotNumber": "LOT-2024-001",
      "expirationDate": "2025-01-15"
    }
  ],
  "notifyLowStock": true
}
```

**Adjustment Types:**

| Type | Description | Quantity |
|------|-------------|----------|
| RECEIPT | Incoming stock | Positive |
| SHIPMENT | Outgoing stock | Negative |
| ADJUSTMENT | Manual adjustment | Positive/Negative |
| DAMAGE | Damaged goods | Negative |
| RETURN | Customer return | Positive |
| TRANSFER_IN | Transfer receipt | Positive |
| TRANSFER_OUT | Transfer shipment | Negative |

**Success Response (200 OK):**

```json
{
  "adjustmentId": "adj-880e8400-e29b-41d4-a716-446655440000",
  "adjustmentNumber": "ADJ-2024-001234",
  "status": "COMPLETED",
  "adjustments": [
    {
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "productSku": "LAPTOP-001",
      "warehouseId": "wh-001",
      "type": "RECEIPT",
      "previousQuantity": 20,
      "adjustedQuantity": 10,
      "newQuantity": 30,
      "cost": 499.99,
      "totalValue": 4999.90
    }
  ],
  "summary": {
    "totalItems": 1,
    "totalQuantity": 10,
    "totalValue": 4999.90
  },
  "adjustedAt": "2024-01-15T10:30:00Z",
  "adjustedBy": "admin-123"
}
```

#### Transfer Inventory

Transfer inventory between warehouses.

```http
POST /inventory/transfer
```

**Request Body:**

```json
{
  "items": [
    {
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "quantity": 10,
      "fromWarehouseId": "wh-001",
      "toWarehouseId": "wh-002"
    }
  ],
  "reason": "STOCK_BALANCING",
  "priority": "NORMAL",
  "expectedDelivery": "2024-01-17T10:00:00Z",
  "shippingMethod": "GROUND",
  "notes": "Balancing inventory levels between warehouses"
}
```

**Success Response (201 Created):**

```json
{
  "transferId": "tr-bb0e8400-e29b-41d4-a716-446655440000",
  "transferNumber": "TRF-2024-001234",
  "status": "PENDING",
  "items": [
    {
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "productSku": "LAPTOP-001",
      "quantity": 10,
      "fromWarehouse": {
        "id": "wh-001",
        "name": "Main Warehouse",
        "previousQuantity": 30,
        "newQuantity": 20,
        "reserved": true
      },
      "toWarehouse": {
        "id": "wh-002",
        "name": "Secondary Warehouse",
        "currentQuantity": 25,
        "expectedQuantity": 35
      }
    }
  ],
  "timeline": {
    "initiated": "2024-01-15T11:00:00Z",
    "expectedShipment": "2024-01-16T08:00:00Z",
    "expectedDelivery": "2024-01-17T10:00:00Z"
  },
  "initiatedBy": "admin-123"
}
```

#### Complete Transfer

Mark a transfer as received.

```http
POST /inventory/transfers/{transferId}/complete
```

**Request Body:**

```json
{
  "receivedItems": [
    {
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "quantityReceived": 10,
      "condition": "GOOD",
      "discrepancyNotes": null
    }
  ],
  "receivedBy": "warehouse-staff-001"
}
```

### Inventory Analytics

#### Get Low Stock Items

Retrieve products below reorder threshold.

```http
GET /inventory/low-stock
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| warehouseId | string | No | Filter by warehouse |
| threshold | number | No | Threshold percentage (0-1) |
| category | string | No | Filter by product category |
| limit | integer | No | Result limit |

**Success Response (200 OK):**

```json
{
  "items": [
    {
      "productId": "990e8400-e29b-41d4-a716-446655440000",
      "productSku": "WIDGET-001",
      "productName": "Widget XL",
      "category": "Widgets",
      "warehouseId": "wh-001",
      "warehouseName": "Main Warehouse",
      "stock": {
        "current": 5,
        "reorderPoint": 20,
        "reorderQuantity": 50,
        "available": 3,
        "reserved": 2
      },
      "metrics": {
        "percentageRemaining": 25.0,
        "daysOfSupply": 2.1,
        "averageDailySales": 2.4,
        "lastRestocked": "2024-01-01T00:00:00Z"
      },
      "status": "CRITICAL",
      "suggestedAction": "URGENT_REORDER",
      "supplier": {
        "id": "sup-123",
        "name": "Widget Supplier Inc",
        "leadTime": 7
      }
    }
  ],
  "summary": {
    "totalItems": 12,
    "criticalItems": 3,
    "lowItems": 9,
    "estimatedValue": 45678.90
  },
  "recommendations": [
    {
      "action": "CREATE_PURCHASE_ORDER",
      "supplier": "sup-123",
      "items": 5,
      "estimatedCost": 12345.67
    }
  ]
}
```

#### Get Inventory Movements

Retrieve inventory movement history.

```http
GET /inventory/movements
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| productId | string | No | Filter by product |
| warehouseId | string | No | Filter by warehouse |
| type | string | No | Movement type filter |
| dateFrom | date | No | Start date |
| dateTo | date | No | End date |
| page | integer | No | Page number |
| size | integer | No | Page size |

**Success Response (200 OK):**

```json
{
  "movements": [
    {
      "movementId": "mov-123",
      "movementNumber": "MOV-2024-001234",
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "productSku": "LAPTOP-001",
      "warehouseId": "wh-001",
      "type": "SHIPMENT",
      "quantity": -2,
      "quantityBefore": 32,
      "quantityAfter": 30,
      "reference": {
        "type": "ORDER",
        "id": "ord-456",
        "number": "ORD-2024-001234"
      },
      "cost": 499.99,
      "totalValue": -999.98,
      "createdAt": "2024-01-15T08:00:00Z",
      "createdBy": "system"
    }
  ],
  "summary": {
    "totalMovements": 156,
    "netChange": -25,
    "totalValue": -12499.75
  },
  "pagination": {
    "page": 0,
    "size": 20,
    "totalElements": 156,
    "totalPages": 8
  }
}
```

#### Get Inventory Valuation

Calculate inventory value.

```http
GET /inventory/valuation
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| warehouseId | string | No | Filter by warehouse |
| method | string | No | FIFO, LIFO, AVERAGE |

**Success Response (200 OK):**

```json
{
  "valuation": {
    "method": "AVERAGE",
    "asOf": "2024-01-15T12:00:00Z",
    "totalValue": 1234567.89,
    "currency": "USD",
    "breakdown": {
      "byWarehouse": [
        {
          "warehouseId": "wh-001",
          "warehouseName": "Main Warehouse",
          "value": 654321.00,
          "itemCount": 1234
        }
      ],
      "byCategory": [
        {
          "category": "Electronics",
          "value": 890123.45,
          "percentage": 72.1
        }
      ]
    },
    "topValueItems": [
      {
        "productId": "550e8400-e29b-41d4-a716-446655440000",
        "productSku": "LAPTOP-001",
        "quantity": 45,
        "unitCost": 499.99,
        "totalValue": 22499.55
      }
    ]
  }
}
```

## Error Handling

### Error Response Format

```json
{
  "error": {
    "code": "INSUFFICIENT_INVENTORY",
    "message": "Insufficient inventory for product LAPTOP-001. Available: 5, Requested: 10",
    "details": {
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "warehouseId": "wh-001",
      "available": 5,
      "requested": 10,
      "reserved": 2
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
| INSUFFICIENT_PERMISSIONS | User lacks permissions | 403 |
| PRODUCT_NOT_FOUND | Product does not exist | 404 |
| WAREHOUSE_NOT_FOUND | Warehouse does not exist | 404 |
| INSUFFICIENT_INVENTORY | Not enough stock | 409 |
| RESERVATION_NOT_FOUND | Reservation does not exist | 404 |
| RESERVATION_EXPIRED | Reservation has expired | 410 |
| INVALID_ADJUSTMENT | Invalid adjustment quantity | 400 |
| TRANSFER_IN_PROGRESS | Transfer already in progress | 409 |
| OPTIMISTIC_LOCK_ERROR | Concurrent update conflict | 409 |
| RATE_LIMIT_EXCEEDED | Too many requests | 429 |
| INTERNAL_ERROR | Server error | 500 |

## Webhooks

### Event Types

- `inventory.reserved` - Inventory reserved
- `inventory.released` - Inventory released
- `inventory.adjusted` - Stock adjusted
- `inventory.low_stock` - Low stock alert
- `inventory.transfer.initiated` - Transfer started
- `inventory.transfer.completed` - Transfer completed

### Webhook Payload

```json
{
  "id": "webhook-001",
  "timestamp": "2024-01-15T10:30:00Z",
  "event": "inventory.low_stock",
  "data": {
    "productId": "990e8400-e29b-41d4-a716-446655440000",
    "productSku": "WIDGET-001",
    "warehouseId": "wh-001",
    "currentStock": 5,
    "reorderPoint": 20,
    "severity": "CRITICAL"
  },
  "signature": "sha256=abcdef123456..."
}
```

## Rate Limiting

- **Standard Users:** 1000 requests per hour
- **Admin Users:** 5000 requests per hour
- **Service Accounts:** 10000 requests per hour

## SDK Examples

### JavaScript/TypeScript

```typescript
import { InventoryClient } from '@firstviscount/inventory-sdk';

const client = new InventoryClient({
  baseUrl: 'https://api.firstviscount.com/inventory/v1',
  apiKey: process.env.API_KEY
});

// Check availability
const availability = await client.checkAvailability({
  items: [
    { productId: '550e8400-e29b-41d4-a716-446655440000', quantity: 2 }
  ]
});

// Reserve inventory
const reservation = await client.reserve({
  orderId: 'ord-123',
  items: [
    { 
      productId: '550e8400-e29b-41d4-a716-446655440000', 
      quantity: 2,
      warehouseId: 'wh-001'
    }
  ]
});

// Release reservation
await client.release({
  reservationId: reservation.reservationId,
  reason: 'ORDER_CANCELLED'
});
```

### Python

```python
from firstviscount import InventoryClient

client = InventoryClient(
    base_url='https://api.firstviscount.com/inventory/v1',
    api_key=os.environ['API_KEY']
)

# Check multiple products
availability = client.check_availability(
    items=[
        {'product_id': '550e8400-e29b-41d4-a716-446655440000', 'quantity': 2},
        {'product_id': '660e8400-e29b-41d4-a716-446655440000', 'quantity': 1}
    ]
)

# Get inventory levels
inventory = client.get_inventory('550e8400-e29b-41d4-a716-446655440000')

# Adjust inventory (admin)
adjustment = client.adjust_inventory(
    adjustments=[{
        'product_id': '550e8400-e29b-41d4-a716-446655440000',
        'warehouse_id': 'wh-001',
        'quantity': 10,
        'type': 'RECEIPT',
        'reason': 'PURCHASE_ORDER'
    }]
)
```

## Testing

### Test Environment

```
Base URL: https://api-test.firstviscount.com/inventory/v1
```

### Test Data

| Product ID | SKU | Available Stock |
|------------|-----|-----------------|
| test-product-001 | TEST-001 | 100 |
| test-product-002 | TEST-002 | 0 |
| test-product-003 | TEST-003 | 5 |

## Changelog

### Version 1.0.0 (2024-01-15)

- Initial release
- Inventory tracking and reservations
- Stock adjustments and transfers
- Low stock alerts
- Movement history
- Webhook support
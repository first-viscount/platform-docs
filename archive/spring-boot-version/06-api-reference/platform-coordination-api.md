# Platform Coordination API Reference

## Overview

The Platform Coordination API serves as the orchestration brain of the First Viscount e-commerce platform. It manages complex workflows, coordinates distributed transactions using the Saga pattern, and ensures business processes complete successfully or are properly compensated in case of failures.

**Base URL:** `http://platform-coordination:8081/api/v1`  
**Port:** 8081

## Authentication

All endpoints require JWT authentication:

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

## OpenAPI Specification

```yaml
openapi: 3.0.3
info:
  title: Platform Coordination API
  version: 1.0.0
  description: Workflow orchestration and saga management for distributed transactions
servers:
  - url: http://platform-coordination:8081/api/v1
    description: Internal service endpoint
  - url: https://api.firstviscount.com/coordination/v1
    description: Production API gateway
```

## Endpoints

### Workflow Management

#### Start Order Workflow

Initiate a complete order fulfillment workflow.

```http
POST /workflows/order/start
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
  "orderId": "ord-550e8400-e29b-41d4-a716-446655440000",
  "customerId": "cust-123e4567-e89b-12d3-a456-426614174000",
  "items": [
    {
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "quantity": 2,
      "unitPrice": 599.99,
      "warehouseId": "wh-001"
    },
    {
      "productId": "660e8400-e29b-41d4-a716-446655440000",
      "quantity": 1,
      "unitPrice": 299.99,
      "warehouseId": "wh-001"
    }
  ],
  "paymentMethod": {
    "type": "CREDIT_CARD",
    "token": "tok_visa_4242",
    "amount": 1499.97,
    "currency": "USD"
  },
  "shippingAddress": {
    "name": "John Doe",
    "street": "123 Main St",
    "city": "New York",
    "state": "NY",
    "zipCode": "10001",
    "country": "US",
    "phone": "+1234567890"
  },
  "shippingMethod": "STANDARD",
  "options": {
    "priority": "NORMAL",
    "requireSignature": true,
    "giftWrap": false,
    "expressProcessing": false
  },
  "metadata": {
    "source": "web_checkout",
    "sessionId": "session-123",
    "campaignId": "winter-sale-2024"
  }
}
```

**Request Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| orderId | string (UUID) | Yes | Order identifier |
| customerId | string (UUID) | Yes | Customer identifier |
| items | array | Yes | Order items |
| paymentMethod | object | Yes | Payment information |
| shippingAddress | object | Yes | Delivery address |
| shippingMethod | string | Yes | Shipping method |
| options | object | No | Additional options |
| metadata | object | No | Additional metadata |

**Success Response (201 Created):**

```json
{
  "workflowId": "wf-660e8400-e29b-41d4-a716-446655440000",
  "workflowType": "ORDER_FULFILLMENT",
  "status": "STARTED",
  "orderId": "ord-550e8400-e29b-41d4-a716-446655440000",
  "customerId": "cust-123e4567-e89b-12d3-a456-426614174000",
  "currentStep": "INVENTORY_RESERVATION",
  "estimatedCompletion": "2024-01-15T10:30:00Z",
  "steps": [
    {
      "name": "INVENTORY_RESERVATION",
      "status": "IN_PROGRESS",
      "description": "Reserving inventory for order items",
      "startedAt": "2024-01-15T10:00:00Z",
      "estimatedDuration": "PT30S"
    },
    {
      "name": "PAYMENT_PROCESSING",
      "status": "PENDING",
      "description": "Processing payment authorization"
    },
    {
      "name": "ORDER_CONFIRMATION",
      "status": "PENDING",
      "description": "Confirming order details"
    },
    {
      "name": "FULFILLMENT_PREPARATION",
      "status": "PENDING",
      "description": "Preparing order for fulfillment"
    },
    {
      "name": "SHIPPING_LABEL_CREATION",
      "status": "PENDING",
      "description": "Creating shipping labels"
    },
    {
      "name": "CUSTOMER_NOTIFICATION",
      "status": "PENDING",
      "description": "Notifying customer of order confirmation"
    }
  ],
  "saga": {
    "sagaId": "saga-770e8400-e29b-41d4-a716-446655440000",
    "state": "ACTIVE",
    "compensationPlan": [
      "RELEASE_INVENTORY",
      "VOID_PAYMENT",
      "CANCEL_SHIPMENT"
    ]
  },
  "timeline": {
    "started": "2024-01-15T10:00:00Z",
    "lastUpdate": "2024-01-15T10:00:00Z"
  },
  "createdAt": "2024-01-15T10:00:00Z"
}
```

**Error Responses:**

- **400 Bad Request** - Invalid workflow data
- **401 Unauthorized** - Missing or invalid authentication
- **409 Conflict** - Workflow already exists (idempotency key conflict)
- **422 Unprocessable Entity** - Business rule violation

#### Get Workflow Status

Retrieve the current status and progress of a workflow.

```http
GET /workflows/{workflowId}/status
```

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| workflowId | string (UUID) | Yes | Workflow identifier |

**Success Response (200 OK):**

```json
{
  "workflowId": "wf-660e8400-e29b-41d4-a716-446655440000",
  "workflowType": "ORDER_FULFILLMENT",
  "status": "IN_PROGRESS",
  "currentStep": "PAYMENT_PROCESSING",
  "progress": {
    "completedSteps": 1,
    "totalSteps": 6,
    "percentage": 16.67
  },
  "steps": [
    {
      "name": "INVENTORY_RESERVATION",
      "status": "COMPLETED",
      "description": "Reserving inventory for order items",
      "startedAt": "2024-01-15T10:00:00Z",
      "completedAt": "2024-01-15T10:00:05Z",
      "duration": "PT5S",
      "result": {
        "reservationId": "res-880e8400-e29b-41d4-a716-446655440000",
        "itemsReserved": 3,
        "warehousesUsed": ["wh-001"]
      }
    },
    {
      "name": "PAYMENT_PROCESSING",
      "status": "IN_PROGRESS",
      "description": "Processing payment authorization",
      "startedAt": "2024-01-15T10:00:06Z",
      "estimatedCompletion": "2024-01-15T10:00:36Z",
      "attempts": 1,
      "provider": "stripe"
    },
    {
      "name": "ORDER_CONFIRMATION",
      "status": "PENDING",
      "description": "Confirming order details"
    },
    {
      "name": "FULFILLMENT_PREPARATION",
      "status": "PENDING",
      "description": "Preparing order for fulfillment"
    },
    {
      "name": "SHIPPING_LABEL_CREATION",
      "status": "PENDING",
      "description": "Creating shipping labels"
    },
    {
      "name": "CUSTOMER_NOTIFICATION",
      "status": "PENDING",
      "description": "Notifying customer of order confirmation"
    }
  ],
  "saga": {
    "sagaId": "saga-770e8400-e29b-41d4-a716-446655440000",
    "state": "ACTIVE",
    "version": 2,
    "compensationExecuted": []
  },
  "metadata": {
    "orderId": "ord-550e8400-e29b-41d4-a716-446655440000",
    "customerId": "cust-123e4567-e89b-12d3-a456-426614174000",
    "totalAmount": 1499.97,
    "priority": "NORMAL"
  },
  "events": [
    {
      "timestamp": "2024-01-15T10:00:00Z",
      "event": "WORKFLOW_STARTED",
      "details": "Order fulfillment workflow initiated"
    },
    {
      "timestamp": "2024-01-15T10:00:05Z",
      "event": "STEP_COMPLETED",
      "step": "INVENTORY_RESERVATION",
      "details": "Successfully reserved inventory for all items"
    },
    {
      "timestamp": "2024-01-15T10:00:06Z",
      "event": "STEP_STARTED",
      "step": "PAYMENT_PROCESSING",
      "details": "Payment processing initiated with Stripe"
    }
  ],
  "estimatedCompletion": "2024-01-15T10:05:00Z",
  "createdAt": "2024-01-15T10:00:00Z",
  "updatedAt": "2024-01-15T10:00:06Z"
}
```

#### Cancel Workflow

Cancel an active workflow and trigger compensation.

```http
POST /workflows/{workflowId}/cancel
```

**Request Body:**

```json
{
  "reason": "CUSTOMER_REQUESTED",
  "reasonDetails": "Customer cancelled order after placing",
  "initiatedBy": "CUSTOMER",
  "force": false,
  "metadata": {
    "userId": "user-123",
    "source": "customer_portal"
  }
}
```

**Request Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| reason | string | Yes | Cancellation reason code |
| reasonDetails | string | No | Additional details |
| initiatedBy | string | Yes | Who initiated cancellation |
| force | boolean | No | Force cancellation even if in progress |
| metadata | object | No | Additional metadata |

**Success Response (202 Accepted):**

```json
{
  "workflowId": "wf-660e8400-e29b-41d4-a716-446655440000",
  "status": "CANCELLING",
  "cancelledAt": "2024-01-15T10:05:00Z",
  "reason": "CUSTOMER_REQUESTED",
  "saga": {
    "sagaId": "saga-770e8400-e29b-41d4-a716-446655440000",
    "state": "COMPENSATING",
    "compensationSteps": [
      {
        "step": "RELEASE_INVENTORY",
        "status": "PENDING",
        "description": "Release reserved inventory"
      },
      {
        "step": "VOID_PAYMENT",
        "status": "PENDING",
        "description": "Void payment authorization"
      },
      {
        "step": "CANCEL_SHIPMENT",
        "status": "NOT_APPLICABLE",
        "description": "Cancel shipping label (if created)"
      },
      {
        "step": "NOTIFY_CUSTOMER",
        "status": "PENDING",
        "description": "Send cancellation notification"
      }
    ]
  },
  "estimatedCompensationTime": "PT2M",
  "refund": {
    "eligible": true,
    "amount": 1499.97,
    "currency": "USD",
    "method": "ORIGINAL_PAYMENT_METHOD"
  }
}
```

#### Retry Failed Workflow

Retry a failed workflow from the last successful step.

```http
POST /workflows/{workflowId}/retry
```

**Request Body:**

```json
{
  "fromStep": "PAYMENT_PROCESSING",
  "resetState": false,
  "metadata": {
    "retryReason": "Payment provider timeout resolved",
    "initiatedBy": "admin-user-123"
  }
}
```

**Success Response (200 OK):**

```json
{
  "workflowId": "wf-660e8400-e29b-41d4-a716-446655440000",
  "status": "IN_PROGRESS",
  "retryAttempt": 2,
  "resumedAt": "2024-01-15T10:10:00Z",
  "currentStep": "PAYMENT_PROCESSING"
}
```

### Workflow Queries

#### List Active Workflows

Retrieve a paginated list of active workflows.

```http
GET /workflows/active
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| type | string | No | Filter by workflow type |
| status | string | No | Filter by status |
| customerId | string | No | Filter by customer |
| priority | string | No | Filter by priority |
| page | integer | No | Page number (0-based) |
| size | integer | No | Page size (max 100) |
| sort | string | No | Sort criteria |

**Success Response (200 OK):**

```json
{
  "workflows": [
    {
      "workflowId": "wf-660e8400-e29b-41d4-a716-446655440000",
      "workflowType": "ORDER_FULFILLMENT",
      "status": "IN_PROGRESS",
      "currentStep": "PAYMENT_PROCESSING",
      "progress": 16.67,
      "startedAt": "2024-01-15T10:00:00Z",
      "estimatedCompletion": "2024-01-15T10:05:00Z",
      "metadata": {
        "orderId": "ord-550e8400-e29b-41d4-a716-446655440000",
        "customerId": "cust-123e4567-e89b-12d3-a456-426614174000",
        "totalAmount": 1499.97,
        "priority": "NORMAL"
      }
    },
    {
      "workflowId": "wf-770e8400-e29b-41d4-a716-446655440000",
      "workflowType": "RETURN_PROCESSING",
      "status": "IN_PROGRESS",
      "currentStep": "REFUND_PROCESSING",
      "progress": 75.0,
      "startedAt": "2024-01-15T09:30:00Z",
      "estimatedCompletion": "2024-01-15T10:00:00Z",
      "metadata": {
        "returnId": "ret-880e8400-e29b-41d4-a716-446655440000",
        "originalOrderId": "ord-990e8400-e29b-41d4-a716-446655440000",
        "refundAmount": 299.99
      }
    }
  ],
  "summary": {
    "totalActive": 45,
    "byType": {
      "ORDER_FULFILLMENT": 40,
      "RETURN_PROCESSING": 3,
      "INVENTORY_REPLENISHMENT": 2
    },
    "byStatus": {
      "IN_PROGRESS": 43,
      "RETRYING": 2
    },
    "averageExecutionTime": "PT3M15S"
  },
  "pagination": {
    "page": 0,
    "size": 20,
    "totalElements": 45,
    "totalPages": 3
  }
}
```

#### Get Workflow Metrics

Retrieve workflow execution metrics and performance data.

```http
GET /workflows/metrics
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| period | string | No | Time period (TODAY, WEEK, MONTH) |
| type | string | No | Filter by workflow type |
| includeHistorical | boolean | No | Include historical data |

**Success Response (200 OK):**

```json
{
  "period": {
    "start": "2024-01-01T00:00:00Z",
    "end": "2024-01-31T23:59:59Z",
    "description": "January 2024"
  },
  "overview": {
    "totalWorkflows": 12543,
    "completed": 11234,
    "failed": 211,
    "cancelled": 1098,
    "successRate": 89.56,
    "averageExecutionTime": "PT2M30S"
  },
  "workflowTypes": {
    "ORDER_FULFILLMENT": {
      "total": 10234,
      "completed": 9567,
      "failed": 123,
      "cancelled": 544,
      "successRate": 93.48,
      "averageExecutionTime": "PT2M15S"
    },
    "RETURN_PROCESSING": {
      "total": 1456,
      "completed": 1389,
      "failed": 34,
      "cancelled": 33,
      "successRate": 95.40,
      "averageExecutionTime": "PT4M30S"
    },
    "INVENTORY_REPLENISHMENT": {
      "total": 853,
      "completed": 801,
      "failed": 52,
      "cancelled": 0,
      "successRate": 93.91,
      "averageExecutionTime": "PT15M"
    }
  },
  "stepPerformance": [
    {
      "step": "INVENTORY_RESERVATION",
      "averageExecutionTime": "PT5S",
      "successRate": 99.2,
      "commonFailures": ["INSUFFICIENT_STOCK", "SYSTEM_TIMEOUT"]
    },
    {
      "step": "PAYMENT_PROCESSING",
      "averageExecutionTime": "PT30S",
      "successRate": 95.8,
      "commonFailures": ["PAYMENT_DECLINED", "PROVIDER_TIMEOUT"]
    }
  ],
  "compensation": {
    "totalCompensations": 1309,
    "successful": 1287,
    "failed": 22,
    "averageCompensationTime": "PT1M45S",
    "mostCommonReasons": [
      {"reason": "PAYMENT_FAILED", "count": 423},
      {"reason": "INVENTORY_UNAVAILABLE", "count": 234},
      {"reason": "CUSTOMER_CANCELLED", "count": 652}
    ]
  },
  "trends": {
    "daily": [
      {
        "date": "2024-01-01",
        "workflows": 423,
        "successRate": 92.1,
        "averageTime": "PT2M45S"
      }
    ],
    "hourly": [
      {
        "hour": 9,
        "workflows": 156,
        "successRate": 94.2,
        "averageTime": "PT2M15S"
      }
    ]
  }
}
```

### Saga Management

#### Get Saga Details

Retrieve detailed information about a saga instance.

```http
GET /sagas/{sagaId}
```

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| sagaId | string (UUID) | Yes | Saga identifier |

**Success Response (200 OK):**

```json
{
  "sagaId": "saga-770e8400-e29b-41d4-a716-446655440000",
  "sagaType": "ORDER_FULFILLMENT_SAGA",
  "workflowId": "wf-660e8400-e29b-41d4-a716-446655440000",
  "state": "ACTIVE",
  "version": 3,
  "currentStep": "PAYMENT_PROCESSING",
  "context": {
    "orderId": "ord-550e8400-e29b-41d4-a716-446655440000",
    "customerId": "cust-123e4567-e89b-12d3-a456-426614174000",
    "reservationId": "res-880e8400-e29b-41d4-a716-446655440000",
    "paymentIntent": "pi_stripe_123456"
  },
  "completedSteps": [
    {
      "step": "INVENTORY_RESERVATION",
      "completedAt": "2024-01-15T10:00:05Z",
      "result": {
        "reservationId": "res-880e8400-e29b-41d4-a716-446655440000",
        "itemsReserved": 3
      }
    }
  ],
  "compensationPlan": [
    {
      "step": "RELEASE_INVENTORY",
      "condition": "IF_INVENTORY_RESERVED",
      "service": "inventory-service",
      "action": "release_reservation",
      "parameters": {
        "reservationId": "{{context.reservationId}}"
      }
    },
    {
      "step": "VOID_PAYMENT",
      "condition": "IF_PAYMENT_AUTHORIZED",
      "service": "payment-service",
      "action": "void_payment",
      "parameters": {
        "paymentIntent": "{{context.paymentIntent}}"
      }
    }
  ],
  "compensationState": {
    "executed": [],
    "pending": []
  },
  "events": [
    {
      "timestamp": "2024-01-15T10:00:00Z",
      "event": "SAGA_STARTED",
      "version": 1
    },
    {
      "timestamp": "2024-01-15T10:00:05Z",
      "event": "STEP_COMPLETED",
      "step": "INVENTORY_RESERVATION",
      "version": 2
    },
    {
      "timestamp": "2024-01-15T10:00:06Z",
      "event": "STEP_STARTED",
      "step": "PAYMENT_PROCESSING",
      "version": 3
    }
  ],
  "createdAt": "2024-01-15T10:00:00Z",
  "updatedAt": "2024-01-15T10:00:06Z"
}
```

#### Compensate Saga

Manually trigger compensation for a saga.

```http
POST /sagas/{sagaId}/compensate
```

**Request Body:**

```json
{
  "reason": "MANUAL_INTERVENTION",
  "fromStep": "PAYMENT_PROCESSING",
  "forceCompensation": false,
  "metadata": {
    "initiatedBy": "admin-user-123",
    "ticket": "SUPPORT-12345"
  }
}
```

**Success Response (202 Accepted):**

```json
{
  "sagaId": "saga-770e8400-e29b-41d4-a716-446655440000",
  "state": "COMPENSATING",
  "compensationId": "comp-990e8400-e29b-41d4-a716-446655440000",
  "compensationSteps": [
    {
      "step": "VOID_PAYMENT",
      "status": "IN_PROGRESS",
      "startedAt": "2024-01-15T10:10:00Z"
    },
    {
      "step": "RELEASE_INVENTORY",
      "status": "PENDING"
    }
  ],
  "estimatedCompletionTime": "PT2M"
}
```

### Workflow Templates

#### List Workflow Templates

Retrieve available workflow templates.

```http
GET /workflow-templates
```

**Success Response (200 OK):**

```json
{
  "templates": [
    {
      "templateId": "tmpl-order-fulfillment-v2",
      "name": "Order Fulfillment Workflow",
      "type": "ORDER_FULFILLMENT",
      "version": "2.1",
      "description": "Complete order fulfillment process with payment and shipping",
      "steps": [
        {
          "name": "INVENTORY_RESERVATION",
          "service": "inventory-service",
          "timeout": "PT30S",
          "retryPolicy": {
            "maxAttempts": 3,
            "backoffMultiplier": 2
          }
        },
        {
          "name": "PAYMENT_PROCESSING",
          "service": "payment-service",
          "timeout": "PT2M",
          "retryPolicy": {
            "maxAttempts": 2,
            "backoffMultiplier": 1.5
          }
        }
      ],
      "compensationSteps": [
        {
          "name": "RELEASE_INVENTORY",
          "service": "inventory-service",
          "condition": "IF_INVENTORY_RESERVED"
        },
        {
          "name": "VOID_PAYMENT",
          "service": "payment-service",
          "condition": "IF_PAYMENT_AUTHORIZED"
        }
      ],
      "active": true,
      "usage": {
        "lastMonth": 10234,
        "successRate": 93.48
      }
    }
  ]
}
```

### Health and Monitoring

#### Get Service Health

Check the health status of the coordination service.

```http
GET /health
```

**Success Response (200 OK):**

```json
{
  "status": "UP",
  "details": {
    "coordination": {
      "status": "UP",
      "activeWorkflows": 45,
      "stuckWorkflows": 0,
      "database": "Connected"
    },
    "dependencies": {
      "inventory-service": {
        "status": "UP",
        "responseTime": "PT0.15S"
      },
      "order-service": {
        "status": "UP",
        "responseTime": "PT0.12S"
      },
      "payment-service": {
        "status": "UP",
        "responseTime": "PT0.45S"
      },
      "delivery-service": {
        "status": "UP",
        "responseTime": "PT0.23S"
      },
      "notification-service": {
        "status": "UP",
        "responseTime": "PT0.18S"
      }
    },
    "infrastructure": {
      "database": {
        "status": "UP",
        "connectionPool": "8/20 active"
      },
      "messageQueue": {
        "status": "UP",
        "queueDepth": 12
      }
    }
  },
  "timestamp": "2024-01-15T10:15:00Z"
}
```

## Error Handling

### Error Response Format

```json
{
  "error": {
    "code": "WORKFLOW_EXECUTION_FAILED",
    "message": "Workflow failed at step PAYMENT_PROCESSING",
    "details": {
      "workflowId": "wf-660e8400-e29b-41d4-a716-446655440000",
      "failedStep": "PAYMENT_PROCESSING",
      "stepError": "Payment was declined by the issuing bank",
      "compensationTriggered": true
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
| WORKFLOW_NOT_FOUND | Workflow does not exist | 404 |
| SAGA_NOT_FOUND | Saga does not exist | 404 |
| WORKFLOW_ALREADY_EXISTS | Duplicate workflow | 409 |
| INVALID_WORKFLOW_STATE | Invalid state transition | 422 |
| WORKFLOW_EXECUTION_FAILED | Workflow step failed | 422 |
| COMPENSATION_FAILED | Compensation step failed | 422 |
| SERVICE_UNAVAILABLE | Dependent service down | 503 |
| TIMEOUT_EXCEEDED | Workflow execution timeout | 504 |
| RATE_LIMIT_EXCEEDED | Too many requests | 429 |
| INTERNAL_ERROR | Server error | 500 |

## Webhooks

### Event Types

- `workflow.started` - Workflow initiated
- `workflow.step.completed` - Workflow step completed
- `workflow.completed` - Workflow finished successfully
- `workflow.failed` - Workflow failed
- `workflow.cancelled` - Workflow cancelled
- `saga.compensating` - Compensation started
- `saga.compensated` - Compensation completed

### Webhook Payload

```json
{
  "id": "webhook-001",
  "timestamp": "2024-01-15T10:30:00Z",
  "event": "workflow.step.completed",
  "data": {
    "workflowId": "wf-660e8400-e29b-41d4-a716-446655440000",
    "workflowType": "ORDER_FULFILLMENT",
    "step": "PAYMENT_PROCESSING",
    "status": "COMPLETED",
    "completedAt": "2024-01-15T10:00:30Z",
    "result": {
      "paymentId": "pay-123456",
      "amount": 1499.97,
      "status": "CAPTURED"
    },
    "nextStep": "ORDER_CONFIRMATION"
  },
  "signature": "sha256=abcdef123456..."
}
```

## Rate Limiting

- **Standard Services:** 10000 requests per hour
- **Admin Users:** 5000 requests per hour
- **External APIs:** 1000 requests per hour

## SDK Examples

### JavaScript/TypeScript

```typescript
import { CoordinationClient } from '@firstviscount/coordination-sdk';

const client = new CoordinationClient({
  baseUrl: 'https://api.firstviscount.com/coordination/v1',
  apiKey: process.env.API_KEY
});

// Start order workflow
const workflow = await client.workflows.startOrder({
  orderId: 'ord-123',
  customerId: 'cust-456',
  items: [
    { productId: 'prod-789', quantity: 2, unitPrice: 599.99 }
  ],
  paymentMethod: { type: 'CREDIT_CARD', token: 'tok_123' },
  shippingAddress: { /* address */ }
});

// Monitor workflow progress
const status = await client.workflows.getStatus(workflow.workflowId);

// Cancel workflow if needed
if (status.status === 'IN_PROGRESS') {
  await client.workflows.cancel(workflow.workflowId, {
    reason: 'CUSTOMER_REQUESTED',
    initiatedBy: 'CUSTOMER'
  });
}
```

### Python

```python
from firstviscount import CoordinationClient

client = CoordinationClient(
    base_url='https://api.firstviscount.com/coordination/v1',
    api_key=os.environ['API_KEY']
)

# Get workflow metrics
metrics = client.workflows.get_metrics(
    period='MONTH',
    type='ORDER_FULFILLMENT'
)

# List active workflows
active_workflows = client.workflows.list_active(
    status='IN_PROGRESS',
    page=0,
    size=50
)

# Get saga details
saga = client.sagas.get('saga-770e8400-e29b-41d4-a716-446655440000')
```

## Testing

### Test Environment

```
Base URL: https://api-test.firstviscount.com/coordination/v1
```

### Test Workflows

| Type | Description | Test Data |
|------|-------------|-----------|
| ORDER_FULFILLMENT | Complete order flow | Use test order IDs |
| RETURN_PROCESSING | Return workflow | Use test return IDs |
| COMPENSATION_TEST | Test compensation | Trigger failure scenarios |

## Changelog

### Version 1.0.0 (2024-01-15)

- Initial release
- Order fulfillment workflows
- Saga pattern implementation
- Compensation handling
- Workflow monitoring
- Template management
# Notification API Reference

## Overview

The Notification API handles all customer communications across multiple channels including email, SMS, and push notifications. It manages notification templates, user preferences, delivery tracking, and ensures reliable message delivery with comprehensive analytics.

**Base URL:** `http://notification-service:8086/api/v1`  
**Port:** 8086

## Authentication

All endpoints require JWT authentication:

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

## OpenAPI Specification

```yaml
openapi: 3.0.3
info:
  title: Notification Management API
  version: 1.0.0
  description: Multi-channel customer communication platform
servers:
  - url: http://notification-service:8086/api/v1
    description: Internal service endpoint
  - url: https://api.firstviscount.com/notifications/v1
    description: Production API gateway
```

## Endpoints

### Notification Management

#### Send Notification

Send a notification through one or more channels using a template.

```http
POST /notifications/send
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
  "recipient": {
    "customerId": "123e4567-e89b-12d3-a456-426614174000",
    "email": "john.doe@example.com",
    "phone": "+1234567890",
    "deviceTokens": ["fcm-token-123", "apns-token-456"],
    "name": "John Doe",
    "language": "en",
    "timezone": "America/New_York"
  },
  "template": "order_confirmation",
  "channels": ["email", "sms"],
  "context": {
    "order": {
      "number": "ORD-2024-001234",
      "totalAmount": 1299.99,
      "currency": "USD",
      "items": [
        {
          "name": "Premium Laptop Pro 15",
          "quantity": 1,
          "price": 1299.99,
          "image": "https://cdn.firstviscount.com/products/laptop-001-thumb.jpg"
        }
      ],
      "estimatedDelivery": "2024-01-22",
      "trackingUrl": "https://track.firstviscount.com/1234567890"
    },
    "customer": {
      "name": "John Doe",
      "email": "john.doe@example.com",
      "loyaltyTier": "GOLD"
    },
    "shipping": {
      "address": "123 Main St, New York, NY 10001",
      "method": "Standard Shipping"
    }
  },
  "options": {
    "priority": "HIGH",
    "scheduledFor": null,
    "respectQuietHours": true,
    "trackingEnabled": true,
    "retryOnFailure": true,
    "maxRetries": 3
  },
  "metadata": {
    "orderId": "ord-770e8400-e29b-41d4-a716-446655440000",
    "source": "order-service",
    "campaignId": "welcome-series-001",
    "tags": ["transactional", "order"]
  }
}
```

**Request Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| recipient | object | Yes | Recipient information |
| template | string | Yes | Template name to use |
| channels | array | Yes | Delivery channels (email, sms, push) |
| context | object | Yes | Template variables |
| options | object | No | Delivery options |
| metadata | object | No | Additional metadata |

**Success Response (201 Created):**

```json
{
  "notificationId": "notif-880e8400-e29b-41d4-a716-446655440000",
  "status": "QUEUED",
  "template": "order_confirmation",
  "recipient": {
    "customerId": "123e4567-e89b-12d3-a456-426614174000",
    "email": "john.doe@example.com",
    "maskedPhone": "+1***-***-7890"
  },
  "channels": [
    {
      "channel": "email",
      "status": "QUEUED",
      "messageId": "email-990e8400-e29b-41d4-a716-446655440000",
      "provider": "sendgrid",
      "estimatedDelivery": "2024-01-15T10:01:00Z"
    },
    {
      "channel": "sms",
      "status": "QUEUED",
      "messageId": "sms-aa0e8400-e29b-41d4-a716-446655440000",
      "provider": "twilio",
      "estimatedDelivery": "2024-01-15T10:00:30Z"
    }
  ],
  "preview": {
    "subject": "Order ORD-2024-001234 Confirmed!",
    "bodySnippet": "Hi John Doe, your order ORD-2024-001234 for $1,299.99 has been confirmed..."
  },
  "queuedAt": "2024-01-15T10:00:00Z",
  "scheduledFor": "2024-01-15T10:00:00Z"
}
```

**Error Responses:**

- **400 Bad Request** - Invalid notification data
- **401 Unauthorized** - Missing or invalid authentication
- **404 Not Found** - Template not found
- **422 Unprocessable Entity** - Template validation failed
- **429 Too Many Requests** - Rate limit exceeded

#### Get Notification Status

Retrieve the status and delivery details of a notification.

```http
GET /notifications/{notificationId}
```

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| notificationId | string (UUID) | Yes | Notification identifier |

**Success Response (200 OK):**

```json
{
  "notificationId": "notif-880e8400-e29b-41d4-a716-446655440000",
  "status": "DELIVERED",
  "template": "order_confirmation",
  "recipient": {
    "customerId": "123e4567-e89b-12d3-a456-426614174000",
    "email": "john.doe@example.com",
    "maskedPhone": "+1***-***-7890"
  },
  "channels": [
    {
      "channel": "email",
      "status": "DELIVERED",
      "messageId": "email-990e8400-e29b-41d4-a716-446655440000",
      "provider": "sendgrid",
      "providerMessageId": "sg-msg-123456",
      "subject": "Order ORD-2024-001234 Confirmed!",
      "timeline": {
        "queued": "2024-01-15T10:00:00Z",
        "sent": "2024-01-15T10:00:30Z",
        "delivered": "2024-01-15T10:00:45Z",
        "opened": "2024-01-15T11:30:00Z",
        "clicked": "2024-01-15T11:31:00Z"
      },
      "engagement": {
        "opened": true,
        "clicked": true,
        "bounced": false,
        "complained": false,
        "unsubscribed": false
      },
      "attempts": 1,
      "lastError": null
    },
    {
      "channel": "sms",
      "status": "DELIVERED",
      "messageId": "sms-aa0e8400-e29b-41d4-a716-446655440000",
      "provider": "twilio",
      "providerMessageId": "SM123456",
      "message": "Hi John! Your order ORD-2024-001234 for $1,299.99 has been confirmed. Track: https://track.firstviscount.com/1234567890",
      "timeline": {
        "queued": "2024-01-15T10:00:00Z",
        "sent": "2024-01-15T10:00:35Z",
        "delivered": "2024-01-15T10:00:40Z"
      },
      "segments": 1,
      "cost": 0.0075,
      "attempts": 1
    }
  ],
  "analytics": {
    "totalCost": 0.0075,
    "deliveryRate": 100.0,
    "engagementRate": 50.0
  },
  "metadata": {
    "orderId": "ord-770e8400-e29b-41d4-a716-446655440000",
    "source": "order-service"
  },
  "createdAt": "2024-01-15T10:00:00Z",
  "completedAt": "2024-01-15T10:00:45Z"
}
```

#### Send Bulk Notifications

Send notifications to multiple recipients efficiently.

```http
POST /notifications/bulk
```

**Request Body:**

```json
{
  "template": "promotional_email",
  "channels": ["email"],
  "recipients": [
    {
      "customerId": "123e4567-e89b-12d3-a456-426614174000",
      "email": "john.doe@example.com",
      "context": {
        "customer": {
          "name": "John Doe",
          "loyaltyPoints": 1250
        },
        "offer": {
          "discountPercent": 20,
          "validUntil": "2024-02-01"
        }
      }
    },
    {
      "customerId": "456e7890-e89b-12d3-a456-426614174000",
      "email": "jane.smith@example.com",
      "context": {
        "customer": {
          "name": "Jane Smith",
          "loyaltyPoints": 890
        },
        "offer": {
          "discountPercent": 15,
          "validUntil": "2024-02-01"
        }
      }
    }
  ],
  "options": {
    "batchSize": 100,
    "delayBetweenBatches": "PT5S",
    "respectQuietHours": true
  },
  "metadata": {
    "campaignId": "winter-sale-2024",
    "campaignType": "PROMOTIONAL"
  }
}
```

**Success Response (202 Accepted):**

```json
{
  "bulkJobId": "bulk-bb0e8400-e29b-41d4-a716-446655440000",
  "status": "PROCESSING",
  "totalRecipients": 2,
  "estimatedCompletion": "2024-01-15T10:05:00Z",
  "progress": {
    "processed": 0,
    "successful": 0,
    "failed": 0,
    "percentage": 0.0
  }
}
```

### Template Management

#### List Templates

Retrieve available notification templates.

```http
GET /templates
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| channel | string | No | Filter by channel (email, sms, push) |
| category | string | No | Filter by category |
| active | boolean | No | Filter by active status |
| search | string | No | Search in name and description |
| page | integer | No | Page number (0-based) |
| size | integer | No | Page size |

**Success Response (200 OK):**

```json
{
  "templates": [
    {
      "id": "tmpl-110e8400-e29b-41d4-a716-446655440000",
      "name": "order_confirmation",
      "displayName": "Order Confirmation",
      "channel": "email",
      "category": "transactional",
      "subject": "Order {{order.number}} Confirmed!",
      "description": "Sent when an order is successfully placed",
      "variables": [
        {
          "name": "order",
          "type": "object",
          "required": true,
          "description": "Order information",
          "fields": [
            {"name": "number", "type": "string", "required": true},
            {"name": "totalAmount", "type": "number", "required": true},
            {"name": "estimatedDelivery", "type": "date", "required": false}
          ]
        },
        {
          "name": "customer",
          "type": "object",
          "required": true,
          "description": "Customer information",
          "fields": [
            {"name": "name", "type": "string", "required": true},
            {"name": "email", "type": "string", "required": true}
          ]
        }
      ],
      "tags": ["order", "confirmation", "transactional"],
      "active": true,
      "version": 3,
      "usage": {
        "lastMonth": 1234,
        "last7Days": 45,
        "deliveryRate": 98.5
      },
      "createdAt": "2024-01-01T10:00:00Z",
      "updatedAt": "2024-01-10T14:30:00Z"
    }
  ],
  "pagination": {
    "page": 0,
    "size": 20,
    "totalElements": 15,
    "totalPages": 1
  }
}
```

#### Get Template Details

Retrieve detailed information about a specific template.

```http
GET /templates/{templateId}
```

**Success Response (200 OK):**

```json
{
  "id": "tmpl-110e8400-e29b-41d4-a716-446655440000",
  "name": "order_confirmation",
  "displayName": "Order Confirmation",
  "channel": "email",
  "category": "transactional",
  "subject": "Order {{order.number}} Confirmed!",
  "body": {
    "html": "<!DOCTYPE html><html><head>...</head><body><h1>Order Confirmed!</h1><p>Hi {{customer.name}},</p><p>Your order {{order.number}} for {{order.totalAmount}} has been confirmed...</p></body></html>",
    "text": "Order Confirmed!\n\nHi {{customer.name}},\n\nYour order {{order.number}} for {{order.totalAmount}} has been confirmed..."
  },
  "variables": [
    {
      "name": "order",
      "type": "object",
      "required": true,
      "description": "Order information",
      "fields": [
        {"name": "number", "type": "string", "required": true},
        {"name": "totalAmount", "type": "number", "required": true}
      ]
    }
  ],
  "metadata": {
    "category": "transactional",
    "tags": ["order", "confirmation"],
    "description": "Sent when an order is successfully placed",
    "designSystem": "v2.1",
    "brandColors": {
      "primary": "#007bff",
      "secondary": "#6c757d"
    }
  },
  "versions": [
    {
      "version": 3,
      "active": true,
      "createdAt": "2024-01-10T14:30:00Z",
      "changes": ["Updated design", "Added tracking link"]
    },
    {
      "version": 2,
      "active": false,
      "createdAt": "2024-01-05T10:00:00Z",
      "changes": ["Fixed typo", "Updated branding"]
    }
  ],
  "analytics": {
    "sentLast30Days": 1234,
    "deliveryRate": 98.5,
    "openRate": 65.2,
    "clickRate": 12.8,
    "unsubscribeRate": 0.1
  },
  "active": true,
  "version": 3,
  "createdAt": "2024-01-01T10:00:00Z",
  "updatedAt": "2024-01-10T14:30:00Z",
  "createdBy": "admin-123"
}
```

#### Create Template (Admin Only)

Create a new notification template.

```http
POST /templates
```

**Request Body:**

```json
{
  "name": "welcome_email",
  "displayName": "Welcome Email",
  "channel": "email",
  "category": "onboarding",
  "subject": "Welcome to First Viscount, {{customer.name}}!",
  "body": {
    "html": "<!DOCTYPE html><html><head><style>/* CSS styles */</style></head><body><div class=\"container\"><h1>Welcome {{customer.name}}!</h1><p>Thank you for joining First Viscount...</p><a href=\"{{links.shopNow}}\" class=\"button\">Start Shopping</a></div></body></html>",
    "text": "Welcome {{customer.name}}!\n\nThank you for joining First Viscount. We're excited to have you as part of our community.\n\nStart shopping: {{links.shopNow}}"
  },
  "variables": [
    {
      "name": "customer",
      "type": "object",
      "required": true,
      "description": "Customer information",
      "fields": [
        {"name": "name", "type": "string", "required": true},
        {"name": "email", "type": "string", "required": true}
      ]
    },
    {
      "name": "links",
      "type": "object",
      "required": true,
      "description": "Action links",
      "fields": [
        {"name": "shopNow", "type": "string", "required": true},
        {"name": "unsubscribe", "type": "string", "required": true}
      ]
    }
  ],
  "metadata": {
    "category": "onboarding",
    "tags": ["welcome", "new-customer"],
    "description": "Welcome email for new customers",
    "designNotes": "Uses brand colors and modern layout"
  },
  "options": {
    "trackingEnabled": true,
    "requiresUnsubscribeLink": true,
    "allowScheduling": true
  }
}
```

**Success Response (201 Created):**

```json
{
  "templateId": "tmpl-330e8400-e29b-41d4-a716-446655440000",
  "name": "welcome_email",
  "version": 1,
  "status": "ACTIVE",
  "validationResult": {
    "valid": true,
    "issues": [],
    "warnings": [
      "Consider adding alt text for images"
    ]
  },
  "createdAt": "2024-01-15T10:30:00Z"
}
```

#### Preview Template

Preview a template with sample data.

```http
POST /templates/{templateId}/preview
```

**Request Body:**

```json
{
  "context": {
    "customer": {
      "name": "John Doe",
      "email": "john.doe@example.com"
    },
    "order": {
      "number": "ORD-2024-001234",
      "totalAmount": 1299.99
    }
  },
  "channel": "email"
}
```

**Success Response (200 OK):**

```json
{
  "preview": {
    "subject": "Order ORD-2024-001234 Confirmed!",
    "body": {
      "html": "<!DOCTYPE html><html>...<h1>Order Confirmed!</h1><p>Hi John Doe,</p><p>Your order ORD-2024-001234 for $1,299.99 has been confirmed...</p>...</html>",
      "text": "Order Confirmed!\n\nHi John Doe,\n\nYour order ORD-2024-001234 for $1,299.99 has been confirmed..."
    }
  },
  "metadata": {
    "renderTime": 45,
    "size": {
      "html": 15420,
      "text": 1250
    }
  }
}
```

### Customer Preferences

#### Get Customer Preferences

Retrieve notification preferences for a customer.

```http
GET /preferences/customer/{customerId}
```

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| customerId | string (UUID) | Yes | Customer identifier |

**Success Response (200 OK):**

```json
{
  "customerId": "123e4567-e89b-12d3-a456-426614174000",
  "channels": {
    "email": {
      "enabled": true,
      "address": "john.doe@example.com",
      "verified": true,
      "verifiedAt": "2024-01-01T10:00:00Z",
      "bounced": false,
      "lastBounceAt": null
    },
    "sms": {
      "enabled": true,
      "number": "+1234567890",
      "verified": true,
      "verifiedAt": "2024-01-02T11:00:00Z",
      "countryCode": "US"
    },
    "push": {
      "enabled": false,
      "devices": [
        {
          "token": "fcm-token-123",
          "platform": "android",
          "appVersion": "1.2.3",
          "addedAt": "2024-01-03T12:00:00Z",
          "lastSeen": "2024-01-14T18:00:00Z"
        }
      ]
    }
  },
  "subscriptions": {
    "order_updates": {
      "enabled": true,
      "channels": ["email", "sms"]
    },
    "shipping_updates": {
      "enabled": true,
      "channels": ["email", "push"]
    },
    "promotions": {
      "enabled": false,
      "channels": []
    },
    "product_updates": {
      "enabled": true,
      "channels": ["email"]
    },
    "price_alerts": {
      "enabled": true,
      "channels": ["email"]
    },
    "newsletter": {
      "enabled": false,
      "channels": []
    }
  },
  "quietHours": {
    "enabled": true,
    "startTime": "22:00",
    "endTime": "08:00",
    "timezone": "America/New_York",
    "excludeCritical": true,
    "excludeChannels": ["sms"]
  },
  "frequencyLimits": {
    "dailyLimit": 10,
    "weeklyLimit": 50,
    "promotionalDailyLimit": 2
  },
  "language": "en",
  "region": "US",
  "unsubscribeTokens": [
    {
      "token": "unsub-token-123",
      "categories": ["promotions"],
      "createdAt": "2024-01-05T10:00:00Z"
    }
  ],
  "analytics": {
    "totalReceived": 123,
    "totalOpened": 89,
    "totalClicked": 34,
    "lastEngagement": "2024-01-14T15:30:00Z"
  },
  "createdAt": "2024-01-01T10:00:00Z",
  "updatedAt": "2024-01-10T15:00:00Z"
}
```

#### Update Customer Preferences

Update notification preferences for a customer.

```http
PUT /preferences/customer/{customerId}
```

**Request Body:**

```json
{
  "channels": {
    "sms": {
      "enabled": false
    },
    "push": {
      "enabled": true
    }
  },
  "subscriptions": {
    "promotions": {
      "enabled": true,
      "channels": ["email"]
    },
    "price_alerts": {
      "enabled": false,
      "channels": []
    }
  },
  "quietHours": {
    "enabled": true,
    "startTime": "21:00",
    "endTime": "09:00",
    "timezone": "America/New_York"
  },
  "frequencyLimits": {
    "dailyLimit": 5,
    "promotionalDailyLimit": 1
  }
}
```

**Success Response (200 OK):**

```json
{
  "customerId": "123e4567-e89b-12d3-a456-426614174000",
  "updated": true,
  "changes": [
    "channels.sms.enabled: true → false",
    "channels.push.enabled: false → true",
    "subscriptions.promotions.enabled: false → true",
    "subscriptions.price_alerts.enabled: true → false",
    "quietHours.startTime: 22:00 → 21:00",
    "quietHours.endTime: 08:00 → 09:00",
    "frequencyLimits.dailyLimit: 10 → 5",
    "frequencyLimits.promotionalDailyLimit: 2 → 1"
  ],
  "updatedAt": "2024-01-15T10:45:00Z"
}
```

### Unsubscribe Management

#### Unsubscribe from Notifications

Process an unsubscribe request.

```http
POST /unsubscribe
```

**Request Body:**

```json
{
  "token": "unsub-token-440e8400-e29b-41d4-a716-446655440000",
  "categories": ["promotions", "newsletter"],
  "reason": "TOO_FREQUENT",
  "feedback": "Receiving too many promotional emails",
  "email": "john.doe@example.com"
}
```

**Success Response (200 OK):**

```json
{
  "success": true,
  "customerId": "123e4567-e89b-12d3-a456-426614174000",
  "unsubscribed": ["promotions", "newsletter"],
  "remaining": ["order_updates", "shipping_updates", "product_updates"],
  "message": "You have been unsubscribed from promotional emails and newsletters.",
  "resubscribeUrl": "https://preferences.firstviscount.com/resubscribe?token=resub-token-123"
}
```

#### Get Unsubscribe Page

Retrieve unsubscribe page content.

```http
GET /unsubscribe/{token}
```

**Success Response (200 OK):**

```json
{
  "customer": {
    "email": "john.doe@example.com",
    "name": "John Doe"
  },
  "currentSubscriptions": [
    {
      "category": "order_updates",
      "displayName": "Order Updates",
      "description": "Updates about your orders and purchases",
      "subscribed": true,
      "required": true
    },
    {
      "category": "promotions",
      "displayName": "Promotions & Offers",
      "description": "Special deals and promotional offers",
      "subscribed": true,
      "required": false
    }
  ],
  "unsubscribeReasons": [
    {"value": "TOO_FREQUENT", "label": "Too many emails"},
    {"value": "NOT_RELEVANT", "label": "Content not relevant"},
    {"value": "NEVER_SIGNED_UP", "label": "I never signed up"},
    {"value": "OTHER", "label": "Other reason"}
  ]
}
```

### Notification History

#### Get Customer Notification History

Retrieve notification history for a customer.

```http
GET /notifications/customer/{customerId}
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| startDate | date | No | Start date filter |
| endDate | date | No | End date filter |
| channel | string | No | Filter by channel |
| template | string | No | Filter by template |
| status | string | No | Filter by status |
| page | integer | No | Page number |
| size | integer | No | Page size |

**Success Response (200 OK):**

```json
{
  "notifications": [
    {
      "notificationId": "notif-880e8400-e29b-41d4-a716-446655440000",
      "template": "order_confirmation",
      "templateDisplayName": "Order Confirmation",
      "channels": ["email", "sms"],
      "status": "DELIVERED",
      "subject": "Order ORD-2024-001234 Confirmed!",
      "summary": "Your order for Premium Laptop Pro 15 has been confirmed",
      "sentAt": "2024-01-15T10:00:30Z",
      "deliveredAt": "2024-01-15T10:00:45Z",
      "engagement": {
        "opened": true,
        "clicked": true,
        "openedAt": "2024-01-15T11:30:00Z",
        "clickedAt": "2024-01-15T11:31:00Z"
      },
      "metadata": {
        "orderId": "ord-770e8400-e29b-41d4-a716-446655440000",
        "category": "transactional"
      }
    }
  ],
  "summary": {
    "total": 45,
    "delivered": 43,
    "failed": 2,
    "pending": 0,
    "byChannel": {
      "email": {
        "sent": 45,
        "delivered": 43,
        "opened": 38,
        "clicked": 25,
        "bounced": 2,
        "complained": 0
      },
      "sms": {
        "sent": 20,
        "delivered": 19,
        "failed": 1
      },
      "push": {
        "sent": 5,
        "delivered": 4,
        "opened": 3
      }
    },
    "byCategory": {
      "transactional": 35,
      "promotional": 8,
      "notification": 2
    }
  },
  "analytics": {
    "engagementRate": 62.8,
    "deliveryRate": 95.6,
    "averageDeliveryTime": "PT45S"
  },
  "pagination": {
    "page": 0,
    "size": 20,
    "totalElements": 45,
    "totalPages": 3
  }
}
```

### Testing and Analytics

#### Send Test Notification (Admin Only)

Send a test notification with sample data.

```http
POST /notifications/test
```

**Request Body:**

```json
{
  "template": "order_confirmation",
  "channel": "email",
  "recipient": "test@example.com",
  "context": {
    "order": {
      "number": "TEST-ORDER-123",
      "totalAmount": 99.99,
      "estimatedDelivery": "2024-01-20"
    },
    "customer": {
      "name": "Test User",
      "email": "test@example.com"
    }
  },
  "options": {
    "bypassPreferences": true,
    "markAsTest": true
  }
}
```

**Success Response (200 OK):**

```json
{
  "testId": "test-550e8400-e29b-41d4-a716-446655440000",
  "status": "SENT",
  "messageId": "test-email-660e8400-e29b-41d4-a716-446655440000",
  "provider": "sendgrid",
  "preview": {
    "subject": "Order TEST-ORDER-123 Confirmed!",
    "bodySnippet": "Hi Test User, Your order TEST-ORDER-123 for $99.99 has been confirmed..."
  },
  "sentAt": "2024-01-15T11:00:00Z",
  "estimatedDelivery": "2024-01-15T11:00:15Z"
}
```

#### Get Notification Analytics

Retrieve comprehensive notification analytics.

```http
GET /analytics/notifications
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| startDate | date | No | Start date for analytics |
| endDate | date | No | End date for analytics |
| channel | string | No | Filter by channel |
| template | string | No | Filter by template |
| groupBy | string | No | Group by (day, week, month) |

**Success Response (200 OK):**

```json
{
  "period": {
    "start": "2024-01-01",
    "end": "2024-01-31",
    "days": 31
  },
  "overview": {
    "totalSent": 12543,
    "totalDelivered": 11987,
    "totalFailed": 556,
    "deliveryRate": 95.56,
    "totalCost": 1254.30
  },
  "channels": {
    "email": {
      "sent": 8234,
      "delivered": 7891,
      "opened": 5012,
      "clicked": 1567,
      "bounced": 234,
      "complained": 12,
      "unsubscribed": 8,
      "deliveryRate": 95.83,
      "openRate": 63.51,
      "clickRate": 19.84,
      "bounceRate": 2.96,
      "cost": 823.40
    },
    "sms": {
      "sent": 3456,
      "delivered": 3298,
      "failed": 158,
      "deliveryRate": 95.43,
      "cost": 259.20
    },
    "push": {
      "sent": 853,
      "delivered": 798,
      "opened": 456,
      "failed": 55,
      "deliveryRate": 93.55,
      "openRate": 57.14,
      "cost": 0.00
    }
  },
  "templates": [
    {
      "template": "order_confirmation",
      "sent": 2345,
      "delivered": 2289,
      "deliveryRate": 97.61,
      "openRate": 78.45,
      "clickRate": 23.12
    },
    {
      "template": "shipping_update",
      "sent": 1876,
      "delivered": 1834,
      "deliveryRate": 97.76,
      "openRate": 65.23,
      "clickRate": 15.67
    }
  ],
  "trends": {
    "daily": [
      {
        "date": "2024-01-01",
        "sent": 423,
        "delivered": 401,
        "deliveryRate": 94.8
      }
    ]
  },
  "providers": {
    "sendgrid": {
      "sent": 6234,
      "delivered": 5987,
      "deliveryRate": 96.04,
      "cost": 623.40
    },
    "twilio": {
      "sent": 3456,
      "delivered": 3298,
      "deliveryRate": 95.43,
      "cost": 259.20
    }
  }
}
```

## Error Handling

### Error Response Format

```json
{
  "error": {
    "code": "TEMPLATE_NOT_FOUND",
    "message": "Template 'invalid_template' not found",
    "details": {
      "templateName": "invalid_template",
      "availableTemplates": ["order_confirmation", "shipping_update"]
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
| TEMPLATE_NOT_FOUND | Template does not exist | 404 |
| CUSTOMER_NOT_FOUND | Customer does not exist | 404 |
| TEMPLATE_VALIDATION_ERROR | Template variables missing | 422 |
| CHANNEL_NOT_AVAILABLE | Channel disabled for customer | 422 |
| RATE_LIMIT_EXCEEDED | Too many notifications | 429 |
| PROVIDER_ERROR | External provider error | 502 |
| SERVICE_UNAVAILABLE | Service temporarily down | 503 |
| QUOTA_EXCEEDED | Monthly quota exceeded | 402 |
| INVALID_RECIPIENT | Recipient data invalid | 422 |
| TEMPLATE_RENDER_ERROR | Template rendering failed | 422 |

## Webhooks

### Event Types

- `notification.sent` - Notification sent to provider
- `notification.delivered` - Notification delivered
- `notification.opened` - Email opened
- `notification.clicked` - Link clicked
- `notification.bounced` - Email bounced
- `notification.failed` - Delivery failed
- `customer.unsubscribed` - Customer unsubscribed
- `template.created` - New template created

### Webhook Payload

```json
{
  "id": "webhook-001",
  "timestamp": "2024-01-15T10:30:00Z",
  "event": "notification.delivered",
  "data": {
    "notificationId": "notif-880e8400-e29b-41d4-a716-446655440000",
    "customerId": "123e4567-e89b-12d3-a456-426614174000",
    "channel": "email",
    "template": "order_confirmation",
    "status": "DELIVERED",
    "deliveredAt": "2024-01-15T10:00:45Z",
    "provider": "sendgrid",
    "providerMessageId": "sg-msg-123456"
  },
  "signature": "sha256=abcdef123456..."
}
```

## Rate Limiting

- **Standard Users:** 1000 requests per hour
- **Admin Users:** 5000 requests per hour
- **Service Accounts:** Unlimited
- **Notification Sending:** Based on plan limits

## SDK Examples

### JavaScript/TypeScript

```typescript
import { NotificationClient } from '@firstviscount/notification-sdk';

const client = new NotificationClient({
  baseUrl: 'https://api.firstviscount.com/notifications/v1',
  apiKey: process.env.API_KEY
});

// Send notification
const notification = await client.notifications.send({
  recipient: {
    customerId: '123e4567-e89b-12d3-a456-426614174000',
    email: 'john.doe@example.com'
  },
  template: 'order_confirmation',
  channels: ['email'],
  context: {
    order: { number: 'ORD-123', totalAmount: 99.99 },
    customer: { name: 'John Doe' }
  }
});

// Get preferences
const preferences = await client.preferences.getCustomer('123e4567-e89b-12d3-a456-426614174000');

// Update preferences
await client.preferences.updateCustomer('123e4567-e89b-12d3-a456-426614174000', {
  subscriptions: { promotions: { enabled: false } }
});
```

### Python

```python
from firstviscount import NotificationClient

client = NotificationClient(
    base_url='https://api.firstviscount.com/notifications/v1',
    api_key=os.environ['API_KEY']
)

# Send bulk notifications
bulk_job = client.notifications.send_bulk(
    template='promotional_email',
    channels=['email'],
    recipients=[
        {
            'customer_id': 'cust-001',
            'email': 'john@example.com',
            'context': {'offer': {'discount': 20}}
        }
    ]
)

# Get analytics
analytics = client.analytics.get_notifications(
    start_date='2024-01-01',
    end_date='2024-01-31',
    group_by='day'
)
```

## Testing

### Test Environment

```
Base URL: https://api-test.firstviscount.com/notifications/v1
```

### Test Templates

| Template Name | Channel | Description |
|---------------|---------|-------------|
| test_email | email | Basic test email template |
| test_sms | sms | Basic test SMS template |
| test_push | push | Basic push notification template |

## Changelog

### Version 1.0.0 (2024-01-15)

- Initial release
- Multi-channel notifications
- Template management
- Customer preferences
- Delivery tracking
- Analytics and reporting
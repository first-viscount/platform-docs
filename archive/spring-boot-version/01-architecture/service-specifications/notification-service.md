# Notification Service Specification

## Overview

The Notification Service handles all customer communications across multiple channels including email, SMS, and push notifications. It manages notification templates, user preferences, and delivery tracking while ensuring reliable message delivery.

### Key Responsibilities
- Multi-channel notification delivery (Email, SMS, Push)
- Template management with dynamic content
- User preference management
- Delivery tracking and retry logic
- Unsubscribe handling and compliance
- Notification analytics and reporting

## API Specification

### Base URL
```
http://notification-service:8086/api/v1
```

### Endpoints

#### Send Notification
```http
POST /notifications/send
Content-Type: application/json
Authorization: Bearer {token}

{
  "recipient": {
    "customerId": "123e4567-e89b-12d3-a456-426614174000",
    "email": "john.doe@example.com",
    "phone": "+1234567890",
    "deviceTokens": ["token123", "token456"]
  },
  "template": "order_confirmation",
  "channels": ["email", "sms"],
  "context": {
    "orderNumber": "ORD-2024-001234",
    "customerName": "John Doe",
    "items": [
      {
        "name": "Premium Laptop Pro 15",
        "quantity": 1,
        "price": 1299.99
      }
    ],
    "totalAmount": 1299.99,
    "estimatedDelivery": "2024-01-22",
    "trackingUrl": "https://track.firstviscount.com/1234567890"
  },
  "metadata": {
    "orderId": "ord-770e8400-e29b-41d4-a716-446655440000",
    "source": "order-service",
    "priority": "high"
  },
  "scheduledFor": null
}

Response:
{
  "notificationId": "notif-880e8400-e29b-41d4-a716-446655440000",
  "status": "QUEUED",
  "channels": [
    {
      "channel": "email",
      "status": "QUEUED",
      "messageId": "email-990e8400-e29b-41d4-a716-446655440000"
    },
    {
      "channel": "sms",
      "status": "QUEUED",
      "messageId": "sms-aa0e8400-e29b-41d4-a716-446655440000"
    }
  ],
  "queuedAt": "2024-01-15T10:00:00Z"
}
```

#### Get Notification Status
```http
GET /notifications/{notificationId}
Authorization: Bearer {token}

Response:
{
  "notificationId": "notif-880e8400-e29b-41d4-a716-446655440000",
  "status": "DELIVERED",
  "template": "order_confirmation",
  "recipient": {
    "customerId": "123e4567-e89b-12d3-a456-426614174000",
    "email": "john.doe@example.com"
  },
  "channels": [
    {
      "channel": "email",
      "status": "DELIVERED",
      "messageId": "email-990e8400-e29b-41d4-a716-446655440000",
      "sentAt": "2024-01-15T10:00:30Z",
      "deliveredAt": "2024-01-15T10:00:45Z",
      "provider": "sendgrid",
      "providerMessageId": "sg-msg-123456"
    },
    {
      "channel": "sms",
      "status": "DELIVERED",
      "messageId": "sms-aa0e8400-e29b-41d4-a716-446655440000",
      "sentAt": "2024-01-15T10:00:35Z",
      "deliveredAt": "2024-01-15T10:00:40Z",
      "provider": "twilio",
      "providerMessageId": "SM123456"
    }
  ],
  "createdAt": "2024-01-15T10:00:00Z",
  "completedAt": "2024-01-15T10:00:45Z"
}
```

#### List Templates
```http
GET /templates?channel=email&active=true
Authorization: Bearer {admin-token}

Response:
{
  "templates": [
    {
      "id": "tmpl-110e8400-e29b-41d4-a716-446655440000",
      "name": "order_confirmation",
      "displayName": "Order Confirmation",
      "channel": "email",
      "subject": "Order {{order.number}} Confirmed!",
      "description": "Sent when an order is successfully placed",
      "variables": [
        {
          "name": "order",
          "type": "object",
          "required": true,
          "fields": ["number", "totalAmount", "estimatedDelivery"]
        },
        {
          "name": "customer",
          "type": "object",
          "required": true,
          "fields": ["name", "email"]
        }
      ],
      "active": true,
      "version": 3,
      "createdAt": "2024-01-01T10:00:00Z",
      "updatedAt": "2024-01-10T14:30:00Z"
    },
    {
      "id": "tmpl-220e8400-e29b-41d4-a716-446655440000",
      "name": "shipping_update",
      "displayName": "Shipping Update",
      "channel": "email",
      "subject": "Your order has shipped!",
      "active": true,
      "version": 2
    }
  ],
  "totalCount": 15
}
```

#### Create/Update Template (Admin)
```http
POST /templates
Content-Type: application/json
Authorization: Bearer {admin-token}

{
  "name": "welcome_email",
  "displayName": "Welcome Email",
  "channel": "email",
  "subject": "Welcome to First Viscount, {{customer.name}}!",
  "body": {
    "html": "<html><body><h1>Welcome {{customer.name}}!</h1><p>Thank you for joining...</p></body></html>",
    "text": "Welcome {{customer.name}}! Thank you for joining..."
  },
  "variables": [
    {
      "name": "customer",
      "type": "object",
      "required": true,
      "fields": ["name", "email"]
    }
  ],
  "metadata": {
    "category": "onboarding",
    "tags": ["welcome", "new-customer"]
  }
}

Response:
{
  "templateId": "tmpl-330e8400-e29b-41d4-a716-446655440000",
  "name": "welcome_email",
  "version": 1,
  "status": "ACTIVE",
  "createdAt": "2024-01-15T10:30:00Z"
}
```

#### Get Customer Preferences
```http
GET /preferences/customer/{customerId}
Authorization: Bearer {token}

Response:
{
  "customerId": "123e4567-e89b-12d3-a456-426614174000",
  "channels": {
    "email": {
      "enabled": true,
      "address": "john.doe@example.com",
      "verified": true,
      "verifiedAt": "2024-01-01T10:00:00Z"
    },
    "sms": {
      "enabled": true,
      "number": "+1234567890",
      "verified": true,
      "verifiedAt": "2024-01-02T11:00:00Z"
    },
    "push": {
      "enabled": false,
      "devices": []
    }
  },
  "subscriptions": {
    "order_updates": true,
    "shipping_updates": true,
    "promotions": false,
    "product_updates": true,
    "price_alerts": true
  },
  "quiet_hours": {
    "enabled": true,
    "start": "22:00",
    "end": "08:00",
    "timezone": "America/New_York"
  },
  "language": "en",
  "updatedAt": "2024-01-10T15:00:00Z"
}
```

#### Update Preferences
```http
PUT /preferences/customer/{customerId}
Content-Type: application/json
Authorization: Bearer {token}

{
  "channels": {
    "sms": {
      "enabled": false
    }
  },
  "subscriptions": {
    "promotions": true,
    "price_alerts": false
  },
  "quiet_hours": {
    "enabled": true,
    "start": "21:00",
    "end": "09:00"
  }
}

Response:
{
  "customerId": "123e4567-e89b-12d3-a456-426614174000",
  "updated": true,
  "changes": {
    "channels.sms.enabled": false,
    "subscriptions.promotions": true,
    "subscriptions.price_alerts": false,
    "quiet_hours.start": "21:00",
    "quiet_hours.end": "09:00"
  },
  "updatedAt": "2024-01-15T10:45:00Z"
}
```

#### Unsubscribe
```http
POST /unsubscribe
Content-Type: application/json

{
  "token": "unsub-token-440e8400-e29b-41d4-a716-446655440000",
  "categories": ["promotions", "product_updates"],
  "reason": "Too many emails"
}

Response:
{
  "success": true,
  "customerId": "123e4567-e89b-12d3-a456-426614174000",
  "unsubscribed": ["promotions", "product_updates"],
  "remaining": ["order_updates", "shipping_updates"],
  "message": "You have been unsubscribed from the selected categories."
}
```

#### Get Notification History
```http
GET /notifications/customer/{customerId}?startDate=2024-01-01&endDate=2024-01-31&page=0&size=20
Authorization: Bearer {token}

Response:
{
  "notifications": [
    {
      "notificationId": "notif-880e8400-e29b-41d4-a716-446655440000",
      "template": "order_confirmation",
      "channels": ["email", "sms"],
      "status": "DELIVERED",
      "subject": "Order ORD-2024-001234 Confirmed!",
      "sentAt": "2024-01-15T10:00:30Z",
      "metadata": {
        "orderId": "ord-770e8400-e29b-41d4-a716-446655440000"
      }
    }
  ],
  "summary": {
    "total": 45,
    "delivered": 43,
    "failed": 2,
    "byChannel": {
      "email": {
        "sent": 45,
        "delivered": 43,
        "opened": 38,
        "clicked": 25
      },
      "sms": {
        "sent": 20,
        "delivered": 19
      }
    }
  },
  "page": 0,
  "totalPages": 3
}
```

#### Send Test Notification (Admin)
```http
POST /notifications/test
Content-Type: application/json
Authorization: Bearer {admin-token}

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
  }
}

Response:
{
  "testId": "test-550e8400-e29b-41d4-a716-446655440000",
  "status": "SENT",
  "preview": {
    "subject": "Order TEST-ORDER-123 Confirmed!",
    "bodySnippet": "Hi Test User, Your order TEST-ORDER-123 for $99.99 has been confirmed..."
  },
  "sentAt": "2024-01-15T11:00:00Z"
}
```

## Event Contracts

### Published Events

#### NotificationSentEvent
```json
{
  "eventId": "ev-110e8400-e29b-41d4-a716-446655440000",
  "notificationId": "notif-880e8400-e29b-41d4-a716-446655440000",
  "customerId": "123e4567-e89b-12d3-a456-426614174000",
  "template": "order_confirmation",
  "channels": ["email", "sms"],
  "metadata": {
    "orderId": "ord-770e8400-e29b-41d4-a716-446655440000"
  },
  "timestamp": "2024-01-15T10:00:30Z"
}
```

#### NotificationDeliveredEvent
```json
{
  "eventId": "ev-220e8400-e29b-41d4-a716-446655440000",
  "notificationId": "notif-880e8400-e29b-41d4-a716-446655440000",
  "channel": "email",
  "deliveredAt": "2024-01-15T10:00:45Z",
  "provider": "sendgrid",
  "providerMessageId": "sg-msg-123456",
  "timestamp": "2024-01-15T10:00:45Z"
}
```

#### NotificationFailedEvent
```json
{
  "eventId": "ev-330e8400-e29b-41d4-a716-446655440000",
  "notificationId": "notif-880e8400-e29b-41d4-a716-446655440000",
  "channel": "sms",
  "failureReason": "Invalid phone number",
  "provider": "twilio",
  "willRetry": false,
  "timestamp": "2024-01-15T10:01:00Z"
}
```

### Consumed Events

The Notification Service consumes events from all other services to send relevant notifications:

- `OrderCreatedEvent` - Send order confirmation
- `OrderShippedEvent` - Send shipping notification
- `OrderDeliveredEvent` - Send delivery confirmation
- `PaymentFailedEvent` - Send payment failure alert
- `LowStockAlertEvent` - Send restock notifications
- `PriceDropEvent` - Send price alert notifications

## Data Model

### Database Schema (MongoDB)

```javascript
// Templates collection
db.templates.createIndex({ "name": 1, "channel": 1 }, { unique: true })
db.templates.createIndex({ "active": 1 })
db.templates.createIndex({ "category": 1 })

{
  _id: ObjectId(),
  name: "order_confirmation",
  displayName: "Order Confirmation",
  channel: "email", // email, sms, push
  subject: "Order {{order.number}} Confirmed!",
  body: {
    html: "<html>...</html>",
    text: "Plain text version..."
  },
  variables: [
    {
      name: "order",
      type: "object",
      required: true,
      fields: [
        { name: "number", type: "string", required: true },
        { name: "totalAmount", type: "number", required: true },
        { name: "estimatedDelivery", type: "date", required: false }
      ]
    }
  ],
  metadata: {
    category: "transactional",
    tags: ["order", "confirmation"],
    description: "Sent when order is confirmed"
  },
  active: true,
  version: 3,
  created_at: ISODate("2024-01-01T10:00:00Z"),
  updated_at: ISODate("2024-01-10T14:30:00Z"),
  created_by: "admin-user-id",
  versions: [
    {
      version: 2,
      subject: "Your Order {{order.number}} is Confirmed!",
      body: { html: "...", text: "..." },
      created_at: ISODate("2024-01-05T10:00:00Z")
    }
  ]
}

// Notifications collection
db.notifications.createIndex({ "recipient.customerId": 1, "created_at": -1 })
db.notifications.createIndex({ "status": 1, "created_at": -1 })
db.notifications.createIndex({ "metadata.orderId": 1 })

{
  _id: ObjectId(),
  notification_id: "notif-880e8400-e29b-41d4-a716-446655440000",
  recipient: {
    customerId: "123e4567-e89b-12d3-a456-426614174000",
    email: "john.doe@example.com",
    phone: "+1234567890",
    deviceTokens: []
  },
  template_id: ObjectId("..."),
  template_name: "order_confirmation",
  context: {
    order: {
      number: "ORD-2024-001234",
      totalAmount: 1299.99,
      estimatedDelivery: "2024-01-22"
    },
    customer: {
      name: "John Doe",
      email: "john.doe@example.com"
    }
  },
  channels: [
    {
      channel: "email",
      status: "delivered",
      message_id: "email-990e8400-e29b-41d4-a716-446655440000",
      provider: "sendgrid",
      provider_message_id: "sg-msg-123456",
      subject: "Order ORD-2024-001234 Confirmed!",
      sent_at: ISODate("2024-01-15T10:00:30Z"),
      delivered_at: ISODate("2024-01-15T10:00:45Z"),
      opened_at: ISODate("2024-01-15T11:30:00Z"),
      clicked_at: ISODate("2024-01-15T11:31:00Z"),
      attempts: 1,
      last_error: null
    },
    {
      channel: "sms",
      status: "delivered",
      message_id: "sms-aa0e8400-e29b-41d4-a716-446655440000",
      provider: "twilio",
      provider_message_id: "SM123456",
      sent_at: ISODate("2024-01-15T10:00:35Z"),
      delivered_at: ISODate("2024-01-15T10:00:40Z"),
      attempts: 1
    }
  ],
  status: "delivered", // queued, processing, delivered, partial, failed
  metadata: {
    orderId: "ord-770e8400-e29b-41d4-a716-446655440000",
    source: "order-service",
    priority: "high",
    event_type: "order_created"
  },
  created_at: ISODate("2024-01-15T10:00:00Z"),
  completed_at: ISODate("2024-01-15T10:00:45Z")
}

// Preferences collection
db.preferences.createIndex({ "customer_id": 1 }, { unique: true })

{
  _id: ObjectId(),
  customer_id: "123e4567-e89b-12d3-a456-426614174000",
  channels: {
    email: {
      enabled: true,
      address: "john.doe@example.com",
      verified: true,
      verified_at: ISODate("2024-01-01T10:00:00Z")
    },
    sms: {
      enabled: true,
      number: "+1234567890",
      verified: true,
      verified_at: ISODate("2024-01-02T11:00:00Z"),
      country_code: "US"
    },
    push: {
      enabled: false,
      devices: [
        {
          token: "device-token-123",
          platform: "ios",
          app_version: "1.2.3",
          added_at: ISODate("2024-01-03T12:00:00Z")
        }
      ]
    }
  },
  subscriptions: {
    order_updates: true,
    shipping_updates: true,
    promotions: false,
    product_updates: true,
    price_alerts: true,
    newsletter: false
  },
  quiet_hours: {
    enabled: true,
    start_time: "22:00",
    end_time: "08:00",
    timezone: "America/New_York",
    exclude_critical: true
  },
  language: "en",
  frequency_caps: {
    daily_limit: 10,
    weekly_limit: 50,
    promotional_daily_limit: 2
  },
  unsubscribe_tokens: [
    {
      token: "unsub-token-123",
      created_at: ISODate("2024-01-01T10:00:00Z"),
      categories: ["all"]
    }
  ],
  created_at: ISODate("2024-01-01T10:00:00Z"),
  updated_at: ISODate("2024-01-10T15:00:00Z")
}

// Provider logs collection (for debugging)
db.provider_logs.createIndex({ "notification_id": 1 })
db.provider_logs.createIndex({ "created_at": -1 }, { expireAfterSeconds: 2592000 }) // 30 days

{
  _id: ObjectId(),
  notification_id: "notif-880e8400-e29b-41d4-a716-446655440000",
  channel: "email",
  provider: "sendgrid",
  request: {
    endpoint: "https://api.sendgrid.com/v3/mail/send",
    headers: { /* ... */ },
    body: { /* ... */ }
  },
  response: {
    status_code: 202,
    headers: { /* ... */ },
    body: { /* ... */ }
  },
  duration_ms: 145,
  created_at: ISODate("2024-01-15T10:00:30Z")
}
```

### Entity Models

```java
@Document(collection = "templates")
@CompoundIndex(name = "name_channel_idx", def = "{'name': 1, 'channel': 1}", unique = true)
public class NotificationTemplate {
    @Id
    private String id;
    
    @Indexed
    private String name;
    
    private String displayName;
    
    @Indexed
    private NotificationChannel channel;
    
    private String subject;
    
    private TemplateBody body;
    
    private List<TemplateVariable> variables;
    
    private Map<String, Object> metadata;
    
    @Indexed
    private boolean active = true;
    
    private int version = 1;
    
    @CreatedDate
    private Instant createdAt;
    
    @LastModifiedDate
    private Instant updatedAt;
    
    private String createdBy;
    
    private List<TemplateVersion> versions = new ArrayList<>();
    
    // Methods
    public String render(Map<String, Object> context) {
        return templateEngine.render(this, context);
    }
    
    public void validate(Map<String, Object> context) {
        for (TemplateVariable variable : variables) {
            if (variable.isRequired() && !context.containsKey(variable.getName())) {
                throw new MissingTemplateVariableException(variable.getName());
            }
        }
    }
}

@Document(collection = "notifications")
public class Notification {
    @Id
    private String id;
    
    @Indexed
    private String notificationId;
    
    private Recipient recipient;
    
    private String templateId;
    
    private String templateName;
    
    private Map<String, Object> context;
    
    private List<ChannelDelivery> channels;
    
    @Indexed
    private NotificationStatus status;
    
    private Map<String, Object> metadata;
    
    @CreatedDate
    private Instant createdAt;
    
    private Instant completedAt;
    
    // Business methods
    public void markChannelSent(NotificationChannel channel, String messageId, String provider) {
        getChannelDelivery(channel).ifPresent(delivery -> {
            delivery.setStatus(DeliveryStatus.SENT);
            delivery.setSentAt(Instant.now());
            delivery.setProvider(provider);
            delivery.setMessageId(messageId);
        });
        updateOverallStatus();
    }
    
    public void markChannelDelivered(NotificationChannel channel, String providerMessageId) {
        getChannelDelivery(channel).ifPresent(delivery -> {
            delivery.setStatus(DeliveryStatus.DELIVERED);
            delivery.setDeliveredAt(Instant.now());
            delivery.setProviderMessageId(providerMessageId);
        });
        updateOverallStatus();
    }
    
    private void updateOverallStatus() {
        boolean allDelivered = channels.stream()
            .allMatch(c -> c.getStatus() == DeliveryStatus.DELIVERED);
        
        boolean anyDelivered = channels.stream()
            .anyMatch(c -> c.getStatus() == DeliveryStatus.DELIVERED);
            
        boolean allFailed = channels.stream()
            .allMatch(c -> c.getStatus() == DeliveryStatus.FAILED);
        
        if (allDelivered) {
            this.status = NotificationStatus.DELIVERED;
            this.completedAt = Instant.now();
        } else if (allFailed) {
            this.status = NotificationStatus.FAILED;
            this.completedAt = Instant.now();
        } else if (anyDelivered) {
            this.status = NotificationStatus.PARTIAL;
        }
    }
}

@Data
public class ChannelDelivery {
    private NotificationChannel channel;
    private DeliveryStatus status = DeliveryStatus.QUEUED;
    private String messageId;
    private String provider;
    private String providerMessageId;
    private String subject;
    private Instant sentAt;
    private Instant deliveredAt;
    private Instant openedAt;
    private Instant clickedAt;
    private int attempts = 0;
    private String lastError;
}

public enum NotificationChannel {
    EMAIL,
    SMS,
    PUSH,
    IN_APP
}

public enum NotificationStatus {
    QUEUED,
    PROCESSING,
    DELIVERED,
    PARTIAL,
    FAILED,
    CANCELLED
}
```

## Configuration

### Application Properties

```yaml
spring:
  application:
    name: notification-service
  
  data:
    mongodb:
      uri: mongodb://${MONGO_HOST:localhost}:27017/notifications
      auto-index-creation: true
  
  kafka:
    bootstrap-servers: ${KAFKA_BROKERS:localhost:9092}
    consumer:
      group-id: notification-service
      auto-offset-reset: earliest

notification:
  providers:
    email:
      primary: sendgrid
      fallback: aws-ses
      
    sms:
      primary: twilio
      fallback: aws-sns
      
    push:
      ios: apns
      android: fcm
  
  sendgrid:
    api-key: ${SENDGRID_API_KEY}
    from-email: noreply@firstviscount.com
    from-name: First Viscount
    webhook-secret: ${SENDGRID_WEBHOOK_SECRET}
    
  twilio:
    account-sid: ${TWILIO_ACCOUNT_SID}
    auth-token: ${TWILIO_AUTH_TOKEN}
    from-number: ${TWILIO_FROM_NUMBER}
    messaging-service-sid: ${TWILIO_MESSAGING_SERVICE_SID}
    
  aws:
    ses:
      region: us-east-1
      from-email: noreply@firstviscount.com
    sns:
      region: us-east-1
  
  retry:
    max-attempts: 3
    backoff-multiplier: 2
    initial-interval: PT1M
    max-interval: PT1H
    
  rate-limits:
    per-customer-daily: 50
    per-customer-hourly: 10
    promotional-daily: 5
    
  quiet-hours:
    respect-timezone: true
    default-timezone: America/New_York
    exclude-critical: true
```

### Environment Variables

```bash
# MongoDB
MONGO_HOST=mongodb
MONGO_USERNAME=notification_user
MONGO_PASSWORD=secure_password

# Kafka
KAFKA_BROKERS=kafka1:9092,kafka2:9092,kafka3:9092

# Email Providers
SENDGRID_API_KEY=SG.xxx
SENDGRID_WEBHOOK_SECRET=xxx
AWS_ACCESS_KEY_ID=xxx
AWS_SECRET_ACCESS_KEY=xxx

# SMS Providers
TWILIO_ACCOUNT_SID=ACxxx
TWILIO_AUTH_TOKEN=xxx
TWILIO_FROM_NUMBER=+1234567890
TWILIO_MESSAGING_SERVICE_SID=MGxxx

# Push Notifications
FCM_SERVER_KEY=xxx
APNS_KEY_ID=xxx
APNS_TEAM_ID=xxx
APNS_PRIVATE_KEY=xxx

# Security
JWT_SECRET=your-secret-key
ENCRYPTION_KEY=xxx
```

## Dependencies

### External Services
- **SendGrid** - Primary email provider
- **AWS SES** - Fallback email provider
- **Twilio** - Primary SMS provider
- **AWS SNS** - Fallback SMS provider
- **Firebase Cloud Messaging** - Android push notifications
- **Apple Push Notification Service** - iOS push notifications

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
        <artifactId>spring-boot-starter-data-mongodb</artifactId>
    </dependency>
    
    <!-- Kafka -->
    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka</artifactId>
    </dependency>
    
    <!-- Template Engine -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-thymeleaf</artifactId>
    </dependency>
    <dependency>
        <groupId>com.github.spullara.mustache.java</groupId>
        <artifactId>compiler</artifactId>
        <version>0.9.10</version>
    </dependency>
    
    <!-- Email Providers -->
    <dependency>
        <groupId>com.sendgrid</groupId>
        <artifactId>sendgrid-java</artifactId>
        <version>4.9.3</version>
    </dependency>
    <dependency>
        <groupId>software.amazon.awssdk</groupId>
        <artifactId>ses</artifactId>
        <version>2.20.0</version>
    </dependency>
    
    <!-- SMS Providers -->
    <dependency>
        <groupId>com.twilio.sdk</groupId>
        <artifactId>twilio</artifactId>
        <version>9.2.0</version>
    </dependency>
    
    <!-- Push Notifications -->
    <dependency>
        <groupId>com.google.firebase</groupId>
        <artifactId>firebase-admin</artifactId>
        <version>9.1.0</version>
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
void shouldRenderTemplateWithContext() {
    // Given
    NotificationTemplate template = NotificationTemplate.builder()
        .subject("Order {{order.number}} Confirmed!")
        .body(TemplateBody.builder()
            .text("Hi {{customer.name}}, your order {{order.number}} for ${{order.totalAmount}} is confirmed.")
            .build())
        .build();
        
    Map<String, Object> context = Map.of(
        "order", Map.of("number", "ORD-123", "totalAmount", 99.99),
        "customer", Map.of("name", "John Doe")
    );
    
    // When
    String rendered = templateEngine.render(template, context);
    
    // Then
    assertThat(rendered).contains("Order ORD-123 Confirmed!");
    assertThat(rendered).contains("Hi John Doe, your order ORD-123 for $99.99 is confirmed.");
}

@Test
void shouldRetryOnProviderFailure() {
    // Given
    when(sendGridClient.send(any())).thenThrow(new ProviderException("Service unavailable"));
    when(sesClient.send(any())).thenReturn(MessageResponse.success("ses-123"));
    
    // When
    NotificationResult result = notificationService.send(createEmailNotification());
    
    // Then
    assertThat(result.isSuccess()).isTrue();
    assertThat(result.getProvider()).isEqualTo("aws-ses");
    verify(sendGridClient, times(1)).send(any());
    verify(sesClient, times(1)).send(any());
}
```

### Integration Tests
```java
@SpringBootTest
@AutoConfigureMockMvc
class NotificationControllerIntegrationTest {
    
    @Test
    void shouldSendNotificationSuccessfully() throws Exception {
        // Send notification
        String response = mockMvc.perform(post("/api/v1/notifications/send")
                .contentType(MediaType.APPLICATION_JSON)
                .content(readJson("send-notification-request.json")))
                .andExpect(status().isOk())
                .andReturn()
                .getResponse()
                .getContentAsString();
                
        String notificationId = JsonPath.read(response, "$.notificationId");
        
        // Wait for processing
        Thread.sleep(2000);
        
        // Check status
        mockMvc.perform(get("/api/v1/notifications/" + notificationId))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.status").value("DELIVERED"));
    }
}
```

## Deployment

### Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: notification-service
  namespace: ecommerce
spec:
  replicas: 3
  selector:
    matchLabels:
      app: notification-service
  template:
    metadata:
      labels:
        app: notification-service
    spec:
      containers:
      - name: notification
        image: firstviscount/notification-service:latest
        ports:
        - containerPort: 8086
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "production"
        - name: MONGO_HOST
          valueFrom:
            configMapKeyRef:
              name: mongodb-config
              key: host
        - name: SENDGRID_API_KEY
          valueFrom:
            secretKeyRef:
              name: notification-secrets
              key: sendgrid-api-key
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
            port: 8086
          initialDelaySeconds: 30
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8086
          initialDelaySeconds: 20
```

## Monitoring

### Health Checks

```java
@Component
public class NotificationProvidersHealthIndicator implements HealthIndicator {
    
    @Override
    public Health health() {
        Map<String, Object> details = new HashMap<>();
        
        // Check email providers
        try {
            sendGridClient.checkHealth();
            details.put("sendgrid", "UP");
        } catch (Exception e) {
            details.put("sendgrid", "DOWN");
        }
        
        // Check SMS providers
        try {
            twilioClient.checkHealth();
            details.put("twilio", "UP");
        } catch (Exception e) {
            details.put("twilio", "DOWN");
        }
        
        // Check MongoDB
        try {
            mongoTemplate.getDb().runCommand(new Document("ping", 1));
            details.put("mongodb", "UP");
        } catch (Exception e) {
            details.put("mongodb", "DOWN");
        }
        
        boolean hasWorkingProvider = details.values().stream()
            .anyMatch("UP"::equals);
            
        return hasWorkingProvider 
            ? Health.up().withDetails(details).build()
            : Health.down().withDetails(details).build();
    }
}
```

### Metrics

Key metrics exposed:
- `notifications.sent.total` - Total notifications sent by channel
- `notifications.delivered.total` - Total delivered notifications
- `notifications.failed.total` - Failed notifications by reason
- `notifications.delivery.time` - Time to deliver notifications
- `provider.requests.total` - Provider API calls
- `template.render.time` - Template rendering performance

### Alerts

```yaml
groups:
  - name: notification-service
    rules:
      - alert: HighNotificationFailureRate
        expr: rate(notifications_failed_total[5m]) > 0.1
        for: 5m
        annotations:
          summary: "High notification failure rate"
          
      - alert: ProviderDown
        expr: provider_health{provider="sendgrid"} == 0
        for: 5m
        annotations:
          summary: "Email provider SendGrid is down"
          
      - alert: NotificationQueueBacklog
        expr: notification_queue_size > 1000
        for: 10m
        annotations:
          summary: "Large notification queue backlog"
```

## Best Practices

1. **Template Versioning** - Keep template history for rollback
2. **Provider Fallback** - Always have backup providers
3. **Rate Limiting** - Respect provider and customer limits
4. **Retry Logic** - Implement exponential backoff
5. **Preference Management** - Honor customer preferences
6. **Compliance** - Include unsubscribe links, respect regulations
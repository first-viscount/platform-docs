# Delivery API Reference

## Overview

The Delivery API handles all aspects of order fulfillment including shipping label generation, carrier integration, package tracking, and delivery confirmation. It supports multiple shipping carriers and provides real-time tracking updates for a seamless delivery experience.

**Base URL:** `http://delivery-service:8085/api/v1`  
**Port:** 8085

## Authentication

All endpoints require JWT authentication:

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

## OpenAPI Specification

```yaml
openapi: 3.0.3
info:
  title: Delivery Management API
  version: 1.0.0
  description: Comprehensive shipping and delivery management system
servers:
  - url: http://delivery-service:8085/api/v1
    description: Internal service endpoint
  - url: https://api.firstviscount.com/delivery/v1
    description: Production API gateway
```

## Endpoints

### Shipment Management

#### Create Shipment

Create a new shipment with shipping labels and tracking.

```http
POST /shipments
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
  "orderId": "ord-770e8400-e29b-41d4-a716-446655440000",
  "items": [
    {
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "productSku": "LAPTOP-001",
      "productName": "Premium Laptop Pro 15",
      "quantity": 2,
      "weight": 3.5,
      "dimensions": {
        "length": 15,
        "width": 10,
        "height": 5,
        "unit": "INCHES"
      },
      "value": 599.99,
      "hsCode": "8471.30.0100"
    }
  ],
  "origin": {
    "warehouseId": "wh-001",
    "address": {
      "name": "First Viscount Warehouse",
      "company": "First Viscount Inc",
      "street": "100 Warehouse Way",
      "streetLine2": "Building A",
      "city": "Dallas",
      "state": "TX",
      "zipCode": "75201",
      "country": "US",
      "phone": "+1-555-0123",
      "email": "warehouse@firstviscount.com"
    }
  },
  "destination": {
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
    "isResidential": true,
    "deliveryInstructions": "Leave at front door"
  },
  "shippingMethod": "GROUND",
  "carrier": "FEDEX",
  "service": "FEDEX_GROUND",
  "insurance": {
    "required": true,
    "value": 1199.98,
    "provider": "CARRIER"
  },
  "options": {
    "signatureRequired": true,
    "saturdayDelivery": false,
    "hazardousMaterials": false,
    "fragile": true,
    "giftWrap": false
  },
  "metadata": {
    "priority": "NORMAL",
    "rush": false,
    "customerNotes": "Handle with care"
  }
}
```

**Request Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| orderId | string (UUID) | Yes | Associated order ID |
| items | array | Yes | Items to ship |
| origin | object | Yes | Shipping origin address |
| destination | object | Yes | Shipping destination address |
| shippingMethod | string | Yes | GROUND, EXPRESS, OVERNIGHT |
| carrier | string | No | Preferred carrier |
| insurance | object | No | Insurance requirements |
| options | object | No | Shipping options |

**Success Response (201 Created):**

```json
{
  "shipmentId": "ship-880e8400-e29b-41d4-a716-446655440000",
  "shipmentNumber": "SHIP-2024-001234",
  "orderId": "ord-770e8400-e29b-41d4-a716-446655440000",
  "carrier": "FEDEX",
  "service": "FedEx Ground",
  "serviceCode": "FEDEX_GROUND",
  "trackingNumber": "1234567890123",
  "status": "LABEL_CREATED",
  "packages": [
    {
      "packageId": "pkg-001",
      "packageNumber": 1,
      "trackingNumber": "1234567890123",
      "weight": 7.0,
      "dimensions": {
        "length": 15,
        "width": 10,
        "height": 5,
        "unit": "INCHES"
      },
      "items": [
        {
          "productId": "550e8400-e29b-41d4-a716-446655440000",
          "quantity": 2
        }
      ]
    }
  ],
  "cost": {
    "base": 12.99,
    "fuel": 1.30,
    "insurance": 5.00,
    "signature": 3.00,
    "residential": 4.50,
    "total": 26.79,
    "currency": "USD"
  },
  "delivery": {
    "estimatedDelivery": {
      "earliest": "2024-01-18",
      "latest": "2024-01-20"
    },
    "guaranteedDelivery": "2024-01-20",
    "deliveryDays": 3,
    "cutoffTime": "15:00"
  },
  "labels": [
    {
      "type": "SHIPPING_LABEL",
      "format": "PDF",
      "size": "4x6",
      "url": "https://api.firstviscount.com/labels/ship-880e8400-001.pdf",
      "base64": null
    },
    {
      "type": "COMMERCIAL_INVOICE",
      "format": "PDF",
      "url": "https://api.firstviscount.com/labels/ship-880e8400-invoice.pdf"
    }
  ],
  "tracking": {
    "url": "https://www.fedex.com/fedextrack/?trknbr=1234567890123",
    "apiUrl": "https://api.firstviscount.com/delivery/v1/shipments/track/1234567890123"
  },
  "createdAt": "2024-01-15T10:00:00Z",
  "estimatedProcessingTime": "PT2H"
}
```

**Error Responses:**

- **400 Bad Request** - Invalid shipment data
- **401 Unauthorized** - Missing or invalid authentication
- **402 Payment Required** - Insufficient shipping credits
- **409 Conflict** - Duplicate shipment (idempotency key conflict)
- **422 Unprocessable Entity** - Address validation failed

#### Get Shipment Details

Retrieve detailed information about a specific shipment.

```http
GET /shipments/{shipmentId}
```

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| shipmentId | string (UUID) | Yes | Shipment identifier |

**Success Response (200 OK):**

```json
{
  "shipmentId": "ship-880e8400-e29b-41d4-a716-446655440000",
  "shipmentNumber": "SHIP-2024-001234",
  "orderId": "ord-770e8400-e29b-41d4-a716-446655440000",
  "carrier": "FEDEX",
  "service": "FedEx Ground",
  "trackingNumber": "1234567890123",
  "status": "IN_TRANSIT",
  "statusHistory": [
    {
      "status": "LABEL_CREATED",
      "timestamp": "2024-01-15T10:00:00Z",
      "location": "Dallas, TX",
      "description": "Shipping label created"
    },
    {
      "status": "PICKED_UP",
      "timestamp": "2024-01-15T14:00:00Z",
      "location": "Dallas, TX",
      "description": "Package picked up by FedEx"
    },
    {
      "status": "IN_TRANSIT",
      "timestamp": "2024-01-16T08:00:00Z",
      "location": "Memphis, TN",
      "description": "Package departed FedEx hub",
      "facilityName": "FedEx Memphis Hub"
    }
  ],
  "currentLocation": {
    "city": "Memphis",
    "state": "TN",
    "country": "US",
    "facilityName": "FedEx Memphis Hub",
    "timestamp": "2024-01-16T08:00:00Z",
    "timeZone": "America/Chicago"
  },
  "addresses": {
    "origin": {
      "name": "First Viscount Warehouse",
      "street": "100 Warehouse Way",
      "city": "Dallas",
      "state": "TX",
      "zipCode": "75201",
      "country": "US"
    },
    "destination": {
      "name": "John Doe",
      "street": "123 Main St",
      "city": "New York",
      "state": "NY",
      "zipCode": "10001",
      "country": "US"
    }
  },
  "packages": [
    {
      "packageId": "pkg-001",
      "trackingNumber": "1234567890123",
      "weight": 7.0,
      "status": "IN_TRANSIT"
    }
  ],
  "delivery": {
    "estimatedDelivery": "2024-01-18",
    "actualDelivery": null,
    "deliveryAttempts": 0,
    "lastAttempt": null,
    "nextAttempt": null
  },
  "timeline": {
    "created": "2024-01-15T10:00:00Z",
    "pickedUp": "2024-01-15T14:00:00Z",
    "inTransit": "2024-01-16T08:00:00Z",
    "outForDelivery": null,
    "delivered": null
  },
  "cost": {
    "total": 26.79,
    "currency": "USD"
  },
  "insurance": {
    "insured": true,
    "value": 1199.98,
    "provider": "FEDEX"
  },
  "options": {
    "signatureRequired": true,
    "residential": true
  },
  "createdAt": "2024-01-15T10:00:00Z",
  "updatedAt": "2024-01-16T08:15:00Z"
}
```

**Error Responses:**

- **401 Unauthorized** - Missing or invalid authentication
- **404 Not Found** - Shipment does not exist

### Shipment Tracking

#### Track Shipment by Tracking Number

Get real-time tracking information for a shipment.

```http
GET /shipments/track/{trackingNumber}
```

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| trackingNumber | string | Yes | Carrier tracking number |

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| includeHistory | boolean | No | Include full tracking history |
| timezone | string | No | Timezone for timestamps |

**Success Response (200 OK):**

```json
{
  "trackingNumber": "1234567890123",
  "shipmentId": "ship-880e8400-e29b-41d4-a716-446655440000",
  "carrier": "FEDEX",
  "service": "FedEx Ground",
  "status": "OUT_FOR_DELIVERY",
  "statusDescription": "Out for delivery",
  "currentLocation": {
    "description": "On FedEx vehicle for delivery",
    "city": "New York",
    "state": "NY",
    "country": "US",
    "coordinates": {
      "latitude": 40.7128,
      "longitude": -74.0060
    },
    "timestamp": "2024-01-18T08:30:00Z",
    "timeZone": "America/New_York"
  },
  "estimatedDelivery": {
    "date": "2024-01-18",
    "timeWindow": {
      "start": "09:00",
      "end": "17:00"
    },
    "confidence": "HIGH"
  },
  "delivery": {
    "attempts": 0,
    "maxAttempts": 3,
    "holdAtLocation": false,
    "specialInstructions": "Leave at front door"
  },
  "events": [
    {
      "timestamp": "2024-01-18T08:30:00Z",
      "status": "OUT_FOR_DELIVERY",
      "description": "Out for delivery",
      "location": {
        "city": "New York",
        "state": "NY",
        "country": "US"
      },
      "eventCode": "OFD"
    },
    {
      "timestamp": "2024-01-18T05:00:00Z",
      "status": "AT_FACILITY",
      "description": "At local FedEx facility",
      "location": {
        "city": "New York",
        "state": "NY",
        "facilityName": "FedEx Ground - Long Island City"
      },
      "eventCode": "AF"
    },
    {
      "timestamp": "2024-01-17T23:45:00Z",
      "status": "IN_TRANSIT",
      "description": "Departed FedEx location",
      "location": {
        "city": "Newark",
        "state": "NJ"
      },
      "eventCode": "DP"
    }
  ],
  "deliveryExceptions": [],
  "lastUpdated": "2024-01-18T08:35:00Z",
  "nextUpdate": "2024-01-18T12:00:00Z"
}
```

#### Track Multiple Shipments

Track multiple shipments in a single request.

```http
POST /shipments/track/batch
```

**Request Body:**

```json
{
  "trackingNumbers": [
    "1234567890123",
    "9876543210987",
    "5555555555555"
  ],
  "includeHistory": false
}
```

**Success Response (200 OK):**

```json
{
  "trackingResults": [
    {
      "trackingNumber": "1234567890123",
      "status": "OUT_FOR_DELIVERY",
      "estimatedDelivery": "2024-01-18",
      "lastUpdate": "2024-01-18T08:30:00Z"
    },
    {
      "trackingNumber": "9876543210987",
      "status": "DELIVERED",
      "deliveredAt": "2024-01-17T14:30:00Z",
      "signedBy": "J. SMITH"
    },
    {
      "trackingNumber": "5555555555555",
      "error": {
        "code": "TRACKING_NOT_FOUND",
        "message": "Tracking information not available"
      }
    }
  ]
}
```

### Shipping Rates

#### Calculate Shipping Rates

Get shipping rates from multiple carriers for comparison.

```http
POST /shipping/calculate
```

**Request Body:**

```json
{
  "origin": {
    "zipCode": "75201",
    "city": "Dallas",
    "state": "TX",
    "country": "US"
  },
  "destination": {
    "zipCode": "10001",
    "city": "New York",
    "state": "NY", 
    "country": "US",
    "residential": true
  },
  "packages": [
    {
      "weight": 5.0,
      "weightUnit": "LBS",
      "dimensions": {
        "length": 12,
        "width": 8,
        "height": 6,
        "unit": "INCHES"
      },
      "value": 599.99,
      "description": "Electronics"
    }
  ],
  "services": ["GROUND", "EXPRESS", "OVERNIGHT"],
  "carriers": ["FEDEX", "UPS", "USPS"],
  "options": {
    "insurance": {
      "required": true,
      "value": 599.99
    },
    "signatureRequired": true,
    "saturdayDelivery": false
  },
  "shipDate": "2024-01-15"
}
```

**Success Response (200 OK):**

```json
{
  "rates": [
    {
      "rateId": "rate-001",
      "carrier": "FEDEX",
      "service": "FedEx Ground",
      "serviceCode": "FEDEX_GROUND",
      "cost": {
        "base": 12.99,
        "fuel": 1.30,
        "insurance": 2.50,
        "signature": 3.00,
        "residential": 4.50,
        "total": 24.29,
        "currency": "USD"
      },
      "transit": {
        "days": 3,
        "businessDays": 3,
        "estimatedDelivery": "2024-01-18"
      },
      "guarantees": {
        "moneyBack": true,
        "onTime": false
      },
      "options": {
        "residential": true,
        "signatureRequired": true,
        "saturdayDelivery": false
      }
    },
    {
      "rateId": "rate-002",
      "carrier": "FEDEX",
      "service": "FedEx 2Day",
      "serviceCode": "FEDEX_2_DAY",
      "cost": {
        "base": 24.99,
        "fuel": 2.50,
        "insurance": 2.50,
        "signature": 3.00,
        "residential": 4.50,
        "total": 37.49,
        "currency": "USD"
      },
      "transit": {
        "days": 2,
        "businessDays": 2,
        "estimatedDelivery": "2024-01-17"
      },
      "guarantees": {
        "moneyBack": true,
        "onTime": true
      }
    },
    {
      "rateId": "rate-003",
      "carrier": "UPS",
      "service": "UPS Ground",
      "serviceCode": "UPS_GROUND",
      "cost": {
        "base": 13.49,
        "fuel": 1.35,
        "insurance": 2.75,
        "signature": 3.25,
        "residential": 4.95,
        "total": 25.79,
        "currency": "USD"
      },
      "transit": {
        "days": 3,
        "businessDays": 3,
        "estimatedDelivery": "2024-01-18"
      }
    }
  ],
  "metadata": {
    "requestId": "req-123",
    "calculatedAt": "2024-01-15T10:00:00Z",
    "validUntil": "2024-01-15T12:00:00Z"
  }
}
```

### Delivery Confirmation

#### Confirm Delivery

Manually confirm delivery of a shipment.

```http
POST /shipments/{shipmentId}/confirm
```

**Request Body:**

```json
{
  "deliveredAt": "2024-01-18T14:30:00Z",
  "signedBy": "J. DOE",
  "deliveryLocation": "FRONT_DOOR",
  "deliveryMethod": "LEFT_AT_DOOR",
  "proofOfDelivery": {
    "imageUrl": "https://cdn.firstviscount.com/pod/ship-880e8400.jpg",
    "signatureUrl": "https://cdn.firstviscount.com/signatures/ship-880e8400.png",
    "gpsCoordinates": {
      "latitude": 40.7589,
      "longitude": -73.9851
    }
  },
  "notes": "Package left at front door as requested",
  "confirmedBy": "DRIVER",
  "driverName": "Mike Johnson",
  "driverId": "DRV-12345"
}
```

**Success Response (200 OK):**

```json
{
  "shipmentId": "ship-880e8400-e29b-41d4-a716-446655440000",
  "status": "DELIVERED",
  "deliveredAt": "2024-01-18T14:30:00Z",
  "confirmationNumber": "DEL-2024011814300123",
  "deliveryDetails": {
    "signedBy": "J. DOE",
    "location": "FRONT_DOOR",
    "method": "LEFT_AT_DOOR",
    "driverName": "Mike Johnson"
  },
  "proofOfDelivery": {
    "available": true,
    "imageId": "img-123",
    "signatureId": "sig-456"
  },
  "notifications": {
    "customerNotified": true,
    "channels": ["email", "sms"]
  }
}
```

### Returns Management

#### Create Return Shipment

Create a return shipping label for customer returns.

```http
POST /returns
```

**Request Body:**

```json
{
  "originalShipmentId": "ship-880e8400-e29b-41d4-a716-446655440000",
  "orderId": "ord-770e8400-e29b-41d4-a716-446655440000",
  "returnReason": "DEFECTIVE",
  "reasonDetails": "Screen has dead pixels",
  "items": [
    {
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "quantity": 1,
      "condition": "DEFECTIVE",
      "returnValue": 599.99
    }
  ],
  "returnAddress": {
    "name": "First Viscount Returns",
    "company": "First Viscount Inc",
    "street": "200 Returns Way",
    "city": "Dallas",
    "state": "TX",
    "zipCode": "75202",
    "country": "US"
  },
  "pickup": {
    "required": true,
    "date": "2024-01-20",
    "timeWindow": {
      "start": "09:00",
      "end": "17:00"
    },
    "specialInstructions": "Ring doorbell"
  },
  "packaging": {
    "useOriginal": true,
    "includeReturnLabel": true
  }
}
```

**Success Response (201 Created):**

```json
{
  "returnId": "ret-990e8400-e29b-41d4-a716-446655440000",
  "returnNumber": "RMA-2024-001234",
  "shipmentId": "ship-aa0e8400-e29b-41d4-a716-446655440000",
  "carrier": "FEDEX",
  "service": "FedEx Ground",
  "trackingNumber": "9876543210987",
  "status": "LABEL_CREATED",
  "returnAddress": {
    "name": "First Viscount Returns",
    "street": "200 Returns Way",
    "city": "Dallas",
    "state": "TX",
    "zipCode": "75202"
  },
  "labels": [
    {
      "type": "RETURN_LABEL",
      "format": "PDF",
      "size": "4x6",
      "url": "https://api.firstviscount.com/labels/ret-990e8400.pdf"
    }
  ],
  "pickup": {
    "scheduled": true,
    "date": "2024-01-20",
    "timeWindow": {
      "start": "09:00",
      "end": "17:00"
    },
    "confirmationNumber": "PICKUP-123456",
    "cost": 0.00
  },
  "returnPolicy": {
    "windowDays": 30,
    "refundMethod": "ORIGINAL_PAYMENT",
    "restockingFee": 0.00
  },
  "instructions": [
    "Pack item securely in original packaging",
    "Attach return label to outside of package",
    "Schedule pickup or drop off at any FedEx location"
  ],
  "createdAt": "2024-01-19T10:00:00Z",
  "expiresAt": "2024-02-18T23:59:59Z"
}
```

#### Get Return Status

Check the status of a return shipment.

```http
GET /returns/{returnId}
```

**Success Response (200 OK):**

```json
{
  "returnId": "ret-990e8400-e29b-41d4-a716-446655440000",
  "returnNumber": "RMA-2024-001234",
  "status": "IN_TRANSIT",
  "trackingNumber": "9876543210987",
  "returnShipment": {
    "status": "IN_TRANSIT",
    "currentLocation": "Memphis, TN",
    "estimatedDelivery": "2024-01-22"
  },
  "timeline": {
    "labelCreated": "2024-01-19T10:00:00Z",
    "pickedUp": "2024-01-20T14:00:00Z",
    "inTransit": "2024-01-21T08:00:00Z",
    "delivered": null,
    "processed": null
  },
  "processing": {
    "status": "PENDING",
    "estimatedProcessingDays": 3,
    "refundAmount": 599.99,
    "refundMethod": "ORIGINAL_PAYMENT"
  }
}
```

### Delivery Analytics

#### Get Delivery Metrics

Retrieve delivery performance metrics and analytics.

```http
GET /metrics/deliveries
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| startDate | date | No | Start date for metrics |
| endDate | date | No | End date for metrics |
| carrier | string | No | Filter by carrier |
| service | string | No | Filter by service type |
| groupBy | string | No | Group results by (day, week, month) |

**Success Response (200 OK):**

```json
{
  "period": {
    "start": "2024-01-01",
    "end": "2024-01-31",
    "days": 31
  },
  "summary": {
    "totalShipments": 12543,
    "totalDelivered": 11234,
    "inTransit": 1098,
    "exceptions": 211,
    "delivered": 11234,
    "cancelled": 56
  },
  "performance": {
    "onTimeDeliveryRate": 94.5,
    "averageDeliveryDays": 2.8,
    "firstAttemptDeliveryRate": 92.1,
    "customerSatisfactionScore": 4.6,
    "damageRate": 0.3
  },
  "deliveryAttempts": {
    "firstAttempt": 10456,
    "secondAttempt": 698,
    "thirdAttempt": 80,
    "averageAttempts": 1.08
  },
  "carrierBreakdown": {
    "FEDEX": {
      "shipments": 7234,
      "onTimeRate": 95.2,
      "averageCost": 14.56,
      "marketShare": 57.7
    },
    "UPS": {
      "shipments": 4521,
      "onTimeRate": 93.8,
      "averageCost": 13.89,
      "marketShare": 36.1
    },
    "USPS": {
      "shipments": 788,
      "onTimeRate": 89.2,
      "averageCost": 8.45,
      "marketShare": 6.2
    }
  },
  "serviceBreakdown": {
    "GROUND": {
      "shipments": 8234,
      "onTimeRate": 94.2,
      "averageCost": 12.34
    },
    "EXPRESS": {
      "shipments": 3456,
      "onTimeRate": 96.8,
      "averageCost": 24.56
    },
    "OVERNIGHT": {
      "shipments": 853,
      "onTimeRate": 98.1,
      "averageCost": 45.67
    }
  },
  "costs": {
    "total": 182456.78,
    "average": 14.54,
    "median": 12.99,
    "byService": {
      "GROUND": 101234.56,
      "EXPRESS": 56789.12,
      "OVERNIGHT": 24433.10
    }
  },
  "geography": {
    "topDestinations": [
      {
        "state": "CA",
        "shipments": 2345,
        "percentage": 18.7
      },
      {
        "state": "NY",
        "shipments": 1876,
        "percentage": 15.0
      }
    ],
    "internationalShipments": 456,
    "domesticShipments": 12087
  },
  "exceptions": {
    "weatherDelay": 89,
    "addressIssue": 67,
    "customerUnavailable": 34,
    "damageReported": 21
  }
}
```

## Error Handling

### Error Response Format

```json
{
  "error": {
    "code": "INVALID_ADDRESS",
    "message": "Destination address validation failed",
    "details": {
      "field": "destination.zipCode",
      "value": "invalid_zip",
      "reason": "Invalid ZIP code format"
    },
    "suggestions": [
      {
        "street": "123 Main St",
        "city": "New York",
        "state": "NY",
        "zipCode": "10001"
      }
    ],
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
| SHIPMENT_NOT_FOUND | Shipment does not exist | 404 |
| TRACKING_NOT_FOUND | Tracking info not available | 404 |
| INVALID_ADDRESS | Address validation failed | 422 |
| CARRIER_ERROR | Carrier API error | 502 |
| SERVICE_UNAVAILABLE | Carrier service down | 503 |
| RATE_LIMIT_EXCEEDED | Too many requests | 429 |
| PACKAGE_TOO_LARGE | Package exceeds limits | 422 |
| HAZARDOUS_MATERIALS | Restricted materials | 422 |
| INSUFFICIENT_CREDITS | Not enough shipping credits | 402 |
| DUPLICATE_SHIPMENT | Idempotency key conflict | 409 |
| DELIVERY_EXCEPTION | Delivery failed | 422 |
| RETURN_EXPIRED | Return window expired | 410 |

## Webhooks

### Event Types

- `shipment.created` - New shipment created
- `shipment.shipped` - Package picked up
- `shipment.in_transit` - Package in transit
- `shipment.out_for_delivery` - Out for delivery
- `shipment.delivered` - Successfully delivered
- `shipment.exception` - Delivery exception
- `return.created` - Return label created
- `return.received` - Return package received

### Webhook Payload

```json
{
  "id": "webhook-001",
  "timestamp": "2024-01-18T14:30:00Z",
  "event": "shipment.delivered",
  "data": {
    "shipmentId": "ship-880e8400-e29b-41d4-a716-446655440000",
    "trackingNumber": "1234567890123",
    "orderId": "ord-770e8400-e29b-41d4-a716-446655440000",
    "carrier": "FEDEX",
    "status": "DELIVERED",
    "deliveredAt": "2024-01-18T14:30:00Z",
    "signedBy": "J. DOE",
    "location": "Front Door",
    "proofOfDelivery": {
      "available": true,
      "imageUrl": "https://cdn.firstviscount.com/pod/ship-880e8400.jpg"
    }
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
import { DeliveryClient } from '@firstviscount/delivery-sdk';

const client = new DeliveryClient({
  baseUrl: 'https://api.firstviscount.com/delivery/v1',
  apiKey: process.env.API_KEY
});

// Create shipment
const shipment = await client.shipments.create({
  orderId: 'ord-123',
  items: [
    {
      productId: 'prod-456',
      quantity: 1,
      weight: 2.5,
      dimensions: { length: 10, width: 8, height: 4, unit: 'INCHES' }
    }
  ],
  origin: { warehouseId: 'wh-001' },
  destination: {
    name: 'John Doe',
    street: '123 Main St',
    city: 'New York',
    state: 'NY',
    zipCode: '10001',
    country: 'US'
  },
  shippingMethod: 'GROUND'
});

// Track shipment
const tracking = await client.tracking.track('1234567890123');

// Calculate rates
const rates = await client.rates.calculate({
  origin: { zipCode: '75201', country: 'US' },
  destination: { zipCode: '10001', country: 'US' },
  packages: [{ weight: 5.0, dimensions: { length: 12, width: 8, height: 6 } }]
});
```

### Python

```python
from firstviscount import DeliveryClient

client = DeliveryClient(
    base_url='https://api.firstviscount.com/delivery/v1',
    api_key=os.environ['API_KEY']
)

# Create return shipment
return_shipment = client.returns.create(
    original_shipment_id='ship-880e8400-e29b-41d4-a716-446655440000',
    order_id='ord-770e8400-e29b-41d4-a716-446655440000',
    return_reason='DEFECTIVE',
    items=[{
        'product_id': '550e8400-e29b-41d4-a716-446655440000',
        'quantity': 1
    }],
    pickup_required=True
)

# Get delivery metrics
metrics = client.metrics.get_delivery_metrics(
    start_date='2024-01-01',
    end_date='2024-01-31',
    carrier='FEDEX'
)
```

## Testing

### Test Environment

```
Base URL: https://api-test.firstviscount.com/delivery/v1
```

### Test Tracking Numbers

| Tracking Number | Status | Carrier |
|----------------|--------|---------|
| TEST1234567890 | DELIVERED | FEDEX |
| TEST9876543210 | IN_TRANSIT | UPS |
| TEST5555555555 | OUT_FOR_DELIVERY | USPS |

## Changelog

### Version 1.0.0 (2024-01-15)

- Initial release
- Multi-carrier shipping support
- Real-time tracking
- Return label generation
- Delivery analytics
- Webhook notifications
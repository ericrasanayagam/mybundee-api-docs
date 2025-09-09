---
layout: default
title: API Reference
nav_order: 2
---

# API Reference
{: .no_toc }

Complete reference for all MyBundee API endpoints
{: .fs-6 .fw-300 }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Authentication

All API requests require authentication using JWT tokens or Firebase authentication.

### Headers Required
```http
Authorization: Bearer {your-jwt-token}
Content-Type: application/json
appId: BUNDEE
```

### Base URLs

| Environment | URL |
|-------------|-----|
| **Development** | `http://localhost:7443/api` |
| **QA** | `https://qabundeeusermanagement.azurewebsites.net/api` |
| **Production** | `https://bundeeusermanagement.azurewebsites.net/api` |

---

## User Management

### POST /v1/user/register

Create a new user account.

**Request Body:**
```json
{
  "email": "user@example.com",
  "firstName": "John",
  "lastName": "Doe",
  "password": "SecurePass123",
  "phoneNumber": "+1234567890",
  "userRole": "DRIVER",
  "address": {
    "street": "123 Main St",
    "city": "Austin",
    "state": "TX",
    "zipcode": "78701"
  }
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "userId": 123,
    "email": "user@example.com",
    "message": "User created successfully"
  }
}
```

### POST /v1/user/checklogin

Authenticate user and create session.

**Request Body:**
```json
{
  "email": "user@example.com",
  "password": "password123"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "user": {
      "id": 123,
      "email": "user@example.com",
      "firstName": "John",
      "lastName": "Doe",
      "userRole": "DRIVER"
    },
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

---

## Vehicle Search

### POST /v1/availability/getByZipCode

Search for available vehicles by location and time.

**Request Body:**
```json
{
  "zipCode": "78701",
  "latitude": 30.2672,
  "longitude": -97.7431,
  "startTime": "2024-01-15T10:00:00Z",
  "endTime": "2024-01-17T18:00:00Z",
  "userId": 123,
  "radius": 10,
  "vehicleType": ["SEDAN", "SUV"],
  "priceRange": {
    "min": 50,
    "max": 200
  }
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "vehicleAllDetails": [
      {
        "id": 250,
        "make": "Toyota",
        "model": "Camry",
        "year": 2022,
        "pricePerHour": 15.00,
        "pricePerDay": 80.00,
        "location": {
          "address": "123 Main St",
          "city": "Austin",
          "state": "TX",
          "latitude": 30.2672,
          "longitude": -97.7431
        },
        "features": ["GPS", "Bluetooth", "USB"],
        "images": ["image1.jpg", "image2.jpg"],
        "rating": 4.8,
        "hostName": "John's Fleet"
      }
    ],
    "totalResults": 15
  }
}
```

### POST /v1/availability/getVehiclesnFeaturesById

Get detailed information for a specific vehicle.

**Request Body:**
```json
{
  "vehicleid": 250,
  "userId": 123
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "vehicleAllDetails": [
      {
        "id": 250,
        "make": "Toyota",
        "model": "Camry",
        "year": 2022,
        "vin": "1HGBH41JXMN109186",
        "licensePlate": "ABC123",
        "description": "Clean, reliable sedan perfect for city driving",
        "seatingCapacity": 5,
        "transmission": "Automatic",
        "fuelType": "Gasoline",
        "features": {
          "hasGPS": true,
          "hasBluetooth": true,
          "hasUSB": true,
          "hasWifi": false,
          "hasBackupCamera": true
        },
        "pricing": {
          "pricePerHour": 15.00,
          "pricePerDay": 80.00,
          "pricePerWeek": 500.00
        },
        "location": {
          "address": "123 Main St",
          "city": "Austin",
          "state": "TX",
          "zipcode": "78701",
          "latitude": 30.2672,
          "longitude": -97.7431
        },
        "images": [
          "https://images.mybundee.com/vehicles/250/front.jpg",
          "https://images.mybundee.com/vehicles/250/interior.jpg"
        ],
        "availability": {
          "isAvailable": true,
          "nextAvailableDate": "2024-01-15T00:00:00Z"
        }
      }
    ]
  }
}
```

---

## Booking Management

### POST /v1/booking/createReservation

Create a new vehicle reservation.

**Request Body:**
```json
{
  "vehicleId": 250,
  "userId": 123,
  "hostId": 456,
  "startTime": "2024-01-15T10:00:00Z",
  "endTime": "2024-01-17T18:00:00Z",
  "pickupLocation": {
    "address": "123 Main St",
    "city": "Austin",
    "state": "TX",
    "zipcode": "78701",
    "latitude": 30.2672,
    "longitude": -97.7431
  },
  "delivery": false,
  "airportDelivery": false,
  "customDelivery": false,
  "notes": "First time renting, please provide detailed instructions",
  "paymentMethodId": "pm_1234567890",
  "protectionPlanId": 1
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "reservationId": 789,
    "tripId": "TRIP_789_2024",
    "status": "CONFIRMED",
    "totalAmount": 160.00,
    "securityDeposit": 200.00,
    "confirmationNumber": "BND789456",
    "pickupInstructions": "Vehicle will be available at the specified location 15 minutes before your pickup time."
  }
}
```

### POST /v1/booking/getTripsbyUserId

Get user's trip history and active bookings.

**Request Body:**
```json
{
  "fromValue": "useractivebookings",
  "id": 123
}
```

**Available fromValue options:**
- `useractivebookings` - Active trips only
- `userupcomingbookings` - Upcoming trips
- `userhistorybookings` - Completed trips
- `userallbookings` - All trips

**Response:**
```json
{
  "success": true,
  "data": {
    "trips": [
      {
        "tripId": 789,
        "vehicleId": 250,
        "vehicleName": "2022 Toyota Camry",
        "startTime": "2024-01-15T10:00:00Z",
        "endTime": "2024-01-17T18:00:00Z",
        "status": "CONFIRMED",
        "totalAmount": 160.00,
        "location": "Austin, TX",
        "hostName": "John's Fleet",
        "canModify": true,
        "canCancel": true
      }
    ],
    "totalTrips": 5
  }
}
```

---

## Payment Processing

### POST /api/checkout/create-setup-intent

Create a Stripe setup intent for payment method registration.

**Request Body:**
```json
{
  "customerId": "cus_1234567890",
  "userId": 123
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "clientSecret": "seti_1234567890_secret_abc123",
    "setupIntentId": "seti_1234567890"
  }
}
```

### POST /api/checkout/process-payment

Process a payment for a trip.

**Request Body:**
```json
{
  "paymentMethodId": "pm_1234567890",
  "amount": 16000,
  "currency": "usd",
  "tripId": 789,
  "description": "Vehicle rental payment for Trip #789"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "paymentIntentId": "pi_1234567890",
    "status": "succeeded",
    "amount": 16000,
    "receiptUrl": "https://pay.stripe.com/receipts/..."
  }
}
```

---

## Chat System

### POST /api/send-chat-notification

Send SMS notification when chat is initiated.

**Request Body:**
```json
{
  "userInfo": {
    "name": "John Doe",
    "email": "john@example.com",
    "phone": "+1234567890"
  },
  "conversationId": "CHxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "userAgent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)..."
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "message": "Chat notification sent successfully",
    "results": {
      "successful": 2,
      "failed": 0,
      "details": [
        {
          "phoneNumber": "+1234567890",
          "success": true,
          "messageSid": "SMxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
        }
      ]
    }
  }
}
```

---

## Error Handling

### Standard Error Response

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request parameters",
    "details": {
      "field": "email",
      "issue": "Invalid email format"
    }
  },
  "timestamp": "2024-01-15T10:00:00Z"
}
```

### Common Error Codes

| Code | Description | HTTP Status |
|------|-------------|-------------|
| `AUTH_REQUIRED` | Authentication required | 401 |
| `ACCESS_DENIED` | Insufficient permissions | 403 |
| `NOT_FOUND` | Resource not found | 404 |
| `VALIDATION_ERROR` | Invalid request data | 400 |
| `VEHICLE_UNAVAILABLE` | Vehicle not available | 409 |
| `PAYMENT_FAILED` | Payment processing failed | 402 |
| `RATE_LIMIT_EXCEEDED` | Too many requests | 429 |

---

## Rate Limits

| Endpoint Category | Limit | Window |
|-------------------|-------|--------|
| Search APIs | 100 requests | per minute |
| Booking APIs | 20 requests | per minute |
| Authentication | 10 attempts | per minute |
| File Upload | 5 uploads | per minute |

---

## SDKs and Libraries

### TypeScript/JavaScript
```bash
npm install @mybundee/api-client
```

### Python
```bash
pip install mybundee-api
```

### cURL Examples

**Search Vehicles:**
```bash
curl -X POST https://qabundeeavailability.azurewebsites.net/api/v1/availability/getByZipCode \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "zipCode": "78701",
    "startTime": "2024-01-15T10:00:00Z",
    "endTime": "2024-01-17T18:00:00Z"
  }'
```

**Create Booking:**
```bash
curl -X POST https://qabundeebooking.azurewebsites.net/api/v1/booking/createReservation \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "vehicleId": 250,
    "startTime": "2024-01-15T10:00:00Z",
    "endTime": "2024-01-17T18:00:00Z",
    "paymentMethodId": "pm_1234567890"
  }'
```
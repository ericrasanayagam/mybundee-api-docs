---
layout: default
title: Home
nav_order: 1
description: "MyBundee API Documentation - Complete guide for developers"
permalink: /
---

# MyBundee API Documentation
{: .fs-9 }

Complete developer guide for integrating with the MyBundee car-sharing platform
{: .fs-6 .fw-300 }

[Get Started](#getting-started){: .btn .btn-primary .fs-5 .mb-4 .mb-md-0 .mr-2 }
[View on GitHub](https://github.com/mybundee/mybundee-api-docs){: .btn .fs-5 .mb-4 .mb-md-0 }

---

## Overview

MyBundee is a comprehensive car-sharing platform built with modern microservices architecture. This documentation provides everything you need to integrate with our APIs, from basic authentication to advanced features like real-time chat and payment processing.

### Platform Architecture

- **Backend**: Java microservices + Node.js auxiliary services
- **Frontend**: Next.js with TypeScript and React
- **Authentication**: JWT sessions with Firebase integration
- **Payments**: Stripe integration with setup intents
- **Communication**: Twilio Conversations with SMS notifications
- **Maps**: Mapbox integration for location services

## Quick Start

### 1. Get API Credentials
Contact the MyBundee team to obtain your API credentials and access tokens.

### 2. Set Up Authentication
```typescript
// Configure your authentication
const session = await createSession({
  userId: user.id,
  email: user.email,
  userRole: user.role
});
```

### 3. Make Your First API Call
```typescript
// Search for available vehicles
const vehicles = await searchVehicles({
  zipCode: "78701",
  startTime: "2024-01-15T10:00:00Z",
  endTime: "2024-01-17T10:00:00Z"
});
```

## Core Services

| Service | Description | Base URL |
|---------|-------------|----------|
| **User Management** | Authentication, profiles, verification | `/api/v1/user` |
| **Vehicle Search** | Availability, search, details | `/api/v1/availability` |
| **Booking** | Reservations, trips, modifications | `/api/v1/booking` |
| **Payments** | Stripe integration, billing | `/api/v1/payment` |
| **Chat** | Support chat with Twilio | `/api/v1/chat` |

## Getting Started

1. [Authentication Guide]({% link authentication.md %})
2. [API Reference]({% link api-reference.md %})
3. [Integration Examples]({% link integration-examples.md %})
4. [How-To Guides]({% link how-to-guides.md %})

## Support

- **Documentation Issues**: [GitHub Issues](https://github.com/mybundee/mybundee-api-docs/issues)
- **API Support**: Contact your MyBundee integration team
- **Developer Portal**: [MyBundee Developer Hub](https://developers.mybundee.com)

---

Last updated: {{ site.time | date: "%B %d, %Y" }}
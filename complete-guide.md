# MyBundee Platform - Complete API Guide & How-To Documentation

## Table of Contents

1. [Platform Overview](#platform-overview)
2. [Authentication & Authorization](#authentication--authorization)
3. [Core API Services](#core-api-services)
4. [How-To Guides](#how-to-guides)
5. [Integration Examples](#integration-examples)
6. [Error Handling](#error-handling)
7. [Testing Guide](#testing-guide)
8. [Deployment & Configuration](#deployment--configuration)

---

## Platform Overview

MyBundee is a comprehensive car-sharing platform built with:

- **Backend**: Java microservices + Node.js auxiliary services
- **Frontend**: Next.js 14+ with TypeScript and React 18+
- **Authentication**: JWT sessions with Firebase integration
- **Payments**: Stripe integration with setup intents
- **Communication**: Twilio Conversations with SMS notifications
- **Maps**: Mapbox integration for location services
- **Database**: PostgreSQL with connection pooling

### Service Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   DriverWeb     │    │  Admin Portal   │    │   Host Portal   │
│  (Next.js 14)   │    │  (Next.js 15)   │    │  (Angular 17)   │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 │
    ┌────────────────────────────┼────────────────────────────┐
    │                            │                            │
┌───▼────┐  ┌─────────┐  ┌──────▼──┐  ┌──────────┐  ┌─────────┐
│ User   │  │ Booking │  │Vehicle  │  │Availability│  │Location │
│Service │  │Service  │  │Service  │  │  Service   │  │Service  │
│:7443   │  │:7443    │  │:9443    │  │   :6443    │  │  :8443  │
└────────┘  └─────────┘  └─────────┘  └────────────┘  └─────────┘
```

---

## Authentication & Authorization

### JWT Session Management

MyBundee uses JWT-based authentication with automatic refresh and secure cookie storage:

```typescript
// Authentication Configuration
interface SessionConfig {
  cookieName: string;           // session-qa, session-prod
  encryptionSecret: string;     // 32-char encryption key
  authSecret: string;           // JWT signing secret
  maxAge: number;              // 24 hours default
}

// Session Structure
interface UserSession {
  userId: number;
  email: string;
  firstName: string;
  lastName: string;
  phoneNumber: string;
  userRole: 'DRIVER' | 'HOST' | 'EMPLOYEE' | 'ADMIN';
  createdAt: string;
  expiresAt: string;
}
```

### Firebase Integration

```typescript
// Firebase Authentication Setup
import { initializeApp } from 'firebase/app';
import { getAuth, signInWithEmailAndPassword } from 'firebase/auth';

const firebaseConfig = {
  apiKey: process.env.NEXT_PUBLIC_FIREBASE_APIKEY,
  authDomain: process.env.NEXT_PUBLIC_FIREBASE_AUTHDOMAIN,
  projectId: process.env.NEXT_PUBLIC_FIREBASE_PROJECTID,
  // ... other config
};

const app = initializeApp(firebaseConfig);
export const auth = getAuth(app);

// Firebase Auth Integration
async function signInUser(email: string, password: string) {
  const userCredential = await signInWithEmailAndPassword(auth, email, password);
  const idToken = await userCredential.user.getIdToken();
  
  // Exchange Firebase token for MyBundee session
  const response = await fetch('/api/auth/firebase-login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ idToken })
  });
  
  return response.json();
}
```

---

## Core API Services

### 1. User Management Service

#### Base URL Configuration
```bash
# Environment Variables
USER_MANAGEMENT_BASEURL=https://qabundeeusermanagement.azurewebsites.net/api
# Production: https://bundeeusermanagement.azurewebsites.net/api
```

#### Authentication Endpoints

**User Registration**
```typescript
// POST /api/v1/user/register
interface UserRegistrationData {
  email: string;
  firstName: string;
  lastName: string;
  password: string;
  phoneNumber: string;
  address?: {
    street: string;
    city: string;
    state: string;
    zipcode: string;
  };
  userRole: 'DRIVER' | 'HOST';
}

async function registerUser(userData: UserRegistrationData) {
  const response = await fetch(`${USER_MANAGEMENT_BASEURL}/v1/user/register`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'appId': 'BUNDEE'
    },
    body: JSON.stringify(userData)
  });
  
  return response.json();
}
```

**User Login**
```typescript
// POST /api/v1/user/checklogin
async function loginUser(email: string, password: string) {
  const response = await fetch(`${USER_MANAGEMENT_BASEURL}/v1/user/checklogin`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'appId': 'BUNDEE'
    },
    body: JSON.stringify({ email, password })
  });
  
  const result = await response.json();
  
  if (result.success) {
    // Create session cookie
    await createSession(result.user);
  }
  
  return result;
}
```

#### Profile Management

**Update Driver Profile**
```typescript
// PUT /api/v1/user/updateUserProfile
interface ProfileUpdateData {
  userId: number;
  firstName: string;
  lastName: string;
  email: string;
  phoneNumber: string;
  address: {
    street: string;
    city: string;
    state: string;
    zipcode: string;
  };
  dateOfBirth: string;
  drivingLicenseNumber: string;
  licenseState: string;
  licenseExpiryDate: string;
}

async function updateProfile(profileData: ProfileUpdateData) {
  const session = await getSession();
  
  const response = await fetch(`${USER_MANAGEMENT_BASEURL}/v1/user/updateUserProfile`, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${session.token}`,
      'appId': 'BUNDEE'
    },
    body: JSON.stringify(profileData)
  });
  
  return response.json();
}
```

**Phone Number Verification**
```typescript
// POST /api/v1/user/phone/verify
async function verifyPhoneNumber(phoneNumber: string, verificationCode: string) {
  const session = await getSession();
  
  const response = await fetch(`${USER_MANAGEMENT_BASEURL}/v1/user/phone/verify`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${session.token}`
    },
    body: JSON.stringify({
      phoneNumber,
      verificationCode,
      userId: session.userId
    })
  });
  
  return response.json();
}
```

### 2. Vehicle Search & Availability Service

#### Base URL Configuration
```bash
AVAILABILITY_BASEURL=https://qabundeeavailability.azurewebsites.net/api
```

#### Vehicle Search Endpoints

**Search by Location**
```typescript
// POST /api/v1/availability/getByZipCode
interface VehicleSearchQuery {
  zipCode: string;
  latitude: number;
  longitude: number;
  startTime: string;  // ISO 8601 format
  endTime: string;    // ISO 8601 format
  userId: number;
  radius?: number;    // Search radius in miles
  vehicleType?: string[];
  priceRange?: {
    min: number;
    max: number;
  };
}

async function searchVehicles(searchQuery: VehicleSearchQuery) {
  const response = await http.post(
    `${AVAILABILITY_BASEURL}/v1/availability/getByZipCode`,
    searchQuery
  );
  
  return response.data;
}
```

**Search by Coordinates**
```typescript
// POST /api/v1/availability/searchVehiclesByLatitudeAndLongitude
async function searchByCoordinates(searchQuery: {
  latitude: number;
  longitude: number;
  startTime: string;
  endTime: string;
  radius: number;
}) {
  const response = await http.post(
    `${AVAILABILITY_BASEURL}/v1/availability/searchVehiclesByLatitudeAndLongitude`,
    searchQuery
  );
  
  return response.data;
}
```

**Get Vehicle Details**
```typescript
// POST /api/v1/availability/getVehiclesnFeaturesById
async function getVehicleDetails(vehicleId: number, userId?: number) {
  const response = await http.post(
    `${AVAILABILITY_BASEURL}/v1/availability/getVehiclesnFeaturesById`,
    {
      vehicleid: vehicleId,
      userId: userId || ''
    }
  );
  
  // Add to recently viewed history
  if (userId) {
    await addToRecentlyViewed(vehicleId);
  }
  
  return response.data;
}
```

#### Availability Management

**Get Vehicle Availability Calendar**
```typescript
// POST /api/v1/availability/getAvailabilityDatesByVehicleId
async function getAvailabilityDates(vehicleId: number, tripId?: number) {
  const payload = tripId ? {
    reservationId: tripId,
    vehicleid: vehicleId
  } : { vehicleid: vehicleId };
  
  const response = await http.post(
    `${AVAILABILITY_BASEURL}/v1/availability/getAvailabilityDatesByVehicleId`,
    payload
  );
  
  return response.data;
}
```

### 3. Booking & Trip Management Service

#### Base URL Configuration
```bash
BOOKING_SERVICES_BASEURL=https://qabundeebooking.azurewebsites.net/api
```

#### Trip Creation & Management

**Create New Reservation**
```typescript
// POST /api/v1/booking/createReservation
interface ReservationData {
  vehicleId: number;
  userId: number;
  hostId: number;
  startTime: string;
  endTime: string;
  pickupLocation: {
    address: string;
    city: string;
    state: string;
    zipcode: string;
    latitude: number;
    longitude: number;
  };
  dropoffLocation?: {
    address: string;
    city: string;
    state: string;
    zipcode: string;
    latitude: number;
    longitude: number;
  };
  delivery: boolean;
  airportDelivery: boolean;
  customDelivery: boolean;
  notes?: string;
  paymentMethodId: string;
  protectionPlanId?: number;
}

async function createReservation(reservationData: ReservationData) {
  const session = await getSession();
  
  const response = await http.post(
    `${BOOKING_SERVICES_BASEURL}/v1/booking/createReservation`,
    {
      ...reservationData,
      userId: session.userId
    }
  );
  
  return response.data;
}
```

**Get User Trips**
```typescript
// POST /api/v1/booking/getTripsbyUserId
async function getUserTrips(fromValue: string = 'useractivebookings') {
  const session = await getSession();
  
  const response = await http.post(
    `${BOOKING_SERVICES_BASEURL}/v1/booking/getTripsbyUserId`,
    {
      fromValue,
      id: session.userId
    }
  );
  
  return response.data;
}
```

**Trip Status Updates**
```typescript
// POST /api/v1/booking/updateTripStatus
interface TripStatusUpdate {
  tripId: number;
  status: 'STARTED' | 'COMPLETED' | 'CANCELLED';
  comments?: string;
  location?: {
    latitude: number;
    longitude: number;
  };
  mileage?: number;
  fuelLevel?: number;
  images?: string[];
}

async function updateTripStatus(updateData: TripStatusUpdate) {
  const session = await getSession();
  
  const response = await http.post(
    `${BOOKING_SERVICES_BASEURL}/v1/booking/updateTripStatus`,
    {
      ...updateData,
      changedBy: 'DRIVER',
      userId: session.userId
    }
  );
  
  return response.data;
}
```

#### Trip Modifications

**Modify Reservation Dates**
```typescript
// POST /api/v1/booking/modification/updateReservation
interface TripModification {
  tripId: number;
  newStartTime: string;
  newEndTime: string;
  reason: string;
}

async function modifyTrip(modificationData: TripModification) {
  const session = await getSession();
  
  // First, check availability for new dates
  const availability = await checkAvailability({
    vehicleId: modificationData.vehicleId,
    startTime: modificationData.newStartTime,
    endTime: modificationData.newEndTime
  });
  
  if (availability.isAvailable) {
    const response = await http.post(
      `${BOOKING_SERVICES_BASEURL}/v1/booking/modification/updateReservation`,
      {
        ...modificationData,
        userId: session.userId
      }
    );
    
    return response.data;
  } else {
    throw new Error('Vehicle not available for requested dates');
  }
}
```

### 4. Payment Processing (Stripe Integration)

#### Stripe Configuration

```typescript
// Stripe Environment Variables
const STRIPE_PUBLISHABLE_KEY = process.env.NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY;
const STRIPE_SECRET_KEY = process.env.STRIPE_SECRET_KEY;

// Stripe Client Setup
import { loadStripe } from '@stripe/stripe-js';

const stripePromise = loadStripe(STRIPE_PUBLISHABLE_KEY!);
```

#### Payment Methods

**Create Setup Intent**
```typescript
// POST /api/checkout/create-setup-intent
async function createSetupIntent(customerId?: string) {
  const session = await getSession();
  
  const response = await fetch('/api/checkout/create-setup-intent', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      customerId,
      userId: session.userId
    })
  });
  
  return response.json();
}
```

**Process Payment**
```typescript
// POST /api/checkout/process-payment
interface PaymentData {
  paymentMethodId: string;
  amount: number;
  currency: string;
  tripId: number;
  description: string;
}

async function processPayment(paymentData: PaymentData) {
  const session = await getSession();
  
  const response = await fetch('/api/checkout/process-payment', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      ...paymentData,
      userId: session.userId
    })
  });
  
  return response.json();
}
```

### 5. Enhanced Chat System

#### Twilio Configuration

```typescript
// Twilio Environment Variables
const TWILIO_ACCOUNT_SID = process.env.TWILIO_ACCOUNT_SID;
const TWILIO_API_KEY = process.env.TWILIO_API_KEY;
const TWILIO_API_SECRET = process.env.TWILIO_API_SECRET;
const TWILIO_CHAT_SERVICE_SID = process.env.TWILIO_CHAT_SERVICE_SID;
const TWILIO_PHONE_NUMBER = process.env.TWILIO_PHONE_NUMBER;
const CHAT_NOTIFICATION_PHONES = process.env.CHAT_NOTIFICATION_PHONES;
```

#### Chat Operations

**Initialize Chat Session**
```typescript
// Using Twilio Conversations SDK
import { Client as ConversationsClient } from '@twilio/conversations';

async function initializeChat(userInfo: { name: string }) {
  // Get access token from Twilio Function
  const tokenResponse = await fetch(process.env.NEXT_PUBLIC_TWILIO_TOKEN_FUNCTION_URL!, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      identity: `customer_${userInfo.name.toLowerCase().replace(/\s+/g, '_')}_${Date.now()}`,
      userAgent: navigator.userAgent,
      userInfo
    })
  });
  
  const { token, identity } = await tokenResponse.json();
  
  // Initialize Twilio client
  const client = new ConversationsClient(token, { region: 'us1' });
  
  // Wait for client initialization
  await new Promise((resolve, reject) => {
    client.on('initialized', resolve);
    client.on('initFailed', reject);
  });
  
  // Get or create conversation
  const conversationName = `support_${identity}`;
  let conversation;
  
  try {
    conversation = await client.getConversationByUniqueName(conversationName);
  } catch (error) {
    // Create new conversation
    conversation = await client.createConversation({
      uniqueName: conversationName,
      friendlyName: userInfo.name,
      attributes: { customerName: userInfo.name }
    });
    
    await conversation.join();
    
    // Send welcome message
    await conversation.sendMessage(
      `Welcome to Bundee Support, ${userInfo.name}! How can we help you today?`,
      { author: 'system' }
    );
    
    // Send SMS notification to support team
    await sendChatNotification({
      userInfo,
      conversationId: conversation.sid
    });
  }
  
  return { client, conversation, identity };
}
```

**SMS Notifications**
```typescript
// POST /api/send-chat-notification
async function sendChatNotification(data: {
  userInfo: { name: string; email?: string; phone?: string };
  conversationId: string;
  userAgent?: string;
}) {
  const response = await fetch('/api/send-chat-notification', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data)
  });
  
  return response.json();
}
```

---

## How-To Guides

### 1. Setting Up a New Integration

#### Prerequisites
- Node.js 18+ with pnpm
- Access to MyBundee staging/production environments
- Valid API credentials and environment variables

#### Step 1: Environment Configuration

Create a `.env.local` file with all required variables:

```bash
# Authentication
AUTH_SECRET="your-jwt-secret-key"
ENCRYPTION_SECRET="your-32-char-encryption-key"
NEXT_PUBLIC_COOKIE_NAME="session-local"

# MyBundee Services
USER_MANAGEMENT_BASEURL="https://qabundeeusermanagement.azurewebsites.net/api"
BOOKING_SERVICES_BASEURL="https://qabundeebooking.azurewebsites.net/api"
AVAILABILITY_BASEURL="https://qabundeeavailability.azurewebsites.net/api"
HOST_SERVICES_BASEURL="https://qabundeehostvehicle.azurewebsites.net/api"
AUXILIARY_SERVICE_BASEURL="https://bundee-auxiliary-services-qa.azurewebsites.net/"

# Third-party Services
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY="pk_test_..."
STRIPE_SECRET_KEY="sk_test_..."
NEXT_PUBLIC_MAPBOX_ACCESS_TOKEN="pk.eyJ1..."
TWILIO_ACCOUNT_SID="ACxxxxxxxxxxxxx"
TWILIO_API_KEY="SKxxxxxxxxxxxxx"
TWILIO_API_SECRET="your-api-secret"

# Firebase
NEXT_PUBLIC_FIREBASE_APIKEY="your-api-key"
NEXT_PUBLIC_FIREBASE_AUTHDOMAIN="your-project.firebaseapp.com"
NEXT_PUBLIC_FIREBASE_PROJECTID="your-project-id"
```

#### Step 2: HTTP Service Setup

Create a centralized HTTP service with authentication:

```typescript
// lib/httpService.ts
import axios, { AxiosInstance, AxiosRequestConfig, AxiosResponse } from 'axios';
import { getSession } from './auth';

class HttpService {
  private instance: AxiosInstance;

  constructor() {
    this.instance = axios.create({
      timeout: 30000,
      headers: {
        'Content-Type': 'application/json',
        'appId': 'BUNDEE'
      }
    });

    // Request interceptor for authentication
    this.instance.interceptors.request.use(
      async (config) => {
        const session = await getSession();
        if (session?.token) {
          config.headers.Authorization = `Bearer ${session.token}`;
        }
        return config;
      },
      (error) => Promise.reject(error)
    );

    // Response interceptor for error handling
    this.instance.interceptors.response.use(
      (response) => response,
      async (error) => {
        if (error.response?.status === 401) {
          // Handle token expiration
          await signOut();
          window.location.href = '/sign-in';
        }
        return Promise.reject(error);
      }
    );
  }

  async get<T>(url: string, config?: AxiosRequestConfig): Promise<AxiosResponse<T>> {
    return this.instance.get(url, config);
  }

  async post<T>(url: string, data?: any, config?: AxiosRequestConfig): Promise<AxiosResponse<T>> {
    return this.instance.post(url, data, config);
  }

  async put<T>(url: string, data?: any, config?: AxiosRequestConfig): Promise<AxiosResponse<T>> {
    return this.instance.put(url, data, config);
  }

  async delete<T>(url: string, config?: AxiosRequestConfig): Promise<AxiosResponse<T>> {
    return this.instance.delete(url, config);
  }
}

export const http = new HttpService();
```

#### Step 3: Authentication Setup

Implement session management with JWT and Firebase:

```typescript
// lib/auth.ts
import { SignJWT, jwtVerify } from 'jose';
import { cookies } from 'next/headers';
import { NextRequest, NextResponse } from 'next/server';

const secret = new TextEncoder().encode(process.env.AUTH_SECRET);
const cookieName = process.env.NEXT_PUBLIC_COOKIE_NAME!;

export async function createSession(userData: any) {
  const payload = {
    userId: userData.id,
    email: userData.email,
    firstName: userData.firstName,
    lastName: userData.lastName,
    userRole: userData.userRole,
    createdAt: new Date().toISOString()
  };

  const token = await new SignJWT(payload)
    .setProtectedHeader({ alg: 'HS256' })
    .setExpirationTime('24h')
    .sign(secret);

  cookies().set(cookieName, token, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    maxAge: 86400 // 24 hours
  });

  return payload;
}

export async function getSession() {
  const cookieStore = cookies();
  const token = cookieStore.get(cookieName)?.value;

  if (!token) return null;

  try {
    const { payload } = await jwtVerify(token, secret);
    return payload as any;
  } catch (error) {
    return null;
  }
}

export async function signOut() {
  cookies().delete(cookieName);
}
```

### 2. Vehicle Search Integration

#### Basic Vehicle Search

```typescript
// hooks/useVehicleSearch.ts
import { useState, useCallback } from 'react';
import { http } from '@/lib/httpService';

interface SearchFilters {
  zipCode?: string;
  latitude?: number;
  longitude?: number;
  startTime: string;
  endTime: string;
  vehicleType?: string[];
  priceRange?: { min: number; max: number };
  features?: string[];
}

export function useVehicleSearch() {
  const [vehicles, setVehicles] = useState([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const searchVehicles = useCallback(async (filters: SearchFilters) => {
    setLoading(true);
    setError(null);

    try {
      const response = await http.post(
        `${process.env.AVAILABILITY_BASEURL}/v1/availability/getByZipCode`,
        filters
      );

      if (response.data.success) {
        setVehicles(response.data.vehicleAllDetails || []);
      } else {
        setError(response.data.message || 'Search failed');
      }
    } catch (err: any) {
      setError(err.message || 'Network error occurred');
    } finally {
      setLoading(false);
    }
  }, []);

  return { vehicles, loading, error, searchVehicles };
}
```

#### Advanced Search with Filters

```typescript
// components/VehicleSearch.tsx
import React, { useState } from 'react';
import { useVehicleSearch } from '@/hooks/useVehicleSearch';

export function VehicleSearchComponent() {
  const { vehicles, loading, error, searchVehicles } = useVehicleSearch();
  const [filters, setFilters] = useState({
    zipCode: '',
    startTime: '',
    endTime: '',
    vehicleType: [] as string[],
    priceRange: { min: 0, max: 500 }
  });

  const handleSearch = async (e: React.FormEvent) => {
    e.preventDefault();
    
    if (!filters.startTime || !filters.endTime) {
      alert('Please select pickup and return dates');
      return;
    }

    await searchVehicles({
      ...filters,
      startTime: new Date(filters.startTime).toISOString(),
      endTime: new Date(filters.endTime).toISOString()
    });
  };

  return (
    <div className="vehicle-search">
      <form onSubmit={handleSearch}>
        <div className="grid grid-cols-2 gap-4">
          <input
            type="text"
            placeholder="Zip Code"
            value={filters.zipCode}
            onChange={(e) => setFilters({...filters, zipCode: e.target.value})}
          />
          
          <input
            type="datetime-local"
            value={filters.startTime}
            onChange={(e) => setFilters({...filters, startTime: e.target.value})}
          />
          
          <input
            type="datetime-local"
            value={filters.endTime}
            onChange={(e) => setFilters({...filters, endTime: e.target.value})}
          />
        </div>
        
        <button type="submit" disabled={loading}>
          {loading ? 'Searching...' : 'Search Vehicles'}
        </button>
      </form>

      {error && <div className="error">{error}</div>}

      <div className="vehicle-results">
        {vehicles.map((vehicle: any) => (
          <VehicleCard key={vehicle.id} vehicle={vehicle} />
        ))}
      </div>
    </div>
  );
}
```

### 3. Booking Workflow Implementation

#### Complete Booking Flow

```typescript
// hooks/useBookingFlow.ts
import { useState, useCallback } from 'react';
import { http } from '@/lib/httpService';

export function useBookingFlow() {
  const [bookingState, setBookingState] = useState({
    step: 1, // 1: Dates, 2: Details, 3: Payment, 4: Confirmation
    selectedVehicle: null,
    bookingDetails: null,
    pricing: null,
    paymentMethod: null
  });

  const calculatePricing = useCallback(async (vehicleId: number, dates: {
    startTime: string;
    endTime: string;
  }) => {
    try {
      const response = await http.post(
        `${process.env.AUXILIARY_SERVICE_BASEURL}/api/v1/priceCalculation`,
        {
          vehicleid: vehicleId,
          startTime: dates.startTime,
          endTime: dates.endTime,
          airportDelivery: false,
          customDelivery: false
        }
      );

      setBookingState(prev => ({ ...prev, pricing: response.data }));
      return response.data;
    } catch (error) {
      console.error('Pricing calculation failed:', error);
      throw error;
    }
  }, []);

  const createBooking = useCallback(async (bookingData: any) => {
    try {
      const response = await http.post(
        `${process.env.BOOKING_SERVICES_BASEURL}/v1/booking/createReservation`,
        bookingData
      );

      if (response.data.success) {
        setBookingState(prev => ({ ...prev, step: 4 }));
        return response.data;
      } else {
        throw new Error(response.data.message);
      }
    } catch (error) {
      console.error('Booking creation failed:', error);
      throw error;
    }
  }, []);

  return {
    bookingState,
    setBookingState,
    calculatePricing,
    createBooking
  };
}
```

### 4. Chat System Integration

#### Setting Up Chat Widget

```typescript
// components/ChatWidget.tsx
import React, { useState, useEffect } from 'react';
import { Client as ConversationsClient } from '@twilio/conversations';

export function ChatWidget() {
  const [isOpen, setIsOpen] = useState(false);
  const [messages, setMessages] = useState([]);
  const [newMessage, setNewMessage] = useState('');
  const [client, setClient] = useState<ConversationsClient | null>(null);
  const [conversation, setConversation] = useState(null);

  const initializeChat = async (userInfo: { name: string }) => {
    try {
      // Get Twilio token
      const tokenResponse = await fetch('/api/twilio/token', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ userInfo })
      });

      const { token, identity } = await tokenResponse.json();

      // Initialize client
      const twilioClient = new ConversationsClient(token);
      await new Promise((resolve, reject) => {
        twilioClient.on('initialized', resolve);
        twilioClient.on('initFailed', reject);
      });

      // Get or create conversation
      const conversationName = `support_${identity}`;
      let conv;
      
      try {
        conv = await twilioClient.getConversationByUniqueName(conversationName);
      } catch {
        conv = await twilioClient.createConversation({
          uniqueName: conversationName,
          friendlyName: userInfo.name
        });
        await conv.join();
      }

      // Load messages
      const msgs = await conv.getMessages();
      setMessages(msgs.items);

      // Listen for new messages
      conv.on('messageAdded', (message) => {
        setMessages(prev => [...prev, message]);
      });

      setClient(twilioClient);
      setConversation(conv);
    } catch (error) {
      console.error('Chat initialization failed:', error);
    }
  };

  const sendMessage = async () => {
    if (!conversation || !newMessage.trim()) return;

    try {
      await conversation.sendMessage(newMessage.trim());
      setNewMessage('');
    } catch (error) {
      console.error('Failed to send message:', error);
    }
  };

  return (
    <div className={`chat-widget ${isOpen ? 'open' : 'closed'}`}>
      <div className="chat-header" onClick={() => setIsOpen(!isOpen)}>
        <span>Support Chat</span>
      </div>

      {isOpen && (
        <div className="chat-content">
          <div className="messages">
            {messages.map((msg, index) => (
              <div key={index} className="message">
                <strong>{msg.author}:</strong> {msg.body}
              </div>
            ))}
          </div>

          <div className="message-input">
            <input
              type="text"
              value={newMessage}
              onChange={(e) => setNewMessage(e.target.value)}
              onKeyPress={(e) => e.key === 'Enter' && sendMessage()}
              placeholder="Type your message..."
            />
            <button onClick={sendMessage}>Send</button>
          </div>
        </div>
      )}
    </div>
  );
}
```

### 5. Error Handling Best Practices

#### Centralized Error Handling

```typescript
// lib/errorHandler.ts
export class APIError extends Error {
  constructor(
    message: string,
    public statusCode: number,
    public code?: string
  ) {
    super(message);
    this.name = 'APIError';
  }
}

export function handleAPIResponse<T>(response: any): T {
  if (response.success) {
    return response.data;
  } else {
    throw new APIError(
      response.message || 'API request failed',
      response.statusCode || 500,
      response.errorCode
    );
  }
}

// Error boundary component
export class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, errorInfo) {
    console.error('Error caught by boundary:', error, errorInfo);
    // Log to monitoring service
  }

  render() {
    if (this.state.hasError) {
      return (
        <div className="error-fallback">
          <h2>Something went wrong</h2>
          <details>{this.state.error?.message}</details>
          <button onClick={() => this.setState({ hasError: false, error: null })}>
            Try again
          </button>
        </div>
      );
    }

    return this.props.children;
  }
}
```

#### Retry Logic Implementation

```typescript
// lib/retryLogic.ts
export async function withRetry<T>(
  operation: () => Promise<T>,
  maxAttempts: number = 3,
  delay: number = 1000
): Promise<T> {
  let lastError: Error;

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await operation();
    } catch (error) {
      lastError = error as Error;
      
      if (attempt === maxAttempts) {
        throw lastError;
      }

      // Exponential backoff
      const waitTime = delay * Math.pow(2, attempt - 1);
      await new Promise(resolve => setTimeout(resolve, waitTime));
    }
  }

  throw lastError!;
}

// Usage example
const searchVehicles = async (query) => {
  return withRetry(async () => {
    const response = await http.post('/api/v1/availability/search', query);
    return handleAPIResponse(response.data);
  }, 3, 1000);
};
```

---

## Testing Guide

### 1. Unit Testing API Calls

```typescript
// __tests__/api/userService.test.ts
import { http } from '@/lib/httpService';
import { loginUser, registerUser } from '@/server/userOperations';

jest.mock('@/lib/httpService');

describe('User Service', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  test('should login user successfully', async () => {
    const mockResponse = {
      data: {
        success: true,
        user: { id: 1, email: 'test@example.com' }
      }
    };

    (http.post as jest.Mock).mockResolvedValue(mockResponse);

    const result = await loginUser('test@example.com', 'password');

    expect(http.post).toHaveBeenCalledWith(
      expect.stringContaining('/v1/user/checklogin'),
      { email: 'test@example.com', password: 'password' }
    );
    expect(result.success).toBe(true);
  });

  test('should handle login failure', async () => {
    const mockResponse = {
      data: {
        success: false,
        message: 'Invalid credentials'
      }
    };

    (http.post as jest.Mock).mockResolvedValue(mockResponse);

    await expect(loginUser('test@example.com', 'wrong')).rejects.toThrow('Invalid credentials');
  });
});
```

### 2. Integration Testing

```typescript
// __tests__/integration/bookingFlow.test.ts
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { BookingComponent } from '@/components/BookingComponent';

describe('Booking Flow Integration', () => {
  test('complete booking process', async () => {
    render(<BookingComponent vehicleId={123} />);

    // Select dates
    fireEvent.change(screen.getByLabelText('Start Date'), {
      target: { value: '2024-01-15T10:00' }
    });
    fireEvent.change(screen.getByLabelText('End Date'), {
      target: { value: '2024-01-17T10:00' }
    });

    // Click next step
    fireEvent.click(screen.getByText('Continue'));

    // Wait for pricing calculation
    await waitFor(() => {
      expect(screen.getByText(/Total: \$/)).toBeInTheDocument();
    });

    // Fill booking details
    fireEvent.change(screen.getByLabelText('Phone Number'), {
      target: { value: '+1234567890' }
    });

    // Complete booking
    fireEvent.click(screen.getByText('Complete Booking'));

    await waitFor(() => {
      expect(screen.getByText('Booking Confirmed')).toBeInTheDocument();
    });
  });
});
```

### 3. API Endpoint Testing

```typescript
// __tests__/api/endpoints.test.ts
import { createMocks } from 'node-mocks-http';
import handler from '@/pages/api/checkout/create-setup-intent';

describe('/api/checkout/create-setup-intent', () => {
  test('should create setup intent', async () => {
    const { req, res } = createMocks({
      method: 'POST',
      body: { customerId: 'cus_123' }
    });

    await handler(req, res);

    expect(res._getStatusCode()).toBe(200);
    const data = JSON.parse(res._getData());
    expect(data.clientSecret).toBeDefined();
  });

  test('should handle missing customer ID', async () => {
    const { req, res } = createMocks({
      method: 'POST',
      body: {}
    });

    await handler(req, res);

    expect(res._getStatusCode()).toBe(400);
  });
});
```

---

## Deployment & Configuration

### 1. Environment-Specific Configuration

#### Development Environment
```bash
# .env.development
NEXT_PUBLIC_APP_ENV=development
USER_MANAGEMENT_BASEURL=http://localhost:7443/api
BOOKING_SERVICES_BASEURL=http://localhost:7443/api
AVAILABILITY_BASEURL=http://localhost:6443/api
```

#### QA Environment
```bash
# .env.qa
NEXT_PUBLIC_APP_ENV=qa
USER_MANAGEMENT_BASEURL=https://qabundeeusermanagement.azurewebsites.net/api
BOOKING_SERVICES_BASEURL=https://qabundeebooking.azurewebsites.net/api
AVAILABILITY_BASEURL=https://qabundeeavailability.azurewebsites.net/api
```

#### Production Environment
```bash
# .env.production
NEXT_PUBLIC_APP_ENV=production
USER_MANAGEMENT_BASEURL=https://bundeeusermanagement.azurewebsites.net/api
BOOKING_SERVICES_BASEURL=https://bundeebooking.azurewebsites.net/api
AVAILABILITY_BASEURL=https://bundeeavailability.azurewebsites.net/api
```

### 2. Docker Configuration

```dockerfile
# Dockerfile
FROM node:18-alpine AS base

# Install dependencies
FROM base AS deps
WORKDIR /app
COPY package*.json pnpm-lock.yaml ./
RUN corepack enable pnpm && pnpm install --frozen-lockfile

# Build application
FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN corepack enable pnpm && pnpm run build

# Production image
FROM base AS runner
WORKDIR /app
ENV NODE_ENV production

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs

EXPOSE 3000
ENV PORT 3000

CMD ["node", "server.js"]
```

### 3. CI/CD Pipeline Configuration

#### GitHub Actions
```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'pnpm'
      
      - run: corepack enable pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm run test
      - run: pnpm run build

  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy to Azure
        uses: azure/webapps-deploy@v2
        with:
          app-name: 'bundee-driverweb'
          publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}
```

### 4. Monitoring and Observability

```typescript
// lib/monitoring.ts
import { trace, context, SpanStatusCode } from '@opentelemetry/api';

const tracer = trace.getTracer('mybundee-api');

export function instrumentAPI<T extends any[], R>(
  name: string,
  fn: (...args: T) => Promise<R>
) {
  return async (...args: T): Promise<R> => {
    const span = tracer.startSpan(name);
    
    try {
      const result = await fn(...args);
      span.setStatus({ code: SpanStatusCode.OK });
      return result;
    } catch (error) {
      span.setStatus({
        code: SpanStatusCode.ERROR,
        message: error instanceof Error ? error.message : 'Unknown error'
      });
      throw error;
    } finally {
      span.end();
    }
  };
}

// Usage
export const searchVehicles = instrumentAPI(
  'searchVehicles',
  async (query: SearchQuery) => {
    // API implementation
  }
);
```

---

## Security Best Practices

### 1. Input Validation

```typescript
// lib/validation.ts
import { z } from 'zod';

export const userRegistrationSchema = z.object({
  email: z.string().email(),
  firstName: z.string().min(1).max(50),
  lastName: z.string().min(1).max(50),
  password: z.string().min(8).regex(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/),
  phoneNumber: z.string().regex(/^\+?[1-9]\d{1,14}$/)
});

export const searchQuerySchema = z.object({
  zipCode: z.string().regex(/^\d{5}$/),
  startTime: z.string().datetime(),
  endTime: z.string().datetime(),
  latitude: z.number().min(-90).max(90).optional(),
  longitude: z.number().min(-180).max(180).optional()
});

export function validateInput<T>(schema: z.ZodSchema<T>, data: unknown): T {
  try {
    return schema.parse(data);
  } catch (error) {
    if (error instanceof z.ZodError) {
      throw new APIError('Invalid input data', 400, 'VALIDATION_ERROR');
    }
    throw error;
  }
}
```

### 2. Rate Limiting

```typescript
// lib/rateLimiting.ts
import { NextRequest } from 'next/server';

interface RateLimitConfig {
  maxRequests: number;
  windowMs: number;
}

const rateLimitMap = new Map<string, { count: number; resetTime: number }>();

export function rateLimit(config: RateLimitConfig) {
  return (req: NextRequest) => {
    const ip = req.ip || req.headers.get('x-forwarded-for') || 'unknown';
    const now = Date.now();
    
    const clientData = rateLimitMap.get(ip);
    
    if (!clientData || now > clientData.resetTime) {
      rateLimitMap.set(ip, {
        count: 1,
        resetTime: now + config.windowMs
      });
      return true;
    }
    
    if (clientData.count >= config.maxRequests) {
      return false;
    }
    
    clientData.count++;
    return true;
  };
}
```

---

This comprehensive API guide provides everything needed to integrate with the MyBundee platform, from basic setup to advanced features like real-time chat and payment processing. The documentation is based on the actual DriverWeb implementation and includes practical examples, error handling, testing strategies, and production deployment guidance.
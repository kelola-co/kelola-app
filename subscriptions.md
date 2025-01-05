
# Subscription Flow Documentation

## Overview

The subscription system operates across iOS and Android platforms using Revenue Cat as the subscription management platform, with a custom backend for business logic and state management.

### Core Components

1. **Client Applications (iOS/Android)**
   - Flutter SDK implementation
   - Revenue Cat SDK integration
   - Platform-specific purchase flows

2. **Revenue Cat**
   - Manages subscription offerings
   - Handles cross-platform purchases
   - Provides webhook notifications
   - Tracks subscription lifecycle

3. **Backend Service**
   - Stores subscription data
   - Processes webhook events
   - Serves as source of truth
   - Manages business relationships

## Integration Architecture

```mermaid
graph TD
    A[Mobile Apps] -->|Get Offerings| B[Revenue Cat SDK]
    A -->|Check Status| C[Backend API]
    B -->|Purchase Event| D[Revenue Cat Backend]
    D -->|Webhook| E[Backend Service]
    E -->|Store| F[Database]
    A -->|Query Status| E
```

## Subscription Data Flow

### Initial Load and Offering Display
```mermaid
sequenceDiagram
    participant App
    participant RevenueCat
    participant Backend
    participant DB

    App->>RevenueCat: Get All Offerings
    RevenueCat-->>App: Return Available Offerings
    App->>Backend: Get Owned Businesses Subscriptions
    Backend->>DB: Query Subscriptions
    DB-->>Backend: Return Status
    Backend-->>App: Return Active Subscriptions
    
    Note over App: Determine Offering Display Logic
    alt Current Business Has Active Subscription
        App->>App: Show Same Offering (For Upgrade/Downgrade)
    else No Active Subscription
        App->>App: Show Available Offering
    end
```

### Purchase Flow
```mermaid
sequenceDiagram
    participant User
    participant App
    participant RevenueCat
    participant Backend
    participant DB

    User->>App: Initiate Purchase
    App->>RevenueCat: Process Purchase
    RevenueCat-->>App: Purchase Success
    RevenueCat->>Backend: Webhook (INITIAL_PURCHASE)
    Backend->>DB: Store Subscription Data
    Backend-->>RevenueCat: Acknowledge Webhook
    App->>Backend: Get Updated Status
    Backend-->>App: Return New Status
```

## Webhook Event Processing

### Event Types and Handling
```mermaid
graph TD
    A[Webhook Event] --> B{Event Type}
    B -->|INITIAL_PURCHASE| C[Create New Subscription]
    B -->|RENEWAL| D[Update Subscription Period]
    B -->|CANCELLATION| E[Mark as Cancelled]
    B -->|PRODUCT_CHANGE| F[Update Product Details]
    B -->|EXPIRATION| G[Mark as Expired]
    
    C --> H[Update DB]
    D --> H
    E --> H
    F --> H
    G --> H
```

## Detailed Process Flows

### 1. Offering Management

#### Client-Side Flow
```mermaid
graph TD
    A[App Start] --> B[Fetch Revenue Cat Offerings]
    B --> C[Get Backend Subscription Status]
    C --> D{Check Business Status}
    D -->|Active| E[Show Current Tier Options]
    D -->|Inactive| F[Show Available Offerings]
    F --> G{Check Subscription Limits}
    G -->|Within Limits| H[Display Purchase Options]
    G -->|Exceeded| I[Show Admin Contact]
```

### 2. Purchase Processing Flow
```mermaid
sequenceDiagram
    participant User
    participant App
    participant RevenueCat
    participant Backend

    User->>App: Select Subscription
    App->>RevenueCat: Initialize Purchase
    RevenueCat-->>App: Present Payment UI
    User->>App: Confirm Payment
    App->>RevenueCat: Process Payment
    RevenueCat-->>App: Confirm Success
    RevenueCat->>Backend: Send Webhook
    Backend->>Backend: Process Event
    App->>Backend: Get Updated Status
    Backend-->>App: Return New Status
```

## Product ID Structure

### Format
`kelola-{tier}-{period}-{offering_number}-{platform-code}`

### Revenue Cat Configuration
- Mapped to platform-specific store products
- Consistent across platforms
- Managed through Revenue Cat dashboard

## Database Structure

### Tables Relationship
```mermaid
erDiagram
    Business ||--o{ Subscription : has
    Business {
        string id
        string subscription_id FK
        string name
        timestamp created_at
        timestamp updated_at
    }
    Subscription {
        string id
        string business_id FK
        string status
        string product_id
        timestamp event_time
        timestamp created_at
    }
```

## Webhook Event Processing

### 1. INITIAL_PURCHASE
- Create new record in subscriptions table
- Update business.subscription_id with new subscription
- Set subscription status as active

### 2. RENEWAL
- Create new record in subscriptions table
- Update business.subscription_id with new subscription
- Maintain subscription status as active

### 3. PRODUCT_CHANGE
- Create new record in subscriptions table
- Platform-specific behavior:
  - iOS: Update business.subscription_id
  - Android: Keep existing business.subscription_id

### 4. CANCELLATION
- Create new record in subscriptions table
- Keep existing business.subscription_id
- Update subscription status

### 5. EXPIRATION
- Create new record in subscriptions table
- Remove subscription_id from business table
- Set subscription status as expired

### Edge Case: Delayed Expiration Events
```mermaid
sequenceDiagram
    participant RC as RevenueCat
    participant BE as Backend
    participant DB as Database

    Note over RC,DB: Subscription Expires
    Note over RC,DB: User Makes New Purchase
    RC->>BE: INITIAL_PURCHASE Event
    BE->>DB: Create New Subscription
    BE->>DB: Update Business.subscription_id
    Note over RC,DB: Delayed Expiration Arrives
    RC->>BE: EXPIRATION Event (Delayed)
    BE->>BE: Check if expiration_time > initial_purchase_time + N days
    alt Invalid Expiration
        BE->>BE: Ignore expiration event
    else Valid Expiration
        BE->>DB: Process normal expiration flow
    end
```

#### Expiration Validation Logic
- Compare expiration event timestamp with latest initial purchase
- If expiration occurs N days after new purchase:
  - Consider expiration event invalid
  - Maintain current subscription status
  - Log incident for monitoring

## Backend API Integration

### Primary Endpoints
1. **Subscription Status**
   - GET /api/business/{businessId}/subscription
   - Returns current subscription state

2. **Business Subscriptions**
   - GET /api/user/businesses/subscriptions
   - Returns all owned business subscriptions

3. **Webhook Handler**
   - POST /api/webhooks/revenuecat
   - Processes Revenue Cat events

## State Management

### Subscription States
```mermaid
stateDiagram-v2
    [*] --> Active: INITIAL_PURCHASE
    Active --> Active: RENEWAL
    Active --> Changed: PRODUCT_CHANGE
    Changed --> Active: Process Complete
    Active --> Cancelled: CANCELLATION
    Active --> Expired: EXPIRATION
    Cancelled --> [*]
    Expired --> Active: New Purchase
```

## Error Handling

### Common Scenarios
1. **Webhook Processing Errors**
   - Retry mechanism
   - Error logging
   - Alert system

2. **Purchase Failures**
   - Revenue Cat error handling
   - User feedback
   - Status consistency checks

3. **State Synchronization**
   - Backend/Revenue Cat reconciliation
   - Data consistency checks
   - Recovery procedures

## Platform-Specific Considerations

### Android
1. **Play Store Integration**
   - Revenue Cat product mapping
   - Purchase flow handling
   - Up to 5 business subscriptions

### iOS
1. **App Store Integration**
   - Revenue Cat product mapping
   - Purchase flow handling
   - Up to 2 business subscriptions

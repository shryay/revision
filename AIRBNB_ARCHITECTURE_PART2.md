# 🏨 AirBnb Backend — LLD & Core Flows (Part 2)

---

## 4. Low-Level Design (LLD) — Core Flows

### 4.1 Booking Lifecycle (State Machine)

```mermaid
stateDiagram-v2
    [*] --> RESERVED : POST /bookings/init<br/>Inventory locked
    RESERVED --> GUESTS_ADDED : POST /bookings/{id}/addGuests
    GUESTS_ADDED --> PAYMENTS_PENDING : POST /bookings/{id}/payments<br/>Stripe session created
    PAYMENTS_PENDING --> CONFIRMED : Webhook checkout.session.completed<br/>reservedCount→bookCount
    RESERVED --> EXPIRED : 10 min timeout
    GUESTS_ADDED --> EXPIRED : 10 min timeout
    CONFIRMED --> CANCELLED : POST /bookings/{id}/cancel<br/>Stripe refund issued
```

**Key Rules:**
- Booking expires if not paid within **10 minutes** of creation ([hasBookingExpired()](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Service/BookingServiceImpl.java#297-300))
- Only **CONFIRMED** bookings can be cancelled
- Cancellation triggers a **Stripe refund** via `RefundCreateParams`

### 4.2 Complete Booking Flow (Sequence Diagram)

```mermaid
sequenceDiagram
    actor User
    participant BC as BookingController
    participant BS as BookingServiceImpl
    participant IR as InventoryRepository
    participant PS as PricingService
    participant BR as BookingRepository
    participant CS as CheckoutServiceImpl
    participant Stripe

    Note over User,Stripe: Step 1: Initialize Booking
    User->>BC: POST /bookings/init (hotelId, roomId, dates, roomsCount)
    BC->>BS: initialiseBooking(request)
    BS->>IR: findAndLockAvailableInventory() [PESSIMISTIC_WRITE]
    IR-->>BS: List of InventoryEntity (locked rows)
    BS->>BS: Validate inventoryList.size() == daysCount
    BS->>IR: initBooking() → reservedCount += roomsCount
    BS->>PS: calculateTotalPrice(inventoryList)
    PS-->>BS: totalPrice (with dynamic pricing)
    BS->>BR: save(booking) [status=RESERVED]
    BR-->>User: BookingDTO

    Note over User,Stripe: Step 2: Add Guests
    User->>BC: POST /bookings/{id}/addGuests
    BC->>BS: addGuests(bookingId, guestList)
    BS->>BS: Validate ownership + not expired + status==RESERVED
    BS->>BR: save(booking) [status=GUESTS_ADDED]
    BR-->>User: BookingDTO

    Note over User,Stripe: Step 3: Initiate Payment
    User->>BC: POST /bookings/{id}/payments
    BC->>BS: initiatePayments(bookingId)
    BS->>BS: Validate ownership + not expired
    BS->>CS: getCheckoutSession(booking, successUrl, failureUrl)
    CS->>Stripe: Create Customer + Session
    Stripe-->>CS: Session URL + Session ID
    CS->>BR: save(booking.paymentSessionId)
    CS-->>BS: sessionUrl
    BS->>BR: save(booking) [status=PAYMENTS_PENDING]
    BS-->>User: sessionUrl → redirect to Stripe

    Note over User,Stripe: Step 4: Payment Confirmation via Webhook
    Stripe->>BC: POST /webhook/payment (payload + Stripe-Signature)
    BC->>BS: capturePayment(event)
    BS->>BS: Verify event.type == "checkout.session.completed"
    BS->>BR: findByPaymentSessionId(sessionId)
    BS->>BR: save(booking) [status=CONFIRMED]
    BS->>IR: findAndLockReservedInventory() [PESSIMISTIC_WRITE]
    BS->>IR: confirmBooking() → reservedCount -= N, bookCount += N
```

### 4.3 Authentication Flow

```mermaid
sequenceDiagram
    actor User
    participant AC as AuthController
    participant AS as AuthService
    participant JWT as JWTService
    participant UR as UserRepository

    Note over User,UR: Signup
    User->>AC: POST /auth/signup {email, password, name}
    AC->>AS: signUp(dto)
    AS->>UR: findByEmail (duplicate check)
    AS->>AS: Encode password (BCrypt)
    AS->>AS: Set role = GUEST
    AS->>UR: save(user)
    AS-->>User: UserDTO

    Note over User,UR: Login
    User->>AC: POST /auth/login {email, password}
    AC->>AS: login(dto)
    AS->>AS: AuthenticationManager.authenticate()
    AS->>JWT: generateAccessToken(user) [10 min TTL]
    AS->>JWT: generateRefreshToken(user) [6 month TTL]
    AS-->>AC: [accessToken, refreshToken]
    AC->>AC: Set refreshToken as HttpOnly Cookie
    AC-->>User: {accessToken}

    Note over User,UR: JWT Filter (every request)
    User->>AC: GET /any-endpoint [Authorization: Bearer {token}]
    AC->>JWT: getUserIdFromToken(token)
    JWT-->>AC: userId
    AC->>UR: findById(userId)
    AC->>AC: Set SecurityContext
```

**JWT Details:**
- **Access Token**: 10 min expiry, claims = {sub: userId, email, roles}
- **Refresh Token**: 6 months expiry, claims = {sub: userId}
- **Secret Key**: HMAC-SHA from env `JWT_SECRET`
- Refresh token stored as **HttpOnly cookie** (not accessible by JS → XSS protection)

### 4.4 Dynamic Pricing Engine (Decorator Pattern)

```mermaid
graph TD
    subgraph "Decorator Chain (applied in order)"
        A["BasePricingStrategy<br/>→ room.basePrice"]
        B["SurgePricingStrategy<br/>→ price × inventory.surgeFactor"]
        C["OccupancyPricingStrategy<br/>→ if occupancy > 80%: price × 1.2"]
        D["UrgencyPricingStrategy<br/>→ if date within 7 days: price × 1.15"]
        E["HolidayPricingStrategy<br/>→ if holiday: price × 1.25"]
    end

    A --> B --> C --> D --> E
    E -->|"Final Price"| F["PricingService.calculateDynamicPricing()"]
```

**How it works in code:**
```java
PricingStrategy strategy = new BasePricingStrategy();         // base
strategy = new SurgePricingStrategy(strategy);                // wrap 1
strategy = new OccupancyPricingStrategy(strategy);            // wrap 2
strategy = new UrgencyPricingStrategy(strategy);              // wrap 3
strategy = new HolidayPricingStrategy(strategy);              // wrap 4
return strategy.calculatePrice(inventory);                     // cascades
```

**Example Calculation (worst case all multipliers apply):**
- Base Price: ₹1000
- × Surge Factor (1.5): ₹1500
- × Occupancy (1.2): ₹1800
- × Urgency (1.15): ₹2070
- × Holiday (1.25): **₹2587.50**

### 4.5 Inventory Concurrency Control

```mermaid
sequenceDiagram
    participant User1
    participant User2
    participant IR as InventoryRepository
    participant DB as PostgreSQL

    Note over User1,DB: Concurrent Booking Scenario
    User1->>IR: findAndLockAvailableInventory(roomId, dates)
    IR->>DB: SELECT ... FOR UPDATE (PESSIMISTIC_WRITE)
    DB-->>IR: Rows LOCKED ✅
    
    User2->>IR: findAndLockAvailableInventory(roomId, dates)
    IR->>DB: SELECT ... FOR UPDATE
    DB-->>User2: ⏳ BLOCKED (waiting for lock)
    
    User1->>IR: initBooking() → reservedCount += 1
    IR->>DB: UPDATE ... SET reservedCount = reservedCount + 1
    DB-->>IR: Done, LOCK RELEASED
    
    DB-->>User2: Lock acquired ✅
    IR-->>User2: Check availability (totalCount - bookCount - reservedCount)
    Note over User2: If no rooms left → "Room not available anymore"
```

**Three-Level Inventory Counting:**

| Field | Meaning |
|---|---|
| `totalCount` | Total rooms of this type available on this date |
| `reservedCount` | Rooms reserved but not yet paid (temporary hold) |
| `bookCount` | Rooms confirmed and paid |
| **Available** | `totalCount - bookCount - reservedCount` |

**Inventory Lifecycle:**
1. **initBooking**: `reservedCount += N` (temporary hold)
2. **confirmBooking**: `reservedCount -= N`, `bookCount += N` (payment confirmed)
3. **cancelBooking**: `bookCount -= N` (refund issued)

### 4.6 Scheduled Pricing Update (Cron Job)

```mermaid
graph TD
    A["PricingUpdateService<br/>@Scheduled cron = 0 0 * * * *<br/>(Every Hour)"]
    B["Fetch all hotels<br/>(paginated, batch=100)"]
    C["For each hotel: fetch inventory<br/>(today → today+1year)"]
    D["Apply PricingService.calculateDynamicPricing()<br/>on each inventory row"]
    E["Bulk save updated inventory prices"]
    F["Compute min price per date per hotel"]
    G["Upsert into HotelMinPrice table"]

    A --> B --> C --> D --> E --> F --> G
```

**Why HotelMinPrice table?** — Search optimization. Instead of scanning all inventory rows, the search query hits the pre-computed [HotelMinPrice](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Entities/HotelMinPriceEntity.java#14-45) table to show "starting from ₹X" prices.

### 4.7 Hotel Search Flow

```mermaid
sequenceDiagram
    actor User
    participant HBC as HotelBrowseController
    participant IS as InventoryServiceImpl
    participant HMPR as HotelMinPriceRepository

    User->>HBC: GET /hotels/search {city, startDate, endDate, roomsCount, page, size}
    HBC->>IS: searchHotels(request)
    IS->>HMPR: findHotelsWithAvailableInventory()
    Note over HMPR: SELECT hotel, AVG(price)<br/>FROM HotelMinPrice<br/>WHERE city=:city AND date BETWEEN<br/>AND hotel.active=true<br/>GROUP BY hotel
    HMPR-->>IS: Page of HotelPriceDTO
    IS-->>User: Paginated results with avg min price
```

### 4.8 Global Response & Exception Handling

**All responses** are wrapped in a standard `ApiResponse` envelope:
```json
{
  "data": { ... },        // on success
  "error": {              // on failure
    "status": "NOT_FOUND",
    "message": "Hotel not found with ID: 5"
  }
}
```

**Exception Mapping:**

| Exception | HTTP Status |
|---|---|
| `ResourceNotFoundException` | 404 |
| [AuthenticationException](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Advice/GlobalExceptionHandler.java#24-32) | 401 |
| [JwtException](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Advice/GlobalExceptionHandler.java#33-41) | 401 |
| [AccessDeniedException](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Advice/GlobalExceptionHandler.java#42-50) | 403 |
| [Exception](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Advice/GlobalExceptionHandler.java#33-41) (catch-all) | 500 |

The [GlobalResponseHandler](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Advice/GlobalResponseHandler.java#13-36) (implements `ResponseBodyAdvice`) wraps ALL controller responses in `ApiResponse` automatically, except Swagger/Actuator endpoints.

### 4.9 Hotel Activation Flow

```mermaid
sequenceDiagram
    actor Admin
    participant HC as HotelController
    participant HS as HotelServiceImpl
    participant IS as InventoryServiceImpl
    participant IR as InventoryRepository

    Admin->>HC: PATCH /admin/hotels/{id}/activate
    HC->>HS: activateHotel(hotelId)
    HS->>HS: Validate ownership
    HS->>HS: hotel.setActive(true)
    loop For each room in hotel
        HS->>IS: initializeRoomForAYear(room)
        IS->>IR: Save 365 InventoryEntity rows<br/>(one per day, basePrice, surgeFactor=1, closed=false)
    end
```

**Key insight:** Inventory is pre-created for 365 days when a hotel is activated. Each row represents one room type on one date.

# 🏨 AirBnb Backend — Interview Cross-Questions & Answers (Part 3)

---

## 5. Interview Cross-Questions & Answers

### Category 1: Architecture & Design Decisions

---

**Q1: Walk me through the architecture of your project in 2 minutes.**

**A:** It's a monolithic Spring Boot 3 backend for a hotel booking platform. The architecture follows a classic **3-tier layered pattern**: Controllers → Services → Repositories, backed by PostgreSQL via JPA/Hibernate. Authentication uses stateless JWT with access + refresh tokens. Payments are handled via Stripe's Checkout Session API with webhook-based confirmation. I implemented dynamic pricing using the **Decorator design pattern** with 4 pricing strategies layered on top of a base price. For concurrency during bookings, I use **pessimistic locking** (`SELECT ... FOR UPDATE`) at the database level to prevent overbooking. A scheduled cron job recalculates prices hourly and maintains a [HotelMinPrice](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Entities/HotelMinPriceEntity.java#14-45) materialized summary table for fast search queries. The whole thing is Dockerized using a multi-stage-like Dockerfile with OpenJDK 23 slim.

---

**Q2: Why did you choose a monolithic architecture instead of microservices?**

**A:** For the scope and scale of this project, a monolith was the right call. Microservices add operational complexity—service discovery, inter-service communication, distributed transactions, deployment pipelines per service. Since this is a single-team project, a well-structured monolith with clear package boundaries (Controller/Service/Repository) gives the same separation of concerns without the overhead. That said, the package structure is modular enough that if we needed to extract, say, the Booking service or Pricing engine into a separate microservice, the interfaces are already defined — like [BookingService](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Service/BookingServiceImpl.java#40-302), [InventoryService](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Service/InventoryServiceImpl.java#31-118), etc.

---

**Q3: Explain the package structure and why you organized it this way.**

**A:** I used a **layer-based** package structure:
- [Controller](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Controller/UserController.java#20-77) — REST endpoints, thin layer, delegates to services
- [Service](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Security/JWTService.java#14-54) + `Service.Interfaces` — Business logic with interface segregation
- `Repositories` — Data access using Spring Data JPA
- `Entities` — JPA entities mapping to DB tables
- `Dto` — Data Transfer Objects to decouple API contracts from entities
- [Security](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Security/WebSecurityConfig.java#20-67) — Auth, JWT, filter chain
- [Strategy](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Strategy/PricingStrategy.java#6-10) — Pricing strategies (Decorator pattern)
- `Advice` — Global exception & response handling
- [Config](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Security/WebSecurityConfig.java#20-67) — Bean configurations
- `Enums` — BookingStatus, Role, Gender
- `Exceptions` — Custom exceptions
- `Utils` — Utility methods like `getCurrentUser()`

The interface segregation in Services (e.g., [BookingService](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Service/BookingServiceImpl.java#40-302) interface, [BookingServiceImpl](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Service/BookingServiceImpl.java#40-302) implementation) follows the **Dependency Inversion Principle** — controllers depend on abstractions, not concrete classes.

---

**Q4: Why do you use DTOs instead of directly exposing entities?**

**A:** Three reasons:
1. **Security** — Entities like [UserEntity](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Entities/UserEntity.java#19-69) contain [password](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Security/WebSecurityConfig.java#49-53). Exposing entities would leak sensitive data.
2. **Decoupling** — If the DB schema changes, the API contract doesn't break. DTOs act as a contract layer.
3. **Control** — I can shape the response differently. For example, `HotelInfoDto` combines `HotelDTO` + `List<RoomDTO>`, which doesn't map 1:1 to any entity.

I use **ModelMapper** for entity↔DTO conversion to avoid boilerplate.

---

### Category 2: Design Patterns

---

**Q5: You mentioned the Decorator pattern for pricing. Explain it in detail.**

**A:** The dynamic pricing system uses the **Decorator (Wrapper) pattern**. 

- **Interface**: [PricingStrategy](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Strategy/PricingStrategy.java#6-10) with a single method [calculatePrice(InventoryEntity)](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Strategy/SurgePricingStrategy.java#13-18)
- **Base**: [BasePricingStrategy](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Strategy/BasePricingStrategy.java#7-13) returns `room.getBasePrice()`
- **Decorators**: [SurgePricingStrategy](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Strategy/SurgePricingStrategy.java#8-19), [OccupancyPricingStrategy](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Strategy/OccupancyPricingStrategy.java#8-23), [UrgencyPricingStrategy](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Strategy/UrgencyPricingStrategy.java#9-25), [HolidayPricingStrategy](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Strategy/HolidayPricingStrategy.java#8-23) — each wraps another [PricingStrategy](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Strategy/PricingStrategy.java#6-10) and multiplies the price

In [PricingService](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Strategy/PricingService.java#9-31), they're chained:
```
Base → Surge → Occupancy → Urgency → Holiday
```
Each decorator calls `wrapped.calculatePrice()` first, then applies its multiplier. This is the classic Decorator pattern because:
- Each strategy implements the same interface
- Each wraps another strategy via constructor injection
- They can be composed in any order
- Adding a new strategy requires zero changes to existing code (**Open/Closed Principle**)

---

**Q6: Could you have used the Strategy pattern instead? What's the difference?**

**A:** Strategy pattern selects ONE algorithm at runtime (if-else replacement). Decorator **composes multiple behaviors** that stack. Here, I need ALL pricing factors to apply simultaneously — surge AND occupancy AND urgency AND holiday — not one OR the other. Strategy would require a single god-class that knows about all factors. Decorator keeps each factor isolated and composable. If the interviewer asks "what if only some strategies apply?" — we can conditionally skip wrapping.

---

**Q7: What other design patterns have you used in this project?**

**A:**
- **Builder Pattern** — [BookingEntity](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Entities/BookingEntity.java#14-71) and [InventoryEntity](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Entities/InventoryEntity.java#12-65) use Lombok's `@Builder` for readable object construction
- **Repository Pattern** — Spring Data JPA repositories abstract data access
- **DTO Pattern** — Decouple API layer from persistence layer
- **Template Method (implicit)** — Spring's `OncePerRequestFilter` in [JWTAuthFilter](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Security/JWTAuthFilter.java#22-62)
- **Singleton** — All Spring `@Service`, `@Configuration`, `@Repository` beans are singletons by default

---

### Category 3: Concurrency & Database

---

**Q8: How do you prevent two users from booking the same room at the same time?**

**A:** I use **pessimistic locking** with `@Lock(LockModeType.PESSIMISTIC_WRITE)` on the [findAndLockAvailableInventory()](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Repositories/InventoryRepository.java#46-61) query. This translates to `SELECT ... FOR UPDATE` in PostgreSQL, which **row-level locks** the inventory rows. If User A locks the rows first, User B's query blocks until User A's transaction commits. After User A's transaction completes (reservedCount is incremented), User B's query runs and checks if rooms are still available. If `totalCount - bookCount - reservedCount < requestedRooms`, it throws `"Room is not available anymore"`.

---

**Q9: Why pessimistic locking over optimistic locking?**

**A:** In a booking system, **contention is expected** — multiple users may try to book the same room on the same date. 

- **Optimistic locking** (version column) would detect conflicts at commit time and throw an exception, requiring retry logic. With high contention, many retries mean poor UX.
- **Pessimistic locking** blocks concurrent writes upfront. The first user gets through, the second waits briefly, then proceeds or fails fast. For a booking system where data consistency is critical and the locked window is small (just the booking transaction), pessimistic is the better choice.

If this were a read-heavy system with rare writes (like a wiki), optimistic would be better.

---

**Q10: Explain the three-level inventory counting system.**

**A:** Each [InventoryEntity](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Entities/InventoryEntity.java#12-65) row represents (hotel, room_type, date) and has:
- `totalCount` — physical rooms available (set when room is created)
- `reservedCount` — temporarily held during booking (before payment)
- `bookCount` — confirmed and paid

Available rooms = `totalCount - bookCount - reservedCount`

State transitions:
1. **Init booking**: `reservedCount += N` (hold)
2. **Confirm payment**: `reservedCount -= N`, `bookCount += N` (commit)
3. **Cancel**: `bookCount -= N` (release)

This prevents overbooking even when payments are pending.

---

**Q11: What's the unique constraint on the Inventory table and why?**

**A:** `@UniqueConstraint(columnNames = {"hotel_id", "room_id", "date"})` — ensures only ONE inventory record exists per hotel + room type + date. Without this, duplicate rows could cause incorrect availability counts.

---

**Q12: What happens if a user reserves but never pays?**

**A:** The [hasBookingExpired()](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Service/BookingServiceImpl.java#297-300) method checks if the booking was created more than 10 minutes ago. If expired:
- Adding guests fails with `"Booking has already expired"`
- Payment initiation fails with the same error
- The `reservedCount` stays incremented (potential leak)

**Improvement I'd make:** Add a scheduled job to clean up expired reservations — find bookings where `status = RESERVED` and `createdAt < now - 10min`, then decrement `reservedCount` and set status to `EXPIRED`.

---

### Category 4: Security

---

**Q13: Explain your JWT authentication flow end-to-end.**

**A:**
1. **Signup**: Password hashed with BCrypt, user saved with role `GUEST`
2. **Login**: `AuthenticationManager` validates credentials. On success, generate access token (10 min) and refresh token (6 months). Access token returned in response body. Refresh token set as **HttpOnly cookie**.
3. **Every Request**: [JWTAuthFilter](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Security/JWTAuthFilter.java#22-62) (extends `OncePerRequestFilter`) extracts Bearer token from `Authorization` header, parses userId from JWT claims, loads [UserEntity](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Entities/UserEntity.java#19-69), sets `SecurityContext`.
4. **Token Refresh**: Client sends refresh token via cookie to `POST /auth/refresh`, server extracts userId, generates new access token.

---

**Q14: Why is the refresh token stored as an HttpOnly cookie?**

**A:** HttpOnly cookies **cannot be accessed by JavaScript** (`document.cookie` doesn't see them). This protects against **XSS attacks** — even if an attacker injects malicious JS, they can't steal the refresh token. The access token in response body has a short 10-min life, limiting damage if stolen.

---

**Q15: How does role-based access control work?**

**A:** In [WebSecurityConfig](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Security/WebSecurityConfig.java#20-67):
```java
.requestMatchers("/admin/**").hasRole("HOTEL_MANAGER")
.requestMatchers("/bookings/**").authenticated()
.requestMatchers("/users/**").authenticated()
.anyRequest().permitAll()
```
Roles are stored as `Set<Role>` in `app_user_roles` table (via `@ElementCollection`). The `UserEntity.getAuthorities()` converts roles to `SimpleGrantedAuthority("ROLE_HOTEL_MANAGER")`. Spring Security checks these during request authorization.

---

**Q16: How are you handling CSRF and why did you disable it?**

**A:** CSRF is disabled because the API is **stateless** (no sessions/cookies for auth). CSRF attacks exploit session-based auth where the browser auto-sends cookies. Since we use JWT in the `Authorization` header (not auto-sent by browsers), CSRF protection is unnecessary. The session policy is `STATELESS`.

---

### Category 5: Payments & Webhooks

---

**Q17: Walk me through the Stripe payment integration.**

**A:**
1. User calls `POST /bookings/{id}/payments`
2. [CheckoutServiceImpl](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Service/CheckoutServiceImpl.java#19-77) creates a Stripe `Customer` with user's name/email
3. Creates a Stripe [Session](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Service/CheckoutServiceImpl.java#26-76) with `mode=PAYMENT`, product name = hotel + room type, amount in paisa (INR × 100), success/failure URLs pointing to frontend
4. Saves `session.getId()` to `booking.paymentSessionId`
5. Returns `session.getUrl()` — user is redirected to Stripe's hosted checkout page
6. After payment, Stripe sends `POST /webhook/payment` with event `checkout.session.completed`
7. [WebhookController](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Controller/WebhookController.java#14-37) verifies the signature using `Webhook.constructEvent()` with the webhook secret
8. [capturePayment()](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Service/BookingServiceImpl.java#161-187) finds the booking by session ID, sets status to `CONFIRMED`, and confirms inventory

---

**Q18: Why use webhooks instead of polling for payment status?**

**A:** Webhooks are **event-driven** — Stripe pushes the event to us immediately when payment completes. Polling would waste resources checking repeatedly, could miss events between poll intervals, and doesn't scale. Webhooks also handle edge cases like delayed payments or failed charges that we get notified about automatically.

---

**Q19: How do you verify the webhook is actually from Stripe?**

**A:** Every webhook request includes a `Stripe-Signature` header. I use `Webhook.constructEvent(payload, sigHeader, endpointSecret)` which:
1. Computes HMAC-SHA256 of the payload using our `stripe.webhook.secret`
2. Compares it with the signature in the header
3. Throws `SignatureVerificationException` if they don't match

This prevents attackers from faking webhook calls.

---

**Q20: How does the refund work on cancellation?**

**A:** In [cancelBooking()](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Repositories/InventoryRepository.java#94-107):
1. Retrieve the Stripe [Session](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Service/CheckoutServiceImpl.java#26-76) using `booking.paymentSessionId`
2. Get the `paymentIntent` from the session
3. Create a `Refund` using `RefundCreateParams.builder().setPaymentIntent(paymentIntent).build()`
4. Call `Refund.create(refundParams)`

Stripe processes the full refund. The inventory is also released (`bookCount -= N`).

---

### Category 6: Spring Boot Internals

---

**Q21: What does `@Transactional` do and where have you used it?**

**A:** `@Transactional` wraps the method in a database transaction. If any exception occurs, the entire transaction rolls back. I've used it in:
- [initialiseBooking()](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Service/BookingServiceImpl.java#57-102) — lock + reserve must be atomic
- [capturePayment()](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Service/BookingServiceImpl.java#161-187) — lock + confirm must be atomic
- [cancelBooking()](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Repositories/InventoryRepository.java#94-107) — cancel + release inventory must be atomic
- [activateHotel()](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Controller/HotelController.java#60-66) — creating 365 inventory rows per room
- [deleteHotelById()](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Service/HotelServiceImpl.java#85-103) — deleting all rooms + inventories + hotel
- [updateInventory()](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Controller/InventoryController.java#31-39) — lock + update must be atomic

Without `@Transactional`, if the app crashes between locking inventory and saving the booking, we'd have orphaned locks.

---

**Q22: What is `@RestControllerAdvice` and how do you use it?**

**A:** It's a combination of `@ControllerAdvice` + `@ResponseBody`. I use it in two places:
1. **[GlobalExceptionHandler](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Advice/GlobalExceptionHandler.java#12-65)** — catches exceptions globally and returns structured error responses with appropriate HTTP status codes
2. **[GlobalResponseHandler](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Advice/GlobalResponseHandler.java#13-36)** — implements `ResponseBodyAdvice` to automatically wrap ALL controller responses in an `ApiResponse` envelope, providing a consistent API contract

---

**Q23: Explain the `OncePerRequestFilter` and why your JWT filter extends it.**

**A:** `OncePerRequestFilter` guarantees the filter runs exactly once per request (even with forwards/includes). My [JWTAuthFilter](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Security/JWTAuthFilter.java#22-62):
1. Checks for `Authorization: Bearer {token}` header
2. If absent, passes through (for public endpoints)
3. If present, extracts token, gets userId from JWT, loads UserEntity, sets SecurityContext
4. Handles [JwtException](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Advice/GlobalExceptionHandler.java#33-41) by delegating to `HandlerExceptionResolver` for consistent error formatting

---

**Q24: What is `SecurityContextHolder` and how do you use it?**

**A:** It stores the authenticated user's info (principal, authorities) for the current thread. After JWT validation, I set it:
```java
SecurityContextHolder.getContext().setAuthentication(authToken);
```
Then anywhere in the service layer, I retrieve the current user:
```java
UserEntity user = (UserEntity) SecurityContextHolder
    .getContext().getAuthentication().getPrincipal();
```
This is wrapped in a utility: `AppUtils.getCurrentUser()`.

---

### Category 7: System Design & Scalability

---

**Q25: How would you scale this system for millions of users?**

**A:** Several improvements:
1. **Microservices** — Split into Hotel Service, Booking Service, User Service, Payment Service, Search Service
2. **Caching** — Redis for hotel search results, user sessions
3. **Message Queue** — RabbitMQ/Kafka for async booking confirmations and notifications
4. **Search** — Elasticsearch for hotel search instead of DB queries
5. **CDN** — For hotel/room images
6. **Read replicas** — PostgreSQL read replicas for search queries
7. **Rate limiting** — To prevent abuse
8. **Connection pooling** — HikariCP (already default in Spring Boot)

---

**Q26: What would you change about the inventory system at scale?**

**A:**
1. **Replace pessimistic locking with optimistic locking + retry** for better throughput at scale
2. **Partition inventory table** by date — old dates can be archived
3. **Use Redis for real-time availability** — maintain counts in Redis, sync to DB asynchronously
4. **Pre-aggregate availability** rather than computing `total - booked - reserved` every time

---

**Q27: The PricingUpdateService runs every hour for ALL hotels. How would you optimize?**

**A:**
1. **Event-driven updates** — Only recalculate when surge_factor or bookings change
2. **Batch processing** — Use `saveAll()` instead of individual saves (already doing this)
3. **Parallel processing** — Use `@Async` or virtual threads for concurrent hotel processing
4. **Incremental updates** — Only process inventory rows where input factors changed
5. **Separate worker** — Move to a dedicated background worker/service

---

**Q28: What's the HotelMinPrice table for and why did you create it?**

**A:** It's a **pre-computed materialized view** for search optimization. When a user searches hotels in a city for certain dates, instead of scanning the entire Inventory table (which has 365 × rooms × hotels rows), we query HotelMinPrice which has the minimum price per hotel per date. The search returns `AVG(minPrice)` grouped by hotel. The cron job keeps this table updated hourly.

---

### Category 8: Error Handling & Edge Cases

---

**Q29: What happens if the Stripe webhook fails or is called twice?**

**A:** 
- **Idempotency**: The current code doesn't handle duplicate webhooks. Fix: check if `booking.status` is already `CONFIRMED` before processing.
- **Failure**: If our server is down, Stripe retries webhooks with exponential backoff for up to 3 days. We should make the handler idempotent.

---

**Q30: What if the app crashes after reserving inventory but before saving the booking?**

**A:** Because [initialiseBooking()](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Service/BookingServiceImpl.java#57-102) is `@Transactional`, if the app crashes mid-transaction, the DB automatically rolls back — the `reservedCount` increment is undone. This is the whole point of database transactions and ACID properties.

---

**Q31: How do you handle situations where a user tries to book a hotel they own?**

**A:** Currently, there's no check for this. A hotel manager could technically book their own hotel. Fix: add a check in [initialiseBooking()](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Service/BookingServiceImpl.java#57-102):
```java
if (getCurrentUser().equals(hotel.getOwner())) {
    throw new IllegalStateException("Cannot book your own hotel");
}
```

---

### Category 9: Database & JPA

---

**Q32: Explain the `@Embedded` and `@Embeddable` annotations you used.**

**A:** [HotelContactInfo](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Entities/HotelContactInfo.java#7-16) is `@Embeddable` — it's not a separate table, its fields (address, phoneNumber, email, location) are embedded directly into the Hotel table as columns. [HotelEntity](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Entities/HotelEntity.java#15-55) uses `@Embedded` to include it. This is for value objects that don't need their own identity/table.

---

**Q33: Why `FetchType.LAZY` on relationships?**

**A:** LAZY loading means related entities are loaded only when accessed, not when the parent is loaded. For example, loading a hotel doesn't load all its rooms until `hotel.getRooms()` is called. This prevents the **N+1 query problem** and reduces memory usage. EAGER loading would load everything upfront, causing performance issues with large datasets.

---

**Q34: What is the N+1 query problem and does your code have it?**

**A:** N+1 occurs when you load N entities, and for each one, a separate query fetches related data. For example, if I load 100 hotels and then iterate calling `hotel.getRooms()`, that's 1 + 100 = 101 queries. My code uses LAZY loading which helps, but could still trigger N+1 in loops. Fix: use `@EntityGraph` or JPQL `JOIN FETCH` for batch loading.

---

**Q35: What is `ddl-auto=update` and would you use it in production?**

**A:** `spring.jpa.hibernate.ddl-auto=update` automatically creates/alters tables to match entity definitions. **Never use in production** — it can drop columns, cause data loss, and doesn't support rollbacks. In production, use **Flyway** or **Liquibase** for versioned, reviewable DB migrations.

---

### Category 10: Testing & DevOps

---

**Q36: How is the application containerized?**

**A:** The Dockerfile:
1. Uses `openjdk:23-jdk-slim` as base
2. Copies Maven wrapper + source code
3. Runs `./mvnw clean package -DskipTests` inside the container
4. Copies the JAR and runs with `java -jar app.jar`
5. Exposes port 8080

**Improvement:** Use multi-stage build — build in one stage, copy only the JAR to a slim runtime image. Reduces final image size significantly.

---

**Q37: Where are the environment variables managed?**

**A:** All secrets are externalized via [application.properties](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/resources/application.properties) using `${ENV_VAR}`:
- `DB_URL`, `DB_USERNAME`, `DB_PASSWORD` — Database
- `JWT_SECRET` — JWT signing key
- `FRONTEND_URL` — For Stripe redirect URLs
- `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET` — Stripe keys

These can be injected via Docker env vars, `.env` file, or a secrets manager in production.

---

**Q38: What testing strategy would you use?**

**A:**
1. **Unit tests** — Test services with mocked repositories using Mockito
2. **Integration tests** — Use `@SpringBootTest` + Testcontainers with a real PostgreSQL
3. **API tests** — Use `MockMvc` or RestAssured for controller-level testing
4. **Contract tests** — Ensure DTO contracts don't break

---

### Category 11: Tricky Questions

---

**Q39: If two users search for the same hotel and both see 1 room available, can they both book it?**

**A:** No. The [findAndLockAvailableInventory()](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Repositories/InventoryRepository.java#46-61) uses `PESSIMISTIC_WRITE` which locks the rows. Only one user's transaction proceeds; the second user's query blocks until the first completes. After the first booking, the availability check fails for the second user.

---

**Q40: Why do you create inventory for 365 days? What if the hotel runs for years?**

**A:** Currently, inventory is only initialized for 1 year. For longer periods, you'd need a scheduled job that extends the inventory window (e.g., daily add 1 day to maintain a rolling 365-day window). Alternatively, use a more dynamic approach where inventory is created on-demand when dates beyond the window are queried.

---

**Q41: The HolidayPricingStrategy always returns true for `isTodayHoliday`. Is that correct?**

**A:** No, it's a placeholder. In the real implementation, this would call an external holiday API or check against a local holiday database. Currently, it always applies the 25% holiday surcharge, which is incorrect. This is a TODO item.

---

**Q42: What would you improve in this project if you had more time?**

**A:**
1. **Email notifications** — Booking confirmation, cancellation emails via SendGrid/SES
2. **Expired booking cleanup** — Cron job to release expired reservations
3. **Idempotent webhooks** — Prevent duplicate payment processing
4. **Reviews & ratings** — User review system for hotels
5. **Image upload** — S3/CloudFront for photo management
6. **Search improvements** — Elasticsearch, filters for amenities, price range
7. **Rate limiting** — Prevent booking API abuse
8. **API versioning** — Already using `/api/v1` context path, but proper versioning strategy
9. **Caching** — Redis for frequently accessed hotel data
10. **Multi-stage Docker build** — Reduce image size
11. **Flyway migrations** — Replace ddl-auto=update
12. **Holiday API integration** — Real holiday calendar for pricing

---

**Q43: Why Spring Boot and not Node.js or Python?**

**A:** Java/Spring Boot excels for enterprise-grade backends:
- Strong type safety catches bugs at compile time
- Spring's dependency injection makes code testable and modular
- JPA/Hibernate provides mature ORM with optimistic/pessimistic locking support
- Spring Security provides robust auth framework
- Excellent for concurrent systems with thread management
- Industry standard for fintech/booking platforms where data consistency is critical

# 🏨 AirBnb Backend — Complete Project Deep Dive (Interview Prep)

> **This is YOUR project.** Read it end-to-end before your interview. Every diagram, flow, and question below is derived directly from your actual codebase.

---

## 1. Project Overview

A **hotel booking platform backend** (inspired by Airbnb) built as a monolithic Spring Boot application. It supports:

- **Hotel & Room Management** (Admin/Hotel Manager)
- **Hotel Search & Browse** (Guest Users)
- **Booking Lifecycle** — Init → Reserve → Add Guests → Pay → Confirm / Cancel
- **Dynamic Pricing** using the **Decorator Design Pattern**
- **Stripe Payment Integration** with Webhook-based confirmation
- **JWT-based Authentication** with Access + Refresh tokens
- **Inventory Management** with **Pessimistic Locking** for concurrency control
- **Scheduled Price Updates** (Cron-based)
- **Dockerized Deployment**

---

## 2. Tech Stack

| Layer | Technology |
|---|---|
| **Language** | Java 23 |
| **Framework** | Spring Boot 3.4.2 |
| **ORM** | Spring Data JPA / Hibernate |
| **Database** | PostgreSQL |
| **Security** | Spring Security + JWT (jjwt 0.12.6) |
| **Payments** | Stripe Java SDK 28.2.0 |
| **API Docs** | SpringDoc OpenAPI (Swagger) 2.8.3 |
| **Object Mapping** | ModelMapper 3.2.2 |
| **Boilerplate** | Lombok 1.18.36 |
| **Build** | Maven |
| **Container** | Docker (OpenJDK 23 slim) |

---

## 3. High-Level Design (HLD)

### 3.1 System Architecture

```mermaid
graph TB
    subgraph Clients
        A[Web / Mobile Client]
    end

    subgraph "Spring Boot Application :8080"
        B[Security Filter Chain<br/>JWTAuthFilter]
        C[REST Controllers]
        D[Service Layer]
        E[Repository Layer<br/>Spring Data JPA]
        F[Pricing Engine<br/>Decorator Pattern]
        G[Scheduler<br/>PricingUpdateService]
    end

    subgraph External
        H[(PostgreSQL DB)]
        I[Stripe Payment Gateway]
    end

    A -->|"HTTP /api/v1/*"| B
    B --> C
    C --> D
    D --> E
    D --> F
    D -->|"Checkout Session"| I
    E --> H
    G -->|"Cron: every hour"| E
    I -->|"Webhook POST"| C
```

### 3.2 Layered Architecture

```mermaid
graph LR
    subgraph "Presentation Layer"
        A1[AuthController]
        A2[HotelController]
        A3[HotelBookingController]
        A4[HotelBrowseController]
        A5[InventoryController]
        A6[RoomAdminController]
        A7[UserController]
        A8[WebhookController]
    end

    subgraph "Business Logic Layer"
        B1[AuthService]
        B2[HotelServiceImpl]
        B3[BookingServiceImpl]
        B4[InventoryServiceImpl]
        B5[RoomServiceImpl]
        B6[UserServiceImpl]
        B7[GuestServiceImpl]
        B8[CheckoutServiceImpl]
        B9[PricingService]
        B10[PricingUpdateService]
    end

    subgraph "Data Access Layer"
        C1[HotelRepository]
        C2[RoomRepository]
        C3[BookingRepository]
        C4[InventoryRepository]
        C5[UserRepository]
        C6[GuestRepository]
        C7[HotelMinPriceRepository]
    end

    subgraph "Cross-Cutting"
        D1[GlobalExceptionHandler]
        D2[GlobalResponseHandler]
        D3[WebSecurityConfig]
        D4[JWTAuthFilter]
        D5[JWTService]
    end

    A1 --> B1
    A2 --> B2
    A3 --> B3
    A4 --> B4
    A5 --> B4
    A6 --> B5
    A7 --> B6
    A7 --> B7
    A8 --> B3
    B2 --> C1
    B3 --> C3
    B3 --> C4
    B4 --> C4
    B5 --> C2
    B6 --> C5
    B7 --> C6
    B10 --> C7
```

### 3.3 Entity-Relationship Diagram (Database Schema)

```mermaid
erDiagram
    app_user {
        Long id PK
        String email UK
        String password
        String name
        LocalDate dateOfBirth
        Gender gender
    }

    app_user_roles {
        Long user_id FK
        Role roles
    }

    Hotel {
        Long id PK
        String name
        String city
        String[] photos
        String[] amenities
        Boolean active
        String address
        String phoneNumber
        String email
        String location
        Long owner_id FK
    }

    Room {
        Long id PK
        Long hotel_id FK
        String type
        BigDecimal basePrice
        String[] photos
        String[] amenities
        Integer totalCount
        Integer capacity
    }

    Inventory {
        Long id PK
        Long hotel_id FK
        Long room_id FK
        LocalDate date
        Integer bookCount
        Integer reservedCount
        Integer totalCount
        BigDecimal surgeFactor
        BigDecimal price
        String city
        Boolean closed
    }

    Booking {
        Long id PK
        Long hotel_id FK
        Long room_id FK
        Long user_id FK
        Integer roomsCount
        LocalDate checkInDate
        LocalDate checkOutDate
        BookingStatus bookingStatus
        BigDecimal amount
        String paymentSessionId UK
    }

    Guest {
        Long id PK
        Long user_id FK
        String name
        Gender gender
        Integer age
    }

    booking_guest {
        Long booking_id FK
        Long guest_id FK
    }

    HotelMinPrice {
        Long id PK
        Long hotel_id FK
        LocalDate date
        BigDecimal price
    }

    app_user ||--o{ app_user_roles : "has roles"
    app_user ||--o{ Hotel : "owns"
    app_user ||--o{ Booking : "books"
    app_user ||--o{ Guest : "has"
    Hotel ||--o{ Room : "contains"
    Hotel ||--o{ Inventory : "has inventory"
    Hotel ||--o{ HotelMinPrice : "min prices"
    Room ||--o{ Inventory : "daily slots"
    Hotel ||--o{ Booking : "bookings"
    Room ||--o{ Booking : "bookings"
    Booking }o--o{ Guest : "booking_guest"
```

### 3.4 Role-Based Access Control

```mermaid
graph TD
    subgraph "Public Endpoints - No Auth"
        P1["POST /auth/signup"]
        P2["POST /auth/login"]
        P3["POST /auth/refresh"]
        P4["POST /webhook/payment"]
        P5["Swagger UI"]
    end

    subgraph "ROLE_GUEST + ROLE_HOTEL_MANAGER - Authenticated"
        U1["GET /hotels/search"]
        U2["GET /hotels/{id}/info"]
        U3["POST /bookings/init"]
        U4["POST /bookings/{id}/payments"]
        U5["POST /bookings/{id}/cancel"]
        U6["GET /bookings/{id}/status"]
        U7["GET|PATCH /users/profile"]
        U8["CRUD /users/guests"]
        U9["GET /users/myBookings"]
    end

    subgraph "ROLE_HOTEL_MANAGER Only"
        A1["CRUD /admin/hotels"]
        A2["CRUD /admin/hotels/{id}/rooms"]
        A3["GET|PATCH /admin/inventory"]
        A4["GET /admin/hotels/{id}/bookings"]
        A5["GET /admin/hotels/{id}/reports"]
    end
```

- **GUEST** — default role on signup; can search, book, manage profile & guests
- **HOTEL_MANAGER** — can create/manage hotels, rooms, inventory, view reports
- Security rules are defined in [WebSecurityConfig](file:///c:/Users/HP/OneDrive/Desktop/resume%20proj/airbnb/AirBnb/src/main/java/com/aman/AirBnb/AirBnb/Security/WebSecurityConfig.java#20-67): `/admin/**` → `ROLE_HOTEL_MANAGER`, `/bookings/**` and `/users/**` → authenticated, all else → `permitAll()`

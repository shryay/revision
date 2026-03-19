# Linklytics (URL Shortener) — End-to-End Architecture + HLD/LLD + Interview Prep

## Table of contents
- [1. Project overview](#1-project-overview)
- [2. Tech stack](#2-tech-stack)
- [3. High level design (HLD)](#3-high-level-design-hld)
- [4. Low level design (LLD)](#4-low-level-design-lld)
- [5. Database design](#5-database-design)
- [6. API design (endpoints + contracts)](#6-api-design-endpoints--contracts)
- [7. End-to-end flows](#7-end-to-end-flows)
- [8. Security design](#8-security-design)
- [9. Deployment & runtime](#9-deployment--runtime)
- [10. Senior-style cross questions + strong fresher answers](#10-senior-style-cross-questions--strong-fresher-answers)

---

## 1. Project overview

**Linklytics** is a URL shortening platform with analytics.

### Core capabilities
- **User auth**: register + login, JWT-based auth.
- **Create short URLs**: user submits `originalUrl`, backend generates an 8-char `shortUrl`.
- **Redirect**: visiting `/{shortUrl}` redirects to the original URL.
- **Analytics**:
  - Per-short-url click events aggregated by date.
  - Total clicks per user aggregated by date (across all their URLs).
- **Dashboard UI**: list of user URLs + charts.

Repository structure:
```
linklytics/Linklytics/
├── client/   # React SPA
└── server/   # Spring Boot app (REST + redirect)
```

---

## 2. Tech stack

### Frontend (`client/`)
- **React 18** + **React Router**
- **Vite** build tooling
- **TailwindCSS** + **MUI** components
- **React Query** for data fetching/caching
- **Axios** as HTTP client (baseURL via `VITE_BACKEND_URL`)
- **Chart.js** for analytics visualization
- **Context API** for auth token storage (`JWT_TOKEN` in localStorage)

### Backend (`server/`)
- **Java 21**
- **Spring Boot 3.4**
- **Spring Web** (REST)
- **Spring Security** + Method Security (`@PreAuthorize`)
- **Spring Data JPA**
- **PostgreSQL** JDBC driver (configured in deps)
- **JWT** via `io.jsonwebtoken` (jjwt 0.12.6)

---

## 3. High level design (HLD)

### 3.1 System components
- **Client (React SPA)**: login/register + dashboard + create short URL + charts.
- **Backend (Spring Boot)**:
  - Auth endpoints (`/api/auth/public/*`)
  - URL endpoints (`/api/urls/*`) protected by JWT
  - Redirect endpoint (`/{shortUrl}`) public
- **Database**:
  - `users` table
  - `url_mapping` table
  - `click_event` table

### 3.2 HLD architecture diagram

```mermaid
graph TB
  U[User Browser] --> FE[React SPA - Linklytics]
  FE -->|Axios: VITE_BACKEND_URL| BE[Spring Boot API]
  BE --> DB[(PostgreSQL)]
  U -->|Open short URL: /{shortUrl}| BE
```

### 3.3 Request types
- **API calls** (JSON):
  - Login/Register/Shorten/List/Analytics
- **Redirect calls** (browser navigation):
  - `GET /{shortUrl}` → 302 redirect to original URL

---

## 4. Low level design (LLD)

### 4.1 Frontend design

#### Routing & subdomain routing
The app supports subdomain-based routing using `getApps()` and `subDomainList`:
- Main: `www` (main app)
- Subdomain: `url`

Conceptually:

```mermaid
flowchart TD
  H[hostname] --> SD[getSubDomain()]
  SD -->|'' or www| Main[AppRouter]
  SD -->|url| Sub[SubDomainRouter]
```

#### Auth state
- Token is stored in **Context API** and persisted in `localStorage` as `JWT_TOKEN`.
- Route protection uses `PrivateRoute`:
  - **publicPage=true**: if logged in → redirect to `/dashboard`
  - otherwise: if not logged in → redirect to `/login`

```mermaid
flowchart TD
  LS[localStorage.JWT_TOKEN] --> Ctx[ContextApi token state]
  Ctx --> PR[PrivateRoute]
  PR -->|no token| Login[/login]
  PR -->|token| Dash[/dashboard]
```

#### Data fetching
React Query hooks wrap the backend API:
- `useFetchMyShortUrls(token)` → `GET /api/urls/myurls` then sorts by `createdDate desc`
- `useFetchTotalClicks(token)` → `GET /api/urls/totalClicks?startDate=...&endDate=...` then converts `{date:count}` map to array for Chart.js

```mermaid
sequenceDiagram
  participant UI as DashboardLayout
  participant RQ as React Query
  participant API as Axios (baseURL)
  participant BE as Spring Boot

  UI->>RQ: useFetchMyShortUrls(token)
  RQ->>API: GET /api/urls/myurls (Authorization: Bearer)
  API->>BE: HTTP request
  BE-->>API: 200 List<UrlMappingDTO>
  API-->>RQ: response
  RQ-->>UI: sorted list (select)

  UI->>RQ: useFetchTotalClicks(token)
  RQ->>API: GET /api/urls/totalClicks?startDate&endDate
  BE-->>RQ: 200 Map<LocalDate,Long>
  RQ-->>UI: [{clickDate,count}, ...]
```

---

### 4.2 Backend design

#### Layering
- **Controller layer**: `AuthController`, `UrlMappingController`, (Redirect controller exists by route rule `/{shortUrl}`)
- **Service layer**: `UserService`, `UrlMappingService`
- **Repository layer**: `UserRepository`, `UrlMappingRepository`, `ClickEventRepository`
- **Security layer**: `WebSecurityConfig`, `JwtAuthenticationFilter`, `JwtUtils`, `UserDetailsServiceImpl`

#### Core domain objects (entities)
- `User` (table: `users`)
- `UrlMapping` (maps original → short, belongs to User)
- `ClickEvent` (timestamped click record, belongs to UrlMapping)

#### Backend class diagram (LLD)

```mermaid
classDiagram
  class AuthController {
    +loginUser(LoginRequest) ResponseEntity
    +registerUser(RegisterRequest) ResponseEntity
  }

  class UrlMappingController {
    +createShortUrl(request, principal) ResponseEntity~UrlMappingDTO~
    +getUserUrls(principal) ResponseEntity~List~UrlMappingDTO~~
    +getUrlAnalytics(shortUrl,startDate,endDate) ResponseEntity~List~ClickEventDTO~~
    +getTotalClicksByDate(principal,startDate,endDate) ResponseEntity~Map~LocalDate,Long~~
  }

  class UserService {
    +registerUser(User) User
    +authenticateUser(LoginRequest) JwtAuthenticationResponse
    +findByUsername(username) User
  }

  class UrlMappingService {
    +createShortUrl(originalUrl, user) UrlMappingDTO
    +getUrlsByUser(user) List~UrlMappingDTO~
    +getClickEventsByDate(shortUrl,start,end) List~ClickEventDTO~
    +getTotalClicksByUserAndDate(user,start,end) Map~LocalDate,Long~
    +getOriginalUrl(shortUrl) UrlMapping
  }

  class WebSecurityConfig {
    +filterChain(HttpSecurity) SecurityFilterChain
    +passwordEncoder() PasswordEncoder
    +authenticationProvider() DaoAuthenticationProvider
  }

  class JwtAuthenticationFilter {
    +doFilterInternal(req,res,chain)
  }

  class JwtUtils {
    +generateToken(UserDetailsImpl) String
    +getJwtFromHeader(req) String
    +getUserNameFromJwtToken(token) String
    +validateToken(token) boolean
  }

  class UserDetailsServiceImpl {
    +loadUserByUsername(username) UserDetails
  }

  class UserDetailsImpl {
    +build(User) UserDetailsImpl
  }

  class UserRepository {
    <<interface>>
    +findOneByUsername(username) Optional~User~
    +existsByUsername(username) boolean
    +existsByEmail(email) boolean
  }

  class UrlMappingRepository {
    <<interface>>
    +findByShortUrl(shortUrl) UrlMapping
    +findByUser(user) List~UrlMapping~
  }

  class ClickEventRepository {
    <<interface>>
    +findByUrlMappingAndClickDateBetween(urlMapping,start,end) List~ClickEvent~
    +findByUrlMappingInAndClickDateBetween(urlMappings,start,end) List~ClickEvent~
  }

  class User {
    +Long id
    +String email
    +String username
    +String password
    +String role
  }

  class UrlMapping {
    +Long id
    +String originalUrl
    +String shortUrl
    +int clickCount
    +LocalDateTime createdDate
    +User user
    +List~ClickEvent~ clickEvents
  }

  class ClickEvent {
    +Long id
    +LocalDateTime clickDate
    +UrlMapping urlMapping
  }

  AuthController --> UserService
  UrlMappingController --> UrlMappingService
  UrlMappingController --> UserService
  UserService --> UserRepository
  UrlMappingService --> UrlMappingRepository
  UrlMappingService --> ClickEventRepository
  WebSecurityConfig --> JwtAuthenticationFilter
  JwtAuthenticationFilter --> JwtUtils
  JwtAuthenticationFilter --> UserDetailsServiceImpl
  UserDetailsServiceImpl --> UserRepository
```

---

## 5. Database design

### 5.1 ER diagram

```mermaid
erDiagram
  USERS {
    BIGINT id PK
    VARCHAR email "unique, not null"
    VARCHAR username "unique, not null"
    VARCHAR password "BCrypt hash"
    VARCHAR role "ROLE_USER"
  }

  URL_MAPPING {
    BIGINT id PK
    VARCHAR original_url
    VARCHAR short_url
    INT click_count
    TIMESTAMP created_date
    BIGINT user_id FK
  }

  CLICK_EVENT {
    BIGINT id PK
    TIMESTAMP click_date
    BIGINT url_mapping_id FK
  }

  USERS ||--o{ URL_MAPPING : owns
  URL_MAPPING ||--o{ CLICK_EVENT : generates
```

### 5.2 Why store click events as rows?
- Enables flexible analytics queries later (by date range, by URL, by user).
- Trade-off: more rows; needs indexing and retention policy at scale.

Recommended indexes (production thinking):
- `url_mapping.short_url` unique index
- `url_mapping.user_id` index
- `click_event.url_mapping_id` index
- `click_event.click_date` index
- composite `(url_mapping_id, click_date)`

---

## 6. API design (endpoints + contracts)

### 6.1 Auth API

| Method | Endpoint | Auth | Request | Response |
|---|---|---|---|---|
| POST | `/api/auth/public/register` | Public | `{ username, email, password }` | `{ status, message, userId }` |
| POST | `/api/auth/public/login` | Public | `{ username, password }` | `{ token }` |

### 6.2 URL shortener API (JWT protected)

| Method | Endpoint | Auth | Request | Response |
|---|---|---|---|---|
| POST | `/api/urls/shorten` | Bearer JWT + `ROLE_USER` | `{ originalUrl }` | `UrlMappingDTO` |
| GET | `/api/urls/myurls` | Bearer JWT + `ROLE_USER` | - | `UrlMappingDTO[]` |
| GET | `/api/urls/analytics/{shortUrl}?startDate&endDate` | Bearer JWT + `ROLE_USER` | ISO_LOCAL_DATE_TIME | `ClickEventDTO[]` (daily counts) |
| GET | `/api/urls/totalClicks?startDate&endDate` | Bearer JWT + `ROLE_USER` | ISO_LOCAL_DATE | `Map<LocalDate,Long>` |

DTOs:
- `UrlMappingDTO`: `{ id, originalUrl, shortUrl, clickCount, createdDate, username }`
- `ClickEventDTO`: `{ clickDate: LocalDate, count: Long }`

### 6.3 Redirect endpoint (public)

| Method | Endpoint | Auth | Behavior |
|---|---|---|---|
| GET | `/{shortUrl}` | Public | Lookup shortUrl → increment clickCount → insert ClickEvent → 302 redirect |

---

## 7. End-to-end flows

### 7.1 Register flow

```mermaid
sequenceDiagram
  participant U as User
  participant FE as React
  participant BE as Spring Boot
  participant DB as Postgres

  U->>FE: Fill register form
  FE->>BE: POST /api/auth/public/register
  BE->>DB: Check username/email unique
  BE->>BE: BCrypt hash password
  BE->>DB: INSERT users
  DB-->>BE: userId
  BE-->>FE: success + userId
```

### 7.2 Login flow (JWT)

```mermaid
sequenceDiagram
  participant U as User
  participant FE as React
  participant SEC as Spring Security
  participant BE as UserService
  participant JWT as JwtUtils

  U->>FE: Submit username + password
  FE->>BE: POST /api/auth/public/login
  BE->>SEC: AuthenticationManager.authenticate()
  SEC-->>BE: principal(UserDetailsImpl)
  BE->>JWT: generateToken(userDetails)
  JWT-->>BE: JWT token
  BE-->>FE: {token}
  FE->>FE: store localStorage.JWT_TOKEN + Context token
  FE-->>U: redirect /dashboard
```

### 7.3 Create short URL flow

```mermaid
sequenceDiagram
  participant U as User
  participant FE as CreateNewShorten
  participant SEC as JwtAuthenticationFilter
  participant BE as UrlMappingService
  participant DB as Postgres

  U->>FE: Enter original URL
  FE->>SEC: POST /api/urls/shorten (Bearer JWT)
  SEC->>SEC: validate token + set SecurityContext
  SEC-->>BE: authenticated Principal(username)
  BE->>DB: INSERT url_mapping(shortUrl, originalUrl, user_id, createdDate)
  DB-->>BE: UrlMapping row
  BE-->>FE: UrlMappingDTO {shortUrl,...}
  FE->>FE: copy `${FRONTEND}/s/{shortUrl}` to clipboard
```

### 7.4 Redirect + click tracking flow

```mermaid
sequenceDiagram
  participant Browser as Browser (anonymous)
  participant BE as Spring Boot
  participant DB as Postgres

  Browser->>BE: GET /{shortUrl}
  BE->>DB: SELECT url_mapping by shortUrl
  alt found
    BE->>DB: UPDATE url_mapping.click_count += 1
    BE->>DB: INSERT click_event(click_date=now, url_mapping_id)
    BE-->>Browser: 302 Location: originalUrl
  else not found
    BE-->>Browser: 404 / error
  end
```

### 7.5 Analytics flow (per URL)

```mermaid
sequenceDiagram
  participant UI as Dashboard
  participant BE as /api/urls/analytics
  participant DB as Postgres

  UI->>BE: GET /api/urls/analytics/{shortUrl}?startDate&endDate (JWT)
  BE->>DB: SELECT click_event WHERE url_mapping_id AND click_date between
  BE->>BE: group by LocalDate, count
  BE-->>UI: [{clickDate,count}, ...]
  UI->>UI: render Chart.js bar chart
```

### 7.6 Total clicks flow (across user URLs)

```mermaid
sequenceDiagram
  participant UI as Dashboard
  participant BE as /api/urls/totalClicks
  participant DB as Postgres

  UI->>BE: GET /api/urls/totalClicks?startDate&endDate (JWT)
  BE->>DB: find url_mapping rows by user_id
  BE->>DB: select click_event where url_mapping_id in (...) and date range
  BE->>BE: group by LocalDate, count
  BE-->>UI: { \"2024-01-01\": 120, ... }
  UI->>UI: convert map -> array -> chart
```

---

## 8. Security design

### 8.1 What is protected vs public?
- Public:
  - `/api/auth/**`
  - `GET /{shortUrl}` redirect endpoint
- Protected:
  - `/api/urls/**` requires JWT authentication

### 8.2 How JWT auth works
- On login, backend issues a JWT whose **subject is username** and includes a `roles` claim.
- `JwtAuthenticationFilter` runs for each request:
  - extracts Bearer token from `Authorization`
  - validates signature/expiry via `JwtUtils.validateToken`
  - loads `UserDetails` and sets `SecurityContext`
- Controllers then use `Principal principal` + `@PreAuthorize("hasRole('USER')")`.

```mermaid
sequenceDiagram
  participant C as Client
  participant F as JwtAuthenticationFilter
  participant J as JwtUtils
  participant UDS as UserDetailsService
  participant SC as SecurityContext
  participant API as Controller

  C->>F: Request + Authorization: Bearer <jwt>
  F->>J: validateToken(jwt)
  J-->>F: ok
  F->>J: getUserNameFromJwtToken(jwt)
  J-->>F: username
  F->>UDS: loadUserByUsername(username)
  UDS-->>F: UserDetails
  F->>SC: setAuthentication(UserDetails)
  F-->>API: proceed
```

### 8.3 CORS
Backend allows CORS only from `frontend.url` property (single origin), allows credentials, and permits `OPTIONS`.

### 8.4 Security gaps / improvements (good interview points)
- **Short URL collisions**: random 8 chars can collide; add uniqueness check + retry; DB unique constraint.
- **Brute force guessing**: rate-limit redirect endpoint and analytics endpoints.
- **Token storage**: client stores JWT in localStorage → XSS risk. Prefer httpOnly cookies in production.
- **ClickEvent retention**: at scale, store aggregates daily + keep raw events for limited time.

---

## 9. Deployment & runtime

### 9.1 Backend Dockerfile
- Multi-stage build: Maven wrapper builds JAR, runtime image runs `java -jar`.
- Exposes `8080`.

### 9.2 Environment variables (client)
- `VITE_BACKEND_URL`: axios base URL for backend.
- `VITE_REACT_FRONT_END_URL`: used to construct the final shareable URL.

### 9.3 Typical run commands

Frontend:
```bash
cd client
npm install
npm run dev
```

Backend:
```bash
cd server
./mvnw spring-boot:run
```

---

## 10. Senior-style cross questions + strong fresher answers

### 10.1 Architecture & HLD

- **Q: Explain the system end-to-end in 60 seconds.**
  - **A:** React SPA handles auth and a dashboard. It calls a Spring Boot REST API via Axios. Login issues a JWT; the client stores it and sends it in `Authorization` headers for protected endpoints. Backend stores users, URL mappings, and click events in Postgres through JPA. The redirect endpoint is public: it resolves the short code, records a click event, increments click count, and redirects.

- **Q: Why is `GET /{shortUrl}` public? Isn’t that unsafe?**
  - **A:** It must be public because anyone clicking a short link shouldn’t need to log in. Security is handled via randomness of short code, rate limiting, and abuse prevention.

- **Q: What happens if two users generate the same shortUrl?**
  - **A:** Collision risk exists. Fix: enforce unique index on `short_url` and on collision retry generation. For stronger guarantees, use base62 encoding of an ID (or Snowflake) instead of pure random.

### 10.2 Authentication & security

- **Q: Walk me through how JWT authentication works in Spring Security here.**
  - **A:** JWT filter reads Bearer token from `Authorization`, validates with HMAC secret, extracts username, loads user details, and sets `SecurityContext`. Then controllers can rely on `Principal` and `@PreAuthorize` role checks.

- **Q: Why BCrypt for passwords?**
  - **A:** BCrypt is adaptive hashing (slow on purpose), making brute-force harder if the DB leaks.

- **Q: Is storing JWT in localStorage safe?**
  - **A:** It’s vulnerable to XSS token theft. In production, I’d prefer httpOnly Secure cookies + CSRF protections, plus a strong CSP.

### 10.3 Backend design & data model

- **Q: Why do you store `ClickEvent` as a separate table?**
  - **A:** It enables analytics by date range and per link. If we only stored clickCount, we couldn’t build graphs. At scale, we can store daily aggregates and keep raw events for limited retention.

- **Q: How do you compute `/totalClicks`?**
  - **A:** Fetch all UrlMappings of the user, then fetch ClickEvents in that date range for those mappings, then group by LocalDate and count.

- **Q: What indexes would you add first?**
  - **A:** Unique index on `url_mapping.short_url`; index on `url_mapping.user_id`; composite index on `(click_event.url_mapping_id, click_event.click_date)`.

### 10.4 Frontend design

- **Q: How do you protect routes?**
  - **A:** `PrivateRoute` checks context token; if missing it redirects to `/login`. For public pages like login/register, if already logged in it redirects to dashboard.

- **Q: Why React Query?**
  - **A:** It caches server state, handles loading/error states, and supports refetching. Here it also uses `select` to sort/shape data before rendering.

### 10.5 System design improvements

- **Q: How would you make redirect endpoint fast under huge traffic?**
  - **A:** Use cache (Redis) for `shortUrl -> originalUrl`, write click events asynchronously (queue), and keep counters in Redis with periodic flush to DB.

- **Q: How would you prevent abuse (bots) hitting short URLs?**
  - **A:** Rate limit by IP/user agent, add WAF rules, detect anomalies, optionally require captcha for suspicious patterns (not for normal users).

### 10.6 “Senior catch” questions

- **Q: Your frontend builds share URL as `${FRONTEND}/s/{shortUrl}` but backend redirect is `/{shortUrl}`. Isn’t that inconsistent?**
  - **A:** Yes, it implies there is a frontend route `/s/:shortUrl` that likely forwards to backend or a reverse proxy that maps `/s/*` to backend `/*`. If not present, the fix is to align: either generate links as `/{shortUrl}` (backend-owned), or create a client route that calls backend and redirects.

- **Q: Why is `useFetchTotalClicks` hardcoding date range?**
  - **A:** It’s a simplification. In production, the dashboard would allow date range selection and pass it dynamically.

---

# Part A — 60–90 second pitch (2 min)

## Q1. Explain Linklytics end-to-end in 60–90 seconds.

**Strong answer (say this):**  
Linklytics is a URL shortener with analytics. The React frontend handles login/register and a dashboard. Backend is a Spring Boot REST API with Spring Security + JWT. Users create short URLs; those are stored in Postgres via JPA as UrlMapping rows, linked to the user. When anyone visits /{shortUrl}, the backend looks up the original URL, increments clickCount, stores a ClickEvent row with timestamp, and returns a redirect. Dashboard calls protected endpoints to list my URLs and to fetch analytics aggregated by date, which the UI renders using Chart.js.

**Follow-ups:**  
- Why JWT instead of session?  
- Why store ClickEvent rows instead of only clickCount?  

---

# Part B — Frontend (3–4 min)

## Q2. How do you protect routes in the frontend?

**Strong answer:**  
I use a PrivateRoute wrapper. It reads the JWT token from Context API (which initializes from localStorage.JWT_TOKEN). If a protected page is accessed without a token, it redirects to /login. For public pages, if token exists it redirects to /dashboard.

**Follow-ups:**  
- What happens on browser refresh?  
- Any security risk storing JWT in localStorage?  

---

## Q3. How do frontend API calls work?

**Strong answer:**  
Axios is configured with a baseURL from VITE_BACKEND_URL. For protected calls, I attach Authorization: Bearer <token>. React Query handles caching/loading/error and transforms responses using select (sorting URLs by createdDate, converting the totalClicks map to an array for charts).

**Follow-ups:**  
- Why React Query? What problem does it solve?  
- How would you handle global 401 errors (token expired)?  

---

# Part C — Backend: auth + security (5 min)

## Q4. Walk me through register flow.

**Strong answer:**  
Frontend calls POST /api/auth/public/register. Backend validates uniqueness for username/email, hashes password with BCrypt, stores a User row with role ROLE_USER, and returns success.

**Follow-ups:**  
- Why BCrypt?  
- How do you validate email format / password strength (where would you add it)?  

---

## Q5. Walk me through login and JWT generation.

**Strong answer:**  
Frontend posts POST /api/auth/public/login with username/password. Spring Security AuthenticationManager authenticates using UserDetailsServiceImpl + BCrypt encoder. On success, JwtUtils.generateToken() creates a signed token with subject=username and a roles claim, and backend returns { token }.

**Follow-ups:**  
- What claims are in your token and why?  
- How do you set token expiration and validate expiry?  

---

## Q6. Explain how JWT is validated on protected endpoints.

**Strong answer:**  
JwtAuthenticationFilter runs for each request. It extracts Bearer token from Authorization header, validates signature/expiry, extracts username, loads user details, and sets SecurityContext. Then @PreAuthorize("hasRole('USER')") checks authorization, and controllers can use Principal to know the current user.

**Follow-ups (senior):**  
- What if token is valid but user was deleted?  
- What if someone modifies the roles claim?  

---

## Q7. CORS/CSRF: why is CSRF disabled and what about CORS?

**Strong answer:**  
CSRF is typically for cookie-based auth. Here we use stateless JWT in Authorization header, so CSRF protection is less relevant and disabled. CORS is configured to allow requests from a single configured frontend origin and to permit OPTIONS.

**Follow-ups:**  
- If you switch to httpOnly cookies, what changes?  

---

# Part D — Backend: URL flows + analytics (6–7 min)

## Q8. How do you generate a short URL and store it?

**Strong answer:**  
Endpoint POST /api/urls/shorten is protected. It reads originalUrl, gets the user from Principal, generates an 8-character base62-like random string, stores a UrlMapping entity with originalUrl, shortUrl, createdDate, clickCount=0, and the User relation, then returns UrlMappingDTO.

**Follow-ups (important):**  
- Collision: how do you prevent two URLs from getting same short code?  
- What DB constraint would you add?  
- Would you allow custom alias? How?  

---

## Q9. Explain the redirect flow and click tracking.

**Strong answer:**  
Public endpoint GET /{shortUrl} looks up the mapping by shortUrl. If found, it increments clickCount, saves it, inserts a new ClickEvent with clickDate=now linked to that mapping, then responds with an HTTP redirect to originalUrl.

**Follow-ups (senior):**  
- What status code do you use (301 vs 302) and why?  
- How do you handle invalid/expired short URLs?  
- How do you avoid the redirect endpoint becoming the bottleneck?  

---

## Q10. How does per-link analytics work?

**Strong answer:**  
GET /api/urls/analytics/{shortUrl}?startDate&endDate loads click events for that shortUrl in the time range, groups by LocalDate, counts events per day, and returns ClickEventDTO[] where each item is {clickDate, count}.

**Follow-ups:**  
- Why is startDate parsed as ISO_LOCAL_DATE_TIME here?  
- What happens if startDate/endDate is invalid?  

---

## Q11. How does “total clicks across all my links” work?

**Strong answer:**  
GET /api/urls/totalClicks?startDate&endDate finds all UrlMappings for the user, fetches click events in the date range for those mappings, groups by LocalDate and counts. It returns a map of date → click count which frontend converts to chart data.

**Follow-ups (senior):**  
- This query can be expensive. What indexes do you add?  
- How would you redesign for millions of click events/day?  

---

# Part E — Database + scaling (3–4 min)

## Q12. Describe your tables and relationships.

**Strong answer:**  
users stores account info and role. url_mapping stores shortUrl/originalUrl/clickCount/createdDate and has a many-to-one relationship to users. click_event stores clickDate and links many-to-one to url_mapping. One UrlMapping has many ClickEvents.

**Follow-ups:**  
- What should be unique? (username, email, short_url)  
- What indexes do you add first and why?  

---

## Q13. How would you scale redirects and analytics?

**Strong answer:**  
Cache shortUrl→originalUrl in Redis to reduce DB reads. Record click events asynchronously using a queue/stream and batch inserts. Maintain daily aggregated counters (materialized view or separate table) so analytics doesn’t scan raw events. Add rate limiting at edge.

**Follow-ups (senior):**  
- Exactly-once vs at-least-once click counting: what do you choose?  
- How do you handle bot traffic?  

---

# Part F — Senior “catch” questions (2–3 min)

## Q14. Your frontend constructs share URLs as /s/{shortUrl}, but backend redirect is /{shortUrl}. Explain.

**Strong answer:**  
That’s a mismatch that needs alignment. Either the frontend must generate /{shortUrl} links (backend owns redirect), or we add a frontend route /s/:shortUrl that calls backend redirect endpoint or triggers navigation. In production, I’d unify it and write tests around it.

---

## Q15. Your shortUrl generator is random. What guarantees uniqueness?

**Strong answer:**  
Random alone doesn’t guarantee uniqueness. Proper approach: add a unique constraint on short_url and retry generation on conflict; or use deterministic encoding of a unique ID (base62 of DB id / Snowflake).

---

# Quick “best answers” to memorize (30 seconds each)

- Why ClickEvent table? To build time-series analytics; clickCount alone can’t show trends.  
- Why JWT? Stateless auth, easy scaling, works for API clients.  
- Main risks? token in localStorage (XSS), short code collisions, DB load on redirect, clickEvent table growth.  
- First improvements? unique index + collision retry, unify /s/ vs /, add redis cache + async click ingestion, better error handling + token expiry handling.

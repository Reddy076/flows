# Session Management — Complete Flow Document
> Part of the `user-service` deep-dive series.

This document explains the full end-to-end flow of every Session Management feature,
from the frontend HTTP request all the way through to the database — including every
file involved, every dependency used, and ASCII diagrams for each flow.

---

## What is a Session in This Application?

This application uses a **dual-layer session system**:

```
LAYER 1: JWT TOKEN (stateless)
    A signed string containing: username + expiry date
    Verified by checking its cryptographic signature (no DB needed for this)
    Lives in: frontend memory / localStorage
    Expiry: ~15-60 minutes (short-lived, configurable in JwtConfig)

LAYER 2: DATABASE SESSION (stateful)
    A UserSession row in MySQL with: token, ip, device, location, is_active, expiry
    Used for: session listing, device tracking, manual revocation
    Expiry: 7 days (can be extended)

WHY BOTH?
    JWT alone cannot be revoked before expiry.
    If a user logs out or their account is compromised, the JWT might still be valid
    for another 30 minutes.
    The DB session is the "off switch" — when you mark is_active=false, the
    JwtAuthenticationFilter checks this and REJECTS the token immediately,
    even if the JWT cryptographic signature is technically still valid.
```

---

## Files Involved

| File | Package | Role |
| :--- | :--- | :--- |
| `SessionController.java` | `controller` | Entry point for all `/api/sessions/**` endpoints |
| `SessionService.java` | `service/auth` | All session business logic — create, list, extend, terminate |
| `JwtAuthenticationFilter.java` | `security` | Calls `sessionService.isSessionActive()` on EVERY request |
| `JwtTokenProvider.java` | `security` | Extracts username and expiry date from JWT tokens |
| `ClientIpUtil.java` | `util` | Extracts real client IP (handles proxy headers like X-Forwarded-For) |
| `UserSessionRepository.java` | `repository` | All DB queries for the `user_sessions` table |
| `UserRepository.java` | `repository` | Needed to look up user by username/email |
| `UserSession.java` | `model/user` | JPA entity representing one session row in the DB |
| `JwtAuthFilter.java` | `api-gateway` | First-line JWT check at gateway level |
| `RateLimitGlobalFilter.java` | `api-gateway` | Rate limiting (100 req/min) before request reaches user-service |
| `CorsConfig.java` | `config` | CORS rules for browser preflight |
| `SecurityConfig.java` | `config` | Marks all `/api/sessions/**` as JWT-protected |

---

## The UserSession Entity — Database Structure

```java
@Entity
@Table(name = "user_sessions")
public class UserSession {
    Long id;                    // Primary key
    User user;                  // FK → users.id
    String token;               // The actual JWT string (max 2048 chars)
    String ipAddress;           // e.g. "192.168.1.100"
    String deviceInfo;          // User-Agent header: "Chrome/121 on Windows"
    String deviceFingerprint;   // Hash of device characteristics
    String location;            // e.g. "Mumbai, India" (from GeoLocationService)
    boolean isActive;           // TRUE = valid, FALSE = logged out / terminated
    LocalDateTime createdAt;    // When was this session first created
    LocalDateTime lastAccessedAt; // Updated when session is extended
    LocalDateTime expiresAt;    // Created + 7 days (or extended + 7 days)
}
```

---

## How Requests Pass Through the Gateway + CORS (for all session endpoints)

```
FRONTEND (Angular)
    |
    | Any request to /api/sessions/**
    | Headers: Authorization: Bearer eyJ...
    | Origin: http://localhost:4200
    |
    v
[API GATEWAY - port 8080]
    |-- RateLimitGlobalFilter: 100 req/min per IP
    |-- JwtAuthFilter:
    |       /api/sessions/** is NOT /api/auth/** → JWT IS required
    |       Validates signature with shared HMAC-SHA256 secret
    |       Adds X-User-Name header, forwards to user-service:8081
    v
[USER-SERVICE - Spring Security]
    |-- CorsConfig: validates Origin
    |-- JwtAuthenticationFilter (the critical one):
    |       jwtTokenProvider.validateToken(jwt)  → signature + expiry check
    |       sessionService.isSessionActive(jwt)  → MYSQL SELECT (is_active = true?)
    |       If both pass → sets SecurityContext
    |-- SecurityConfig: /api/sessions/** → anyRequest().authenticated()
    v
[SessionController.java]
```

---

## Feature 1 — List All Active Sessions

### What it does
Shows the user every device/location where they are currently logged in. This allows
them to spot suspicious logins from unknown devices or locations.

### ASCII Flow Diagram

```
FRONTEND
    |
    | GET /api/sessions
    | Headers: Authorization: Bearer <token>
    |
    v
[API GATEWAY] --> [JwtAuthenticationFilter] --> [SessionController.getActiveSessions()]
    |
    | @AuthenticationPrincipal UserDetails userDetails
    |   → Spring provides the logged-in user's UserDetails automatically
    |
    v
[SessionService.getUserSessions(username)]
    |
    |-- userRepository.findByUsername(username)
    |       OR findByEmail(username)   (handles both login types)
    |       --> MYSQL SELECT from users
    |
    |-- userSessionRepository.findByUserIdAndIsActiveTrue(userId)
    |       --> MYSQL SELECT FROM user_sessions
    |           WHERE user_id = ? AND is_active = true
    |
    |-- Maps each UserSession to SessionResponse:
    |       { id, ipAddress, deviceInfo, location, isActive, createdAt,
    |         lastAccessedAt, expiresAt }
    |       NOTE: the raw JWT token is NEVER included in the response
    |
    v
[RESPONSE 200 OK]
    | Body: [
    |   {
    |     id: 1,
    |     ipAddress: "192.168.1.100",
    |     deviceInfo: "Mozilla/5.0 (Windows NT 10.0) Chrome/121",
    |     location: "Mumbai, India",
    |     isActive: true,
    |     createdAt: "2026-03-20T10:00:00",
    |     lastAccessedAt: "2026-03-23T18:00:00",
    |     expiresAt: "2026-03-27T10:00:00"
    |   },
    |   {
    |     id: 2,
    |     ipAddress: "10.0.0.5",
    |     deviceInfo: "Safari on iPhone",
    |     location: "Delhi, India",
    |     ...
    |   }
    | ]
```

---

## Feature 2 — Get Current Session

### What it does
Returns the details of the session associated with the specific JWT token
in the current request. Useful for the frontend to display "This device" info.

### ASCII Flow Diagram

```
FRONTEND
    |
    | GET /api/sessions/current
    | Headers: Authorization: Bearer eyJ...
    |
    v
[SessionController.getCurrentSession(authHeader)]
    |
    | Extracts token: authHeader.substring(7)
    |   (strips "Bearer " prefix from the header value)
    |
    v
[SessionService.getCurrentSession(token)]
    |
    |-- userSessionRepository.findFirstByTokenOrderByIdDesc(token)
    |       --> MYSQL SELECT FROM user_sessions
    |           WHERE token = ? ORDER BY id DESC LIMIT 1
    |       Ordered by id DESC to handle edge case of duplicate tokens (failsafe)
    |
    |-- If not found → throw ResourceNotFoundException("Session not found")
    |
    |-- mapToResponse(session)
    |       → SessionResponse without the raw token field
    |
    v
[RESPONSE 200 OK]
    | Body: { id, ipAddress, deviceInfo, location, createdAt, lastAccessedAt, expiresAt }
```

---

## Feature 3 — Extend Session

### What it does
Resets the session's `expiresAt` timestamp to **now + 7 days** and updates
`lastAccessedAt` to now. This is called by the frontend to keep the user
logged in without requiring a full re-login (like a "keep me logged in" mechanism).

### ASCII Flow Diagram

```
FRONTEND
    | (user has been active, want to extend before session expires)
    |
    | POST /api/sessions/extend
    | Headers: Authorization: Bearer eyJ...
    |
    v
[SessionController.extendSession(authHeader)]
    |
    | Extracts token from Authorization header
    |
    v
[SessionService.extendSession(token)]
    |
    |-- STEP 1: FIND SESSION
    |   userSessionRepository.findFirstByTokenOrderByIdDesc(token) --> MYSQL SELECT
    |   If not found --> throw ResourceNotFoundException
    |
    |-- STEP 2: IS IT STILL ACTIVE?
    |   If session.isActive() == false
    |       --> throw AuthenticationException("Cannot extend an inactive session")
    |       (Protects against extending a manually terminated session)
    |
    |-- STEP 3: EXTEND TIMESTAMPS
    |   session.setLastAccessedAt(LocalDateTime.now())   -- mark as recently used
    |   session.setExpiresAt(LocalDateTime.now() + 7 days)  -- push expiry forward
    |   userSessionRepository.save(session) --> MYSQL UPDATE user_sessions
    |
    v
[RESPONSE 200 OK]
    | Body: { id, ipAddress, deviceInfo, location, lastAccessedAt, expiresAt(+7days) }
```

---

## Feature 4 — Terminate a Specific Session (Remote Logout)

### What it does
Allows a user to remotely log out of a specific device. For example:
"I see a session from Delhi that I don't recognise — terminate it."
The session row is NOT deleted — it is marked `is_active = false`,
so the JwtAuthenticationFilter will instantly reject any future requests using that token.

### ASCII Flow Diagram

```
FRONTEND
    | (user sees suspicious session in the list, clicks "Log out this device")
    |
    | DELETE /api/sessions/{sessionId}
    | e.g. DELETE /api/sessions/2
    | Headers: Authorization: Bearer <current_token>
    |
    v
[SessionController.terminateSession(sessionId, userDetails)]
    |
    | @AuthenticationPrincipal UserDetails userDetails
    |   → Spring provides current user's username from SecurityContext
    |
    v
[SessionService.terminateSession(sessionId, username)]
    |
    |-- STEP 1: FIND THE SESSION BY ID
    |   userSessionRepository.findById(sessionId) --> MYSQL SELECT
    |   If not found → throw ResourceNotFoundException
    |
    |-- STEP 2: OWNERSHIP CHECK (CRITICAL SECURITY CHECK)
    |   session.getUser().getUsername().equals(username)?
    |   If NO → throw ResourceNotFoundException
    |       (user cannot terminate ANOTHER user's session)
    |       (We intentionally throw "not found" and NOT "forbidden"
    |        to avoid leaking session ownership information)
    |
    |-- STEP 3: MARK AS INACTIVE
    |   session.setActive(false)
    |   userSessionRepository.save(session) --> MYSQL UPDATE
    |       SET is_active = false WHERE id = ?
    |
    v
[RESPONSE 200 OK]
    | Body: { message: "Session terminated successfully" }

EFFECT ON THE TERMINATED DEVICE:
    | Next API call from that device:
    | GET /api/vault/entries
    | Authorization: Bearer <terminated_token>
    |
    v
[JwtAuthenticationFilter.java]
    |-- jwtTokenProvider.validateToken(jwt) → PASSES (JWT still cryptographically valid)
    |-- sessionService.isSessionActive(jwt)
    |       userSessionRepository.findFirstByTokenOrderByIdDesc(token)
    |       session.isActive() == FALSE ← was just set to false
    |       --> returns false
    |-- SecurityContext is NOT populated (auth fails silently)
    |-- SecurityConfig: anyRequest().authenticated() → returns 401 Unauthorized
    v
[RESPONSE 401 Unauthorized]
    | Body: { "error": "Unauthorized" }
    | The terminated device is now effectively logged out immediately.
```

---

## Feature 5 — Terminate ALL Sessions (Global Logout)

### What it does
Logs the user out of **every device at once** — every active session is
set to `is_active = false`. This is the "nuclear option" used when an
account has been compromised or the user wants to start fresh.

### ASCII Flow Diagram

```
FRONTEND
    | (user clicks "Log out all devices" in security settings)
    |
    | DELETE /api/sessions
    | Headers: Authorization: Bearer <current_token>
    |
    v
[SessionController.terminateAllSessions(userDetails)]
    |
    v
[SessionService.terminateAllUserSessions(username)]
    |
    |-- STEP 1: GET USER
    |   userRepository.findByUsername(username)
    |       OR findByEmail(username) → MYSQL SELECT
    |   If not found → throw ResourceNotFoundException
    |
    |-- STEP 2: FIND ALL ACTIVE SESSIONS
    |   userSessionRepository.findByUserIdAndIsActiveTrue(userId)
    |       --> MYSQL SELECT FROM user_sessions
    |           WHERE user_id = ? AND is_active = true
    |       Returns ALL active sessions (could be 5, 10, or 50 devices)
    |
    |-- STEP 3: DEACTIVATE ALL AT ONCE
    |   For each session: session.setActive(false)
    |   userSessionRepository.saveAll(sessions) --> MYSQL batch UPDATE
    |       SET is_active = false WHERE id IN (1, 2, 3, ...)
    |
    v
[RESPONSE 200 OK]
    | Body: { message: "All sessions terminated successfully" }

EFFECT:
    Every device (phone, laptop, tablet) with an active session will get
    401 Unauthorized on their next API call — they are all simultaneously
    logged out, even if their JWT has not expired yet.
```

---

## How Sessions Are Created (for context)

Sessions are created during login — managed by `SessionService.createSession()`
which is called by `AuthenticationService` after successful credential validation:

```
[AuthenticationService.authenticate()]
    |
    |-- After JWT tokens generated
    |
    v
[SessionService.createSession(user, token, request, location, fingerprint)]
    |
    |-- clientIpUtil.getClientIpAddress(request)
    |       Checks headers in order:
    |           X-Forwarded-For (set by load balancers/proxies)
    |           X-Real-IP
    |           RemoteAddr (last resort — direct connection IP)
    |       Returns the real client IP, not the proxy's IP
    |
    |-- request.getHeader("User-Agent")
    |       → e.g. "Mozilla/5.0 (Windows NT 10.0; Win64) AppleWebKit Chrome/121"
    |
    |-- Builds UserSession entity:
    |       user: the authenticated User object
    |       token: the raw JWT access token string
    |       ipAddress: real client IP
    |       deviceInfo: User-Agent string
    |       deviceFingerprint: hash from DeviceFingerprintUtil
    |       location: "Mumbai, India" (from GeoLocationService)
    |       isActive: true
    |       lastAccessedAt: now
    |       expiresAt: now + 7 days
    |
    |-- userSessionRepository.save(session) → MYSQL INSERT into user_sessions
```

---

## The Critical Role of Sessions in JWT Validation

The `JwtAuthenticationFilter` runs on **every single protected request**.
This is where the session check happens:

```java
// JwtAuthenticationFilter.java - line 44
if (jwtTokenProvider.validateToken(jwt) && sessionService.isSessionActive(jwt)) {
    // Only if BOTH pass → user is authenticated
    username = jwtTokenProvider.getUsernameFromToken(jwt);
    ...
    SecurityContextHolder.getContext().setAuthentication(authToken);
}

// SessionService.isSessionActive(token)
public boolean isSessionActive(String token) {
    return userSessionRepository
        .findFirstByTokenOrderByIdDesc(token)
        .map(UserSession::isActive)
        .orElse(false);  // if session not found → treat as inactive
}
```

This creates two independent revocation checks:

```
Check 1 - JWT Signature (cryptographic, no DB):
    Is the token signed with our secret? Is it within expiry date?
    → Prevents forged or tampered tokens

Check 2 - DB Session (real-time):
    Is this specific token still marked isActive = true in the DB?
    → Enables immediate revocation even when JWT is still valid
```

---

## Database Table: user_sessions

| Column | Type | Nullable | Description |
| :--- | :--- | :--- | :--- |
| `id` | BIGINT PK | NO | Auto-generated primary key |
| `user_id` | BIGINT FK | NO | References `users.id` |
| `token` | VARCHAR(2048) | NO | The full JWT access token string |
| `ip_address` | VARCHAR | YES | Real client IP address |
| `device_info` | VARCHAR | YES | User-Agent header value |
| `device_fingerprint` | VARCHAR(255) | YES | Hashed device characteristics |
| `location` | VARCHAR | YES | Human-readable location from IP |
| `is_active` | BOOLEAN | NO | FALSE = terminated/logged out |
| `created_at` | DATETIME | NO | When session was first created (auto) |
| `last_accessed_at` | DATETIME | YES | Updated on session extend |
| `expires_at` | DATETIME | YES | Created+7d, or extended+7d |

---

## Dependencies Summary

| Dependency | Used For |
| :--- | :--- |
| `spring-data-jpa` | All database reads and writes via `UserSessionRepository` |
| `jjwt` (io.jsonwebtoken) | Parsing the JWT to find the session token in DB |
| `Lombok` | `@Builder`, `@Data`, `@RequiredArgsConstructor`, `@Slf4j` |
| `Spring Security` | `@AuthenticationPrincipal` to inject current user into controller |
| `jakarta.persistence` | `@Entity`, `@Table`, `@Column` on `UserSession` |

---

## Security Design Notes

| Decision | Why |
| :--- | :--- |
| Sessions soft-deleted (`isActive=false`) not hard-deleted | Preserves login history for audit/forensics |
| `findFirstByTokenOrderByIdDesc` instead of `findByToken` | Handles edge case of duplicate token rows gracefully |
| Ownership check before terminate — throws "not found" not "forbidden" | Prevents leaking info about whether session ID exists for other users |
| `saveAll()` used for bulk terminate | Single batch SQL UPDATE instead of N individual updates |
| Token stored in `user_sessions` (not Redis) | Persistence across server restarts; allows long-term session history |

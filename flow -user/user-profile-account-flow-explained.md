# User Profile & Account Feature — Complete Flow Document
> Part of the `user-service` deep-dive series.

This document explains the full end-to-end flow of every User Profile & Account feature,
from the frontend HTTP request all the way through to the database — including every file
involved, every dependency used, and ASCII diagrams for each flow.

---

## Files Involved

| File | Package | Role |
| :--- | :--- | :--- |
| `UserController.java` | `controller` | Entry point — receives all `/api/users/**` HTTP requests |
| `UserService.java` | `service/user` | Core profile logic (get, update, change password) |
| `AccountDeletionService.java` | `service/user` | Handles account deletion scheduling and cancellation |
| `UserSettingsService.java` | `service/user` | Manages per-user settings (theme, language, auto-logout) |
| `SecurityQuestionService.java` | `service/auth` | Saves and verifies security Q&A pairs |
| `AccountDeletionScheduler.java` | `scheduler` | Background job that permanently deletes expired accounts |
| `EncryptionUtil.java` | `util` | Derives AES key from master password hash + salt |
| `VaultClient.java` | `client` | Feign client — calls vault-service to re-encrypt vault |
| `SecurityClient.java` | `client` | Feign client — sends security alerts on sensitive actions |
| `UserRepository.java` | `repository` | DB queries for `users` table |
| `UserSettingsRepository.java` | `repository` | DB queries for `user_settings` table |
| `SecurityConfig.java` | `config` | Marks all `/api/users/**` routes as protected (JWT required) |
| `JwtAuthenticationFilter.java` | `security` | Validates JWT before any request reaches the controller |

---

## Feature 1 — Get User Profile

### What it does
Returns the current user's profile data (email, username, name, phone, 2FA status).

### ASCII Flow Diagram

```
FRONTEND
    |
    | GET /api/users/profile
    | Headers: Authorization: Bearer <token>
    |
    v
[JwtAuthenticationFilter.java]
    | -- Validates JWT signature and session
    | -- Sets username in SecurityContext
    v
[UserController.java]
    | -- getCurrentUsername() from SecurityContextHolder
    v
[UserService.getUserProfile(username)]
    |-- userRepository.findByUsernameOrThrow(username) --> MYSQL SELECT
    |-- Builds UserResponse { id, email, username, name, phone, is2faEnabled, createdAt }
    v
[RESPONSE 200 OK]
    | Body: {
    |   id: 1,
    |   email: "john@example.com",
    |   username: "john",
    |   name: "John Doe",
    |   is2faEnabled: true,
    |   createdAt: "2024-01-01"
    | }
    v
FRONTEND
```

---

## Feature 2 — Update User Profile

### What it does
Allows the user to update their display name, phone number, and email address. Email uniqueness
is validated before the update is saved.

### ASCII Flow Diagram

```
FRONTEND
    |
    | PUT /api/users/profile
    | Headers: Authorization: Bearer <token>
    | Body: { name: "John Smith", phoneNumber: "+1234567890", email: "new@email.com" }
    |
    v
[JwtAuthenticationFilter.java] --> [UserController.java]
    v
[UserService.updateProfile(username, request)]
    |-- 1. userRepository.findByUsernameOrThrow(username) --> MYSQL SELECT
    |
    |-- 2. IF name provided --> user.setName()
    |-- 3. IF phoneNumber provided --> user.setPhoneNumber()
    |-- 4. IF email provided AND different from current email:
    |         userRepository.existsByEmail(newEmail) --> MYSQL SELECT
    |         If already used --> throw "Email already in use"
    |         Otherwise --> user.setEmail(newEmail)
    |
    |-- 5. userRepository.save(user) --> MYSQL UPDATE users table
    |
    v
[RESPONSE 200 OK]
    | Body: updated UserResponse
    v
FRONTEND
```

> **Note:** Only the fields provided in the request body are updated.
> If `name` is null in the request, the existing name is not touched.

---

## Feature 3 — Change Master Password (Most Complex)

### What it does
This is the most critical profile operation. Changing the master password requires:
1. Verifying the old password
2. Deriving the **old AES key** (used to encrypt vault data)
3. Hashing the new password
4. Deriving the **new AES key**
5. Sending both keys to `vault-service` so every entry can be **re-encrypted** with the new key
6. Firing a security alert

Without the re-encryption step, the vault data would be permanently unreadable after a password change.

### ASCII Flow Diagram

```
FRONTEND
    |
    | PUT /api/users/change-password
    | Headers: Authorization: Bearer <token>
    | Body: { oldPassword: "OldPass123!", newPassword: "NewPass456!" }
    |
    v
[JwtAuthenticationFilter.java] --> [UserController.java]
    v
[UserService.changeMasterPassword(username, request)]
    |
    |-- STEP 1: VERIFY OLD PASSWORD
    |   userRepository.findByUsernameOrThrow() --> MYSQL SELECT
    |   passwordEncoder.matches(oldPassword, user.masterPasswordHash)
    |   If WRONG --> throw "Invalid old password"
    |
    |-- STEP 2: DERIVE OLD ENCRYPTION KEY
    |   encryptionUtil.deriveKey(user.masterPasswordHash, user.salt)
    |       --> PBKDF2WithHmacSHA256(hash + salt) --> AES-256 SecretKey (oldKey)
    |
    |-- STEP 3: HASH NEW PASSWORD
    |   passwordEncoder.encode(newPassword) --> BCrypt hash (newPasswordHash)
    |
    |-- STEP 4: DERIVE NEW ENCRYPTION KEY
    |   encryptionUtil.deriveKey(newPasswordHash, user.salt)
    |       --> PBKDF2WithHmacSHA256(newHash + salt) --> AES-256 SecretKey (newKey)
    |
    |-- STEP 5: UPDATE PASSWORD HASH (NOT SAVED YET)
    |   user.setMasterPasswordHash(newPasswordHash)
    |
    |-- STEP 6: RE-ENCRYPT ENTIRE VAULT
    |   vaultClient.reencryptVault(userId, { oldKey, newKey })
    |       --> HTTP POST to vault-service internal endpoint
    |       --> vault-service: for each VaultEntry:
    |             decrypt(encryptedPassword, oldKey) --> plainText
    |             encrypt(plainText, newKey)         --> newCipherText
    |             update VaultEntry
    |
    |-- STEP 7: SAVE NEW HASH TO DB
    |   userRepository.save(user) --> MYSQL UPDATE users SET master_password_hash = ?
    |
    |-- STEP 8: FIRE SECURITY ALERT
    |   securityClient.createAlert(username, "PASSWORD_CHANGED", "HIGH")
    |       --> HTTP call to security-service
    |       --> Security team is notified immediately
    |
    v
[RESPONSE 200 OK]
    | Body: { message: "Password changed successfully" }
    v
FRONTEND
    | -- Typically prompts user to log in again
```

### Why the order matters

```
  IF we save the new hash BEFORE re-encrypting the vault:
      --> Vault still encrypted with oldKey
      --> But user.masterPasswordHash is now the new hash
      --> encryptionUtil.deriveKey(newHash, salt) gives DIFFERENT key
      --> Vault becomes permanently inaccessible!

  CORRECT ORDER:
      1. Derive oldKey  (before changing anything)
      2. Derive newKey  (using new hash but DO NOT save yet)
      3. Re-encrypt vault with both keys
      4. Save new hash to DB
```

---

## Feature 4 — Get Dashboard Stats

### What it does
Returns aggregate counts for the user's vault (total entries, favorites, trash count) by
calling the vault-service internally.

### ASCII Flow Diagram

```
FRONTEND
    |
    | GET /api/users/dashboard
    | Headers: Authorization: Bearer <token>
    |
    v
[JwtAuthenticationFilter.java] --> [UserController.java]
    |
    |-- getCurrentUsername() from SecurityContext
    |-- userRepository.findByUsername() --> MYSQL SELECT (to get userId)
    |
    |-- vaultClient.getDashboardStats(userId)
    |       --> HTTP GET to vault-service: /api/internal/vault/stats/{userId}
    |       --> Returns: { vaultCount, favoriteCount, trashCount }
    |
    v
[RESPONSE 200 OK]
    | Body: {
    |   totalVaultEntries: 42,
    |   totalFavorites: 7,
    |   trashCount: 3
    | }
    |
    | NOTE: If vault-service is unavailable, returns { 0, 0, 0 }
    |       (graceful degradation — dashboard still loads)
    v
FRONTEND
```

---

## Feature 5 — Get & Update Security Questions

### What it does
Users can view the text of their 3 security questions and update them (requires master password
confirmation). These questions are critical for account recovery.

### ASCII Flow Diagram

```
FRONTEND
    |
    | GET /api/users/security-questions
    | Headers: Authorization: Bearer <token>
    |
    v
[UserController.java]
    |-- userRepository.findByUsername() --> MYSQL SELECT
    |-- securityQuestionService.getSecurityQuestions(user)
    |       --> MYSQL SELECT from security_questions WHERE user_id = ?
    |       --> Returns list of { questionText, answerHash }
    |-- Maps to DTO: strips answer → returns only { questionText, "" }
    v
[RESPONSE 200 OK]
    | Body: [
    |   { question: "Your childhood pet?", answer: "" },
    |   { question: "Your mother's maiden name?", answer: "" },
    |   { question: "City you were born in?", answer: "" }
    | ]

---

FRONTEND
    |
    | PUT /api/users/security-questions
    | Body: { masterPassword, securityQuestions: [{question, answer}] }
    |
    v
[UserController.java]
    |-- userRepository.findByUsername() --> MYSQL SELECT
    |-- passwordEncoder.matches(masterPassword, user.hash) --> BCrypt verify
    |   If WRONG --> throw 401
    |
    |-- securityQuestionService.saveSecurityQuestions(user, questions)
    |       --> BCrypt.encode(answer) for each question
    |       --> DELETE old questions --> MYSQL DELETE
    |       --> INSERT new questions --> MYSQL INSERT
    v
[RESPONSE 200 OK]
```

---

## Feature 6 — Toggle Read-Only Mode

### What it does
Puts the vault into read-only mode — the user can **view** all vault entries but cannot
**create**, **update**, or **delete** anything. Useful when sharing a device temporarily.

### ASCII Flow Diagram

```
FRONTEND
    |
    | PUT /api/users/read-only-mode
    | Body: { readOnlyMode: true }
    |
    v
[UserController.java]
    v
[UserSettingsService.toggleReadOnlyMode(username, isReadOnly)]
    |-- userRepository.findByUsernameOrThrow() --> MYSQL SELECT
    |-- userSettingsRepository.findByUserId()  --> MYSQL SELECT
    |   If no settings exist yet --> createDefaultSettings()
    |       --> { theme: "SYSTEM", language: "en-US", autoLogout: 15min, readOnly: false }
    |       --> userSettingsRepository.save() --> MYSQL INSERT
    |-- settings.setReadOnlyMode(true/false)
    |-- userSettingsRepository.save() --> MYSQL UPDATE
    v
[RESPONSE 200 OK]
    | Body: { theme, language, autoLogoutMinutes, readOnlyMode: true }
    |
    |
HOW IT IS ENFORCED (vault-service side):
    |
    | When user calls any vault write operation (create/update/delete):
    | vaultClient.getUserDetailsByUsername(username)
    |   --> Returns UserVaultDetails including readOnlyMode flag
    | vault-service checks: if (user.isReadOnlyMode()) throw IllegalStateException
```

---

## Feature 7 — User Settings (Theme, Language, Auto-Logout)

### What it does
Stores user preferences like UI theme, display language, and auto-logout timeout.
A default settings record is auto-created on first access if none exists.

### ASCII Flow Diagram

```
FRONTEND
    |
    | GET /api/settings
    | Headers: Authorization: Bearer <token>
    |
    v
[UserSettingsController.java] --> [UserSettingsService.getSettings()]
    |-- userRepository.findByUsernameOrThrow() --> MYSQL SELECT
    |-- userSettingsRepository.findByUserId()  --> MYSQL SELECT
    |   If NOT FOUND --> createDefaultSettings(user)
    |       Defaults: theme="SYSTEM", language="en-US", autoLogoutMinutes=15
    |       --> userSettingsRepository.save() --> MYSQL INSERT
    v
[RESPONSE 200 OK]
    | Body: { theme: "DARK", language: "en-US", autoLogoutMinutes: 30, readOnlyMode: false }

---

FRONTEND
    |
    | PUT /api/settings
    | Body: { theme: "DARK", autoLogoutMinutes: 30 }
    |
    v
[UserSettingsService.updateSettings()]
    |-- Load existing settings (or create defaults)
    |-- Apply only the non-null fields from the request
    |-- userSettingsRepository.save() --> MYSQL UPDATE
    v
[RESPONSE 200 OK with updated settings]
```

---

## Feature 8 — Schedule Account Deletion

### What it does
The user requests to delete their account. Instead of immediate deletion, it is **scheduled**
for 30 days later — giving the user time to cancel if they change their mind.

### ASCII Flow Diagram

```
FRONTEND
    |
    | DELETE /api/users/account
    | Headers: Authorization: Bearer <token>
    | Body: { masterPassword: "MyPass123!" }
    |
    v
[JwtAuthenticationFilter.java] --> [UserController.java]
    v
[AccountDeletionService.scheduleAccountDeletion(username, request)]
    |
    |-- 1. userRepository.findByUsernameOrThrow() --> MYSQL SELECT
    |-- 2. passwordEncoder.matches(masterPassword, user.hash) --> BCrypt verify
    |   If WRONG --> throw "Invalid master password"
    |
    |-- 3. user.setDeletionRequestedAt(now)
    |   user.setDeletionScheduledAt(now + 30 days)
    |   userRepository.save() --> MYSQL UPDATE
    |       SET deletion_requested_at = ?, deletion_scheduled_at = ?
    |
    |-- 4. securityClient.createAlert(username, "ACCOUNT_DELETION_SCHEDULED", "CRITICAL")
    |       --> HTTP call to security-service
    |       --> User receives: "Your account will be deleted in 30 days..."
    v
[RESPONSE 200 OK]
    | Body: { message: "Account scheduled for deletion in 30 days..." }
    v
FRONTEND
    | -- Shows countdown timer and "Cancel Deletion" button
```

---

## Feature 9 — Cancel Account Deletion

### ASCII Flow Diagram

```
FRONTEND
    |
    | POST /api/users/account/cancel-deletion
    | Headers: Authorization: Bearer <token>
    |
    v
[AccountDeletionService.cancelAccountDeletion(username)]
    |-- 1. userRepository.findByUsernameOrThrow() --> MYSQL SELECT
    |-- 2. Check: deletionScheduledAt != null (is it actually scheduled?)
    |   If NOT scheduled --> throw "Account is not scheduled for deletion"
    |
    |-- 3. user.setDeletionRequestedAt(null)
    |   user.setDeletionScheduledAt(null)
    |   userRepository.save() --> MYSQL UPDATE (clears the deletion timestamps)
    |
    |-- 4. securityClient.createAlert(username, "ACCOUNT_DELETION_CANCELLED", "LOW")
    v
[RESPONSE 200 OK]
    | Body: { message: "Account deletion cancelled." }
```

---

## Feature 10 — Background: Permanent Account Deletion

### What it does
Runs **automatically every night at midnight**. Finds all accounts where
`deletionScheduledAt` is in the past and permanently removes them from the database.

### ASCII Flow Diagram

```
[SYSTEM CLOCK] - midnight every day
    |
    | Triggers: @Scheduled(cron = "0 0 0 * * ?")
    v
[AccountDeletionScheduler.java]
    |-- userRepository.findByDeletionScheduledAtBefore(LocalDateTime.now())
    |       --> MYSQL: SELECT * FROM users WHERE deletion_scheduled_at < NOW()
    |
    |-- For each expired user:
    |       userRepository.delete(user)
    |       --> MYSQL: DELETE FROM users WHERE id = ?
    |       --> Cascades: deletes related sessions, login attempts, OTPs,
    |                     security questions, 2FA records (via @OnDelete)
    v
[LOG] "Deleted N expired accounts"
```

> **Note:** If a user cancelled their deletion before midnight, `deletionScheduledAt`
> is null and they will **not** appear in the query — they are safe.

---

---

## How the API Gateway Fits Into Every Request

Every request — profile, settings, password change — passes through the **API Gateway** before it ever reaches the user-service. Here is exactly what happens inside the gateway on each request.

### Files in the API Gateway

| File | Role |
| :--- | :--- |
| `JwtAuthFilter.java` | First-line JWT check — rejects invalid tokens before they reach user-service |
| `RateLimitGlobalFilter.java` | Rate limiter — limits requests per IP to prevent brute force and abuse |

### API Gateway Flow (applies to every request)

```
BROWSER / FRONTEND (Angular - http://localhost:4200)
    |
    | Any request e.g. GET /api/users/profile
    | Headers: Authorization: Bearer eyJ...
    |
    v
[API GATEWAY - port 8080]  <-- The single entry point for the entire system
    |
    |-- STEP 1: CORS PREFLIGHT CHECK (browser does this automatically)
    |   If method == OPTIONS:
    |       JwtAuthFilter skips JWT validation
    |       Gateway returns 200 with CORS headers immediately
    |           Access-Control-Allow-Origin: http://localhost:4200
    |           Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS, PATCH
    |           Access-Control-Allow-Headers: *
    |           Access-Control-Max-Age: 3600
    |
    |-- STEP 2: RATE LIMIT CHECK (RateLimitGlobalFilter.java)
    |   Extract client IP from request headers
    |   Check Guava cache: how many requests from this IP in last 60 seconds?
    |   /api/auth/** endpoints --> max 10 req/min (strict — prevents brute force)
    |   All other endpoints   --> max 100 req/min (general limit)
    |   If over limit --> return 429 Too Many Requests (request STOPS here)
    |
    |-- STEP 3: JWT PRE-VALIDATION (JwtAuthFilter.java)
    |   If path starts with /api/auth/ --> SKIP (public routes, no token needed)
    |   If no Authorization header   --> return 401 (request STOPS here)
    |   If header doesn't start with "Bearer " --> return 401
    |   Parse JWT: Jwts.parser().verifyWith(secret).parseSignedClaims(token)
    |       If expired, malformed, or bad signature --> return 401 (STOPS here)
    |   Extract: username = claims.getSubject()
    |   Extract: isDuress = claims.get("duress", Boolean.class)
    |
    |-- STEP 4: MUTATE AND FORWARD
    |   Add headers to the request before forwarding:
    |       X-User-Name: "john"          (so services don't need to re-parse the JWT)
    |       X-Is-Duress: "false"         (vault-service reads this to show decoy data)
    |   Route to correct service based on path:
    |       /api/users/**   --> user-service:8081
    |       /api/vault/**   --> vault-service:8082
    |       /api/auth/**    --> user-service:8081
    |       /api/ai/**      --> ai-service:8085
    |
    v
[USER-SERVICE - port 8081]
    | -- Receives already-verified request
    | -- Still runs its own JwtAuthenticationFilter (defence in depth)
```

---

## CORS — How It Actually Works

CORS (Cross-Origin Resource Sharing) is what allows your Angular frontend (running on `localhost:4200`) to call your backend (running on `localhost:8080`). Browsers block cross-origin requests by default unless the server explicitly allows them.

### The Two-Level CORS Setup

Your application handles CORS at **two layers**:

```
LAYER 1: API Gateway (Spring Cloud Gateway)
    The Gateway is the first to receive CORS preflight requests from the browser.
    It is configured via application.yml on the config-server with allowed origins.
    OPTIONS requests are passed through by JwtAuthFilter without JWT validation.

LAYER 2: user-service CorsConfig.java (WebMvcConfigurer)
    Even if a request gets through the gateway, the user-service has its own
    CORS rules as a safety net — this handles direct calls or misconfigured gateways.
```

### CorsConfig.java — What It Does

```java
// CorsConfig.java in user-service config package
@Configuration
public class CorsConfig implements WebMvcConfigurer {

    @Value("${cors.allowed.origins:http://localhost:4200,http://localhost,http://frontend}")
    private String allowedOrigins;   // <-- Read from config-server

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")      // applies to all /api routes
            .allowedOrigins(origins)        // e.g. http://localhost:4200
            .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS", "PATCH")
            .allowedHeaders("*")            // any headers allowed
            .exposedHeaders("Authorization") // frontend can READ this response header
            .allowCredentials(true)          // allows cookies to be sent
            .maxAge(3600);                   // browser caches preflight for 1 hour
    }
}
```

### How SecurityConfig.java Cooperates with CORS

```java
// SecurityConfig.java
http
    .cors(Customizer.withDefaults())   // <-- tells Spring Security to USE CorsConfig
    .csrf(AbstractHttpConfigurer::disable)  // CSRF disabled (JWT is stateless)
    .sessionManagement(session ->
        session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
    // PUBLIC routes (no JWT needed):
    .authorizeHttpRequests(auth -> auth
        .requestMatchers(
            "/api/auth/**",        // login, register, forgot password
            "/api/users/internal/**", // internal service-to-service calls
            "/swagger-ui/**", "/v3/api-docs/**",  // Swagger docs
            "/actuator/health"     // health checks
        ).permitAll()
        .anyRequest().authenticated()  // everything else needs JWT
    )
    .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);
```

### Full CORS Flow — Browser Preflight

```
BROWSER (first request to a new endpoint or after 1hr cache expires)
    |
    | OPTIONS /api/users/profile
    | Origin: http://localhost:4200
    | Access-Control-Request-Method: GET
    | Access-Control-Request-Headers: authorization
    |
    v
[API GATEWAY - JwtAuthFilter]
    | -- Method is OPTIONS --> SKIP JWT check
    | -- Forward to user-service
    v
[USER-SERVICE - Spring MVC CorsConfig]
    | -- Origin "http://localhost:4200" matches allowed origins? YES
    | -- Method "GET" is in allowed methods? YES
    | -- Header "authorization" allowed? YES (allowedHeaders = "*")
    v
[RESPONSE 200 OK - no body]
    | Access-Control-Allow-Origin: http://localhost:4200
    | Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS, PATCH
    | Access-Control-Allow-Headers: *
    | Access-Control-Max-Age: 3600  <-- browser caches this for 1 hour
    v
BROWSER
    | -- Preflight passed, now sends the REAL request:
    | GET /api/users/profile
    | Authorization: Bearer eyJ...
    | Origin: http://localhost:4200
    |
    v
[API GATEWAY] --> [USER-SERVICE] --> [RESPONSE 200 OK with profile data]
```

### What happens if CORS fails

```
BROWSER
    | GET /api/users/profile
    | Origin: http://evil-site.com   <-- NOT in allowed origins
    v
[USER-SERVICE CorsConfig]
    | -- "http://evil-site.com" NOT in allowedOrigins
    | -- Returns response WITHOUT Access-Control-Allow-Origin header
    v
BROWSER
    | -- No CORS header = browser BLOCKS the response
    | -- Console error: "CORS policy: No 'Access-Control-Allow-Origin' header"
    | -- The data never reaches the frontend JavaScript (security feature!)
```

---

## Updated Files Table (with Gateway)

| File | Service | Role |
| :--- | :--- | :--- |
| `JwtAuthFilter.java` | `api-gateway` | First-line JWT validation, adds X-User-Name and X-Is-Duress headers |
| `RateLimitGlobalFilter.java` | `api-gateway` | IP-based rate limiting — 10 req/min auth, 100 req/min general |
| `CorsConfig.java` | `user-service` | Defines allowed origins, methods, headers for CORS |
| `SecurityConfig.java` | `user-service` | Integrates CORS + JWT filter, defines public vs protected routes |
| `JwtAuthenticationFilter.java` | `user-service` | Second JWT check inside user-service (defence in depth) |

---

## Dependencies Summary

| Dependency | Used For |
| :--- | :--- |
| `spring-data-jpa` | All database reads & writes |
| `spring-security-crypto` | BCryptPasswordEncoder for password verification |
| `javax.crypto` | AES key derivation via `EncryptionUtil` |
| `spring-cloud-openfeign` | `VaultClient` (re-encrypt), `SecurityClient` (alerts) |
| `Jakarta Validation` | `@Valid` on `UpdateProfileRequest`, `ChangePasswordRequest` |
| `Lombok` | `@RequiredArgsConstructor`, `@Builder`, `@Slf4j` |
| `spring-scheduling` | `@Scheduled` cron in `AccountDeletionScheduler` |

---

## Database Tables Used

| Table | Written By | Read By |
| :--- | :--- | :--- |
| `users` | `UserService`, `AccountDeletionService`, `AccountDeletionScheduler` | All services |
| `user_settings` | `UserSettingsService` | `UserSettingsService`, `VaultClient` response |
| `security_questions` | `SecurityQuestionService` | `SecurityQuestionService` |

# Internal / Service-to-Service API — Complete Flow Document
> Part of the `user-service` deep-dive series.

This document explains the full end-to-end flow of every **internal** API endpoint
in the user-service — endpoints that are **not called by the frontend** but instead
by other microservices (`vault-service`, `security-service`) to fetch user data.

---

## What is Service-to-Service Communication?

In a microservice architecture, services often need data from each other.
Instead of every service having its own copy of user data, they call the
`user-service` directly using **Feign clients** (HTTP clients that look like
regular Java method calls).

```
MICROSERVICE ARCHITECTURE:

    vault-service          security-service
         |                       |
         | Feign HTTP call        | Feign HTTP call
         |                       |
         v                       v
    [user-service internal endpoints]
         |
         v
    [UserRepository] --> MYSQL
```

These endpoints are **never called by the browser/frontend**. They are only
called by other services running inside Docker or Kubernetes — they are
protected at the infrastructure level, not by JWT.

---

## Files Involved

### In `user-service` (the server side)

| File | Package | Role |
| :--- | :--- | :--- |
| `InternalUserController.java` | `controller` | Handles all internal HTTP requests from other services |
| `UserRepository.java` | `repository` | Fetches user data from MySQL |
| `TwoFactorAuthRepository.java` | `repository` | Fetches TOTP secret key for vault-service when 2FA is enabled |
| `SecurityConfig.java` | `config` | Marks internal routes as `permitAll()` — no JWT needed |

### In `vault-service` (Feign client caller)

| File | Package | Role |
| :--- | :--- | :--- |
| `UserClient.java` | `client` | Feign interface — calls `/api/users/internal/**` |
| `UserVaultDetails` | (inner class in `UserClient`) | Maps the JSON response from user-service |

### In `security-service` (Feign client caller)

| File | Package | Role |
| :--- | :--- | :--- |
| `UserClient.java` | `client` | Feign interface — calls `/internal/users/**` |
| `UserVaultDetails` | (inner class in `UserClient`) | Maps the minimal JSON response |

---

## Why Are There Two Different URL Patterns?

```
vault-service calls:      /api/users/internal/by-username/{username}
                          /api/users/internal/by-id/{id}

security-service calls:   /internal/users/{username}
                          /internal/users/email/{email}
```

They use **different URL prefixes** because they evolved from different teams /
service templates. The business logic is the same but:
- `vault-service` uses the `/api/users/internal/**` pattern
- `security-service` uses the `/internal/users/**` pattern

Both sets of routes are whitelisted in `SecurityConfig.java`:

```java
// SecurityConfig.java
.requestMatchers(
    "/api/users/internal/**",   // for vault-service
    "/internal/users/**"        // for security-service
).permitAll()
```

---

## The Two Response Payloads

The `InternalUserController` returns **different data** depending on which service is calling
and what it needs. This is the principle of data minimisation — each service only receives
what it actually needs.

### `toVaultMap()` — Full details for `vault-service`

```java
{
    "id": 1,
    "username": "john",
    "masterPasswordHash": "$2a$10$...",   // BCrypt hash — vault uses this to derive AES key
    "salt": "abc123xyz...",               // Salt for AES key derivation
    "is2faEnabled": true,
    "twoFactorSecret": "JBSWY3DP...",     // TOTP secret (only if 2FA is enabled)
    "readOnlyMode": false                 // Whether vault writes should be blocked
}
```

> **Why does vault-service need `masterPasswordHash` and `salt`?**
> The vault encryption key is **derived from** the master password hash + salt using
> PBKDF2. The vault-service needs these to encrypt/decrypt vault entries.

### `toSecurityMap()` — Minimal details for `security-service`

```java
{
    "id": 1,
    "username": "john",
    "email": "john@example.com",
    "createdAt": "2024-01-01T10:00:00"
}
```

> **Why does security-service get less data?**
> It only needs to link audit logs, alerts, and security events to a specific user.
> It does NOT need the password hash or encryption keys.

---

## Endpoint 1 — vault-service: Get User by Username

### Who calls it?
`vault-service` → `UserClient.getUserDetailsByUsername(username)`

### When is it called?
Every time a user performs a vault operation (view, create, update, delete).
The vault-service needs to: verify encryption key, check read-only mode, check 2FA status.

### ASCII Flow Diagram

```
USER (browser)
    |
    | GET /api/vault/entries
    | Authorization: Bearer eyJ...
    |
    v
[API GATEWAY] --> [vault-service VaultController]
    |
    | vault-service needs to know the user's encryption key materials
    |
    v
[vault-service UserClient.java] (Feign client)
    |
    | GET http://user-service/api/users/internal/by-username/john
    | (no Authorization header — internal call, no JWT)
    |
    v
[user-service InternalUserController.getUserByUsername("john")]
    |
    |-- userRepository.findByUsername("john") --> MYSQL SELECT FROM users
    |   If not found --> throw ResourceNotFoundException (vault-service gets 404)
    |
    |-- toVaultMap(user):
    |       Builds HashMap:
    |           id, username,
    |           masterPasswordHash (BCrypt hash),
    |           salt,
    |           is2faEnabled,
    |           readOnlyMode (hardcoded false here — TODO: pull from user_settings)
    |
    |       IF user.is2faEnabled() == true:
    |           twoFactorAuthRepository.findByUser(user) --> MYSQL SELECT
    |               .ifPresent(tfa -> map.put("twoFactorSecret", tfa.getSecretKey()))
    |           (so vault can verify OTP for unlocking sensitive entries)
    |
    v
[RESPONSE 200 OK → back to vault-service]
    | Body: {
    |   "id": 1,
    |   "username": "john",
    |   "masterPasswordHash": "$2a$10$...",
    |   "salt": "abc123...",
    |   "is2faEnabled": true,
    |   "twoFactorSecret": "JBSWY3DP...",
    |   "readOnlyMode": false
    | }
    v
[vault-service] continues processing vault entries
    | encryptionUtil.deriveKey(masterPasswordHash, salt) --> AES-256 key
    | decrypt(encryptedPassword, aesKey) --> plaintext password
```

---

## Endpoint 2 — vault-service: Get User by ID

### Who calls it?
`vault-service` → `UserClient.getUserDetailsById(userId)`

### When is it called?
When vault-service has only the userId (e.g. from a vault entry's `user_id` column)
and needs the full user details to perform decryption.

### ASCII Flow Diagram

```
[vault-service]
    |
    | GET http://user-service/api/users/internal/by-id/1
    |
    v
[user-service InternalUserController.getUserById(1L)]
    |
    |-- userRepository.findById(1L) --> MYSQL SELECT FROM users WHERE id = 1
    |   If not found --> throw ResourceNotFoundException
    |
    |-- toVaultMap(user) (same as above — full encryption details)
    |       + twoFactorSecret if 2FA enabled
    |
    v
[RESPONSE 200 OK → same payload as endpoint 1]
```

---

## Endpoint 3 — security-service: Get User by Username

### Who calls it?
`security-service` → `UserClient.getUserDetailsByUsername(username)`

### When is it called?
When security-service needs to create a `SecurityAlert` or `AuditLog` record
linked to a specific user (e.g. "password changed" event, "new login from IP" alert).

### ASCII Flow Diagram

```
[security-service SecurityAlertService]
    |
    | securityClient.createAlert(username, "PASSWORD_CHANGED", ...)
    | → security-service needs the user's ID to link the alert
    |
    v
[security-service UserClient.java] (Feign client)
    |
    | GET http://user-service/internal/users/john
    |
    v
[user-service InternalUserController.getUserByUsernameInternal("john")]
    |
    |-- userRepository.findByUsername("john") --> MYSQL SELECT
    |
    |-- toSecurityMap(user):
    |       { id, username, email, createdAt }
    |       (NO password hash, NO salt, NO 2FA secret)
    |
    v
[RESPONSE 200 OK → back to security-service]
    | Body: {
    |   "id": 1,
    |   "username": "john",
    |   "email": "john@example.com",
    |   "createdAt": "2024-01-01T10:00:00"
    | }
    v
[security-service] creates SecurityAlert with user.id as foreign key
```

---

## Endpoint 4 — security-service: Get User by Email

### Who calls it?
`security-service` → `UserClient.getUserDetailsByEmail(email)`

### When is it called?
When security-service receives an event keyed by email address rather than username
(e.g. alerts from email-based login flow).

### ASCII Flow Diagram

```
[security-service]
    |
    | GET http://user-service/internal/users/email/john@example.com
    |
    v
[user-service InternalUserController.getUserByEmailInternal("john@example.com")]
    |
    |-- userRepository.findByEmail("john@example.com") --> MYSQL SELECT
    |   If not found --> throw ResourceNotFoundException
    |
    |-- toSecurityMap(user) (same minimal payload)
    |
    v
[RESPONSE 200 OK] → { id, username, email, createdAt }
```

---

## How Security Works for Internal Endpoints

These endpoints have **no JWT token requirement** — but they are not exposed to the internet:

```
PUBLIC INTERNET
    |
    | BLOCKED by firewall / Docker network rules
    | Internal endpoints are only reachable from within
    | the Docker compose network (service-to-service only)
    |
    v
[API GATEWAY - port 8080]
    | External traffic ONLY flows through port 8080
    | Direct calls to user-service:8081 are NOT accessible from outside
    |
    v
[vault-service / security-service]
    | These run inside the same Docker network
    | They CAN reach user-service:8081 directly (by service name)
    |
    v
[user-service - InternalUserController]
    | SecurityConfig: .permitAll() for /api/users/internal/** and /internal/users/**
    | No JWT needed — network isolation is the security boundary
```

---

## Feign Client: How vault-service Calls user-service

```java
// vault-service: UserClient.java
@FeignClient(name = "user-service")  // Spring Cloud finds user-service via Eureka/config
public interface UserClient {

    @GetMapping("/api/users/internal/by-username/{username}")
    UserVaultDetails getUserDetailsByUsername(@PathVariable("username") String username);

    @GetMapping("/api/users/internal/by-id/{id}")
    UserVaultDetails getUserDetailsById(@PathVariable("id") Long id);
}
```

When vault-service calls `userClient.getUserDetailsByUsername("john")`:
1. Feign resolves `user-service` hostname via service discovery (Eureka / config-server)
2. Makes an HTTP GET to `http://user-service:8081/api/users/internal/by-username/john`
3. JSON response is automatically deserialized into `UserVaultDetails` class
4. vault-service uses the result as a regular Java object

---

## Data Flow Summary

```
                    vault-service
                         |
    +--------------------+--------------------+
    |                                         |
    | by-username/{username}          by-id/{id}
    |                                         |
    v                                         v
[InternalUserController]              [InternalUserController]
    |                                         |
    +--------------------+--------------------+
                         |
                         | toVaultMap(user)
                         | FULL PAYLOAD:
                         | id, username,
                         | masterPasswordHash, salt,
                         | is2faEnabled, twoFactorSecret,
                         | readOnlyMode
                         |
                         v
                    [vault-service uses data to
                     derive AES key, check 2FA,
                     check read-only mode]

                  security-service
                         |
    +--------------------+--------------------+
    |                                         |
    | by-username/{username}      by-email/{email}
    |                                         |
    v                                         v
[InternalUserController]              [InternalUserController]
    |                                         |
    +--------------------+--------------------+
                         |
                         | toSecurityMap(user)
                         | MINIMAL PAYLOAD:
                         | id, username, email, createdAt
                         |
                         v
                  [security-service links alerts
                   and audit logs to this user.id]
```

---

## Database Queries Used

| Endpoint | Repository Call | SQL |
| :--- | :--- | :--- |
| `by-username/{username}` | `findByUsername()` | `SELECT * FROM users WHERE username = ?` |
| `by-id/{id}` | `findById()` | `SELECT * FROM users WHERE id = ?` |
| `/internal/users/{username}` | `findByUsername()` | `SELECT * FROM users WHERE username = ?` |
| `/internal/users/email/{email}` | `findByEmail()` | `SELECT * FROM users WHERE email = ?` |
| (supplemental) | `twoFactorAuthRepository.findByUser()` | `SELECT * FROM two_factor_auth WHERE user_id = ?` |

---

## Dependencies Summary

| Dependency | Used For |
| :--- | :--- |
| `spring-data-jpa` | `UserRepository`, `TwoFactorAuthRepository` |
| `spring-cloud-openfeign` | Feign clients in vault-service and security-service |
| `spring-cloud-netflix-eureka` | Service discovery — resolves `user-service` hostname |
| `Lombok` | `@RequiredArgsConstructor` in controller |

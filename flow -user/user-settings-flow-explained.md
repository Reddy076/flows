# User Settings — Complete Flow Document
> Part of the `user-service` deep-dive series.

This document explains the full end-to-end flow of every User Settings feature,
from the frontend HTTP request all the way through to the database — including every
file involved, every dependency used, and ASCII diagrams for each flow.

---

## What are User Settings?

User Settings store per-account preferences that control how the application looks
and behaves for that specific user. There is exactly **one settings record per user**,
stored in the `user_settings` table.

```
Settings available:
    theme            → "LIGHT" | "DARK" | "SYSTEM"  (UI theme)
    language         → "en-US" | "fr-FR" | etc.      (display language)
    autoLogoutMinutes→ integer (e.g. 15)              (idle timeout before auto-logout)
    readOnlyMode     → true | false                   (blocks vault writes)

Default values (auto-created on first access):
    theme            = "SYSTEM"
    language         = "en-US"
    autoLogoutMinutes= 15
    readOnlyMode     = false
```

---

## Files Involved

| File | Package | Role |
| :--- | :--- | :--- |
| `UserSettingsController.java` | `controller` | Entry point for `/api/settings` |
| `UserController.java` | `controller` | Entry point for `/api/users/read-only-mode` |
| `UserSettingsService.java` | `service/user` | All settings logic — get, update, toggle read-only, create defaults |
| `UserSettings.java` | `model/user` | JPA entity for the `user_settings` table |
| `UserSettingsRepository.java` | `repository` | DB queries for `user_settings` |
| `UserRepository.java` | `repository` | Looks up user by username from SecurityContext |
| `JwtAuthFilter.java` | `api-gateway` | First-line JWT check |
| `RateLimitGlobalFilter.java` | `api-gateway` | Rate limiting (100 req/min) |
| `CorsConfig.java` | `config` | CORS rules for browser preflight |
| `SecurityConfig.java` | `config` | `/api/settings/**` is JWT-protected |
| `JwtAuthenticationFilter.java` | `security` | Second JWT + session check inside user-service |

---

## The UserSettings Entity — Database Structure

```java
@Entity
@Table(name = "user_settings")
public class UserSettings {
    Long id;                       // Primary key
    User user;                     // @OneToOne FK → users.id (unique per user)
    String theme = "SYSTEM";       // UI colour theme
    String language = "en-US";     // Display language
    Integer autoLogoutMinutes = 15; // Minutes of inactivity before auto-logout
    Boolean readOnlyMode = false;  // Prevents vault writes when true
    LocalDateTime createdAt;       // Auto-set on INSERT
    LocalDateTime updatedAt;       // Auto-set on UPDATE
}
```

> **Key design:** `@OneToOne` with `unique = true` on `user_id` column.
> This enforces that one user has exactly one settings row in the database.

---

## How Every Request Passes Through the Gateway + CORS

```
FRONTEND (Angular)
    |
    | GET or PUT /api/settings
    | Headers: Authorization: Bearer eyJ...
    | Origin: http://localhost:4200
    |
    v
[API GATEWAY - port 8080]
    |-- RateLimitGlobalFilter: 100 req/min per IP
    |-- JwtAuthFilter:
    |       /api/settings is NOT /api/auth/** → JWT IS required
    |       Validates signature → adds X-User-Name header
    |       Forwards to user-service:8081
    v
[USER-SERVICE - Spring Security]
    |-- CorsConfig: validates Origin header
    |-- JwtAuthenticationFilter:
    |       Validates JWT signature AND checks session is active in DB
    |       Sets SecurityContextHolder with authenticated username
    |-- SecurityConfig: .anyRequest().authenticated() → JWT required
    v
[UserSettingsController.java]
    | getCurrentUsername():
    |   SecurityContextHolder.getContext().getAuthentication().getName()
    |   → returns "john" (the logged-in username)
```

---

## Feature 1 — Get User Settings

### What it does
Returns the current user's settings. If no settings record exists yet
(e.g. on first login after registration), default settings are **automatically created**
and saved before returning.

### ASCII Flow Diagram

```
FRONTEND
    |
    | GET /api/settings
    | Headers: Authorization: Bearer <token>
    |
    v
[API GATEWAY] --> [JwtAuthenticationFilter] --> [UserSettingsController.getSettings()]
    |
    | getCurrentUsername() → "john" (from SecurityContext)
    |
    v
[UserSettingsService.getSettings("john")]
    |
    |-- STEP 1: FIND USER
    |   userRepository.findByUsernameOrThrow("john") --> MYSQL SELECT from users
    |
    |-- STEP 2: FIND SETTINGS
    |   userSettingsRepository.findByUserId(userId)  --> MYSQL SELECT from user_settings
    |
    |-- STEP 3: NO SETTINGS YET? AUTO-CREATE DEFAULTS
    |   .orElseGet(() -> createDefaultSettings(user))
    |       --> Builds UserSettings:
    |               user = current user
    |               theme = "SYSTEM"
    |               language = "en-US"
    |               autoLogoutMinutes = 15
    |               readOnlyMode = false
    |       --> userSettingsRepository.save(defaults) --> MYSQL INSERT into user_settings
    |
    |-- STEP 4: MAP TO RESPONSE
    |   UserSettingsResponse:
    |       { theme, language, autoLogoutMinutes, readOnlyMode }
    |   (createdAt, updatedAt, userId are NOT exposed to frontend)
    |
    v
[RESPONSE 200 OK]
    | Body: {
    |   "theme": "DARK",
    |   "language": "en-US",
    |   "autoLogoutMinutes": 30,
    |   "readOnlyMode": false
    | }
```

---

## Feature 2 — Update User Settings

### What it does
Allows the user to update any combination of their settings in a single request.
Only the fields that are **present and non-null** in the request body are updated —
any null fields are ignored (partial update / PATCH-style behaviour via PUT).

### ASCII Flow Diagram

```
FRONTEND
    |
    | PUT /api/settings
    | Headers: Authorization: Bearer <token>
    | Body: { "theme": "DARK", "autoLogoutMinutes": 30 }
    |       (language and readOnlyMode are NOT sent → they won't be changed)
    |
    v
[UserSettingsController.updateSettings(request)]
    |
    | getCurrentUsername() → "john"
    |
    v
[UserSettingsService.updateSettings("john", request)]
    |
    |-- STEP 1: FIND USER
    |   userRepository.findByUsernameOrThrow("john") --> MYSQL SELECT
    |
    |-- STEP 2: LOAD EXISTING SETTINGS (or create defaults)
    |   userSettingsRepository.findByUserId(userId)
    |   .orElseGet(() -> createDefaultSettings(user))
    |       (same auto-create logic as GET — ensures settings always exist)
    |
    |-- STEP 3: APPLY ONLY NON-NULL FIELDS
    |   if (request.getTheme() != null)
    |       settings.setTheme("DARK")          ← applied
    |   if (request.getLanguage() != null)
    |       settings.setLanguage(...)           ← skipped (null in request)
    |   if (request.getAutoLogoutMinutes() != null)
    |       settings.setAutoLogoutMinutes(30)  ← applied
    |   if (request.getReadOnlyMode() != null)
    |       settings.setReadOnlyMode(...)       ← skipped (null in request)
    |
    |-- STEP 4: SAVE
    |   userSettingsRepository.save(settings)
    |       --> MYSQL UPDATE user_settings SET theme='DARK', auto_logout_minutes=30
    |           WHERE user_id = ?
    |       @UpdateTimestamp auto-sets updated_at = NOW()
    |
    v
[RESPONSE 200 OK]
    | Body: {
    |   "theme": "DARK",            ← changed
    |   "language": "en-US",        ← unchanged (kept original)
    |   "autoLogoutMinutes": 30,    ← changed
    |   "readOnlyMode": false       ← unchanged
    | }
```

---

## Feature 3 — Toggle Read-Only Mode

### What it does
A special shortcut endpoint (`PUT /api/users/read-only-mode`) that flips the
`readOnlyMode` boolean on the user's settings. When enabled, the vault-service
rejects any create/update/delete operations on vault entries.

> **Note:** This endpoint lives in `UserController`, not `UserSettingsController`,
> but delegates entirely to `UserSettingsService`.

### ASCII Flow Diagram

```
FRONTEND
    | (user clicks "Enable Read-Only Mode" toggle in security settings)
    |
    | PUT /api/users/read-only-mode
    | Headers: Authorization: Bearer <token>
    | Body: { "readOnlyMode": true }
    |
    v
[UserController.toggleReadOnlyMode(request)]
    |
    | getCurrentUsername() → "john"
    |
    v
[UserSettingsService.toggleReadOnlyMode("john", true)]
    |
    |-- userRepository.findByUsernameOrThrow() --> MYSQL SELECT
    |-- userSettingsRepository.findByUserId()  --> MYSQL SELECT
    |   (or create defaults if not found)
    |-- settings.setReadOnlyMode(true)
    |-- userSettingsRepository.save() --> MYSQL UPDATE
    |       SET read_only_mode = true WHERE user_id = ?
    |
    v
[RESPONSE 200 OK]
    | Body: { theme, language, autoLogoutMinutes, readOnlyMode: true }

HOW READ-ONLY IS ENFORCED (vault-service side):
    |
    | When user calls: POST /api/vault/entries  (create a new vault entry)
    |
    v
[vault-service VaultController]
    |-- Feign call to user-service internal endpoint: GET /internal/users/{username}
    |       --> Returns UserDetails including readOnlyMode flag
    |
    |-- if (userDetails.isReadOnlyMode()) {
    |       throw new IllegalStateException("Vault is in read-only mode");
    |   }
    |       --> Returns 400 Bad Request to frontend
    |
    | The vault entry is never saved.
```

---

## Default Settings Auto-Creation Flow

This happens whenever any settings endpoint is called but no settings record exists yet
(e.g. the user has never visited the settings page before).

```
[UserSettingsService.createDefaultSettings(user)]  ← private helper method
    |
    |-- Builds UserSettings entity with defaults:
    |       user          = current User object
    |       theme         = "SYSTEM"   ← follows OS light/dark preference
    |       language      = "en-US"
    |       autoLogoutMinutes = 15     ← 15 minutes of inactivity
    |       readOnlyMode  = false
    |
    |-- userSettingsRepository.save(settings)
    |       --> MYSQL INSERT INTO user_settings
    |           (user_id, theme, language, auto_logout_minutes, read_only_mode)
    |           VALUES (?, 'SYSTEM', 'en-US', 15, false)
    |       @CreationTimestamp auto-sets created_at = NOW()
    |       @UpdateTimestamp  auto-sets updated_at = NOW()
    |
    v
    Returns the saved UserSettings object
    → Caller (getSettings / updateSettings) continues as if it existed all along
```

---

## Database Table: user_settings

| Column | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `id` | BIGINT PK | auto | Auto-generated primary key |
| `user_id` | BIGINT FK UNIQUE | — | References `users.id` — ONE per user |
| `theme` | VARCHAR(20) | `SYSTEM` | `LIGHT`, `DARK`, or `SYSTEM` |
| `language` | VARCHAR(10) | `en-US` | Language/locale code |
| `auto_logout_minutes` | INT | `15` | Inactivity timeout in minutes |
| `read_only_mode` | BOOLEAN | `false` | Blocks vault writes when `true` |
| `created_at` | DATETIME | auto | Set on INSERT by Hibernate |
| `updated_at` | DATETIME | auto | Updated on every SAVE by Hibernate |

---

## Full Picture — All Settings Flows

```
                        FRONTEND
                            |
          +-----------------+-----------------+
          |                                   |
    GET /api/settings             PUT /api/settings
    (fetch current prefs)         (update prefs)
          |                                   |
          v                                   v
    UserSettingsService                 UserSettingsService
    .getSettings()                      .updateSettings()
          |                                   |
          +---------- find user ----------+   |
          |           findByUserId()      |   |
          |                               v   v
          |                         Apply non-null fields
          |                         userSettingsRepository.save()
          |
    findByUserId()
          |
    Found? → mapToResponse()
    Not found? → createDefaultSettings() → save() → mapToResponse()
          |
          v
    RESPONSE 200 OK { theme, language, autoLogoutMinutes, readOnlyMode }
```

---

## Dependencies Summary

| Dependency | Used For |
| :--- | :--- |
| `spring-data-jpa` | All DB reads and writes via `UserSettingsRepository` |
| `Hibernate` | `@CreationTimestamp`, `@UpdateTimestamp` auto-fill timestamps |
| `Jakarta Persistence` | `@OneToOne`, `@Entity`, `@Table`, `@Column` |
| `Lombok` | `@Builder`, `@Data`, `@Builder.Default` for entity defaults |
| `Spring Security` | `SecurityContextHolder` to get current username |

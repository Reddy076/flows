# Notification Service — Internal & Infrastructure Flow Complete Document
> Part of the `notification-service` deep-dive series.

This document details the complete end-to-end flow, dependencies, and implementations of the internal communication, cross-cutting configurations, and enum-based Notification types within the `notification-service`.

---

## 1. Internal Notification Creation Flow (Service-to-Service)

### What it does
The `notification-service` is a passive consumer. It does not generate alerts on its own. Instead, it exposes a secure internal endpoint used by other microservices (like `vault-service`) to create an in-app alert when a system action occurs.

### Complete ASCII Flow Diagram

```text
[VAULT-SERVICE] (or other producing microservices)
    │
    │  // Somewhere in VaultService.java
    │  notificationClient.createNotification(
    │      username,
    │      "ACCOUNT_ACTIVITY",
    │      "New Vault Entry",
    │      "You just added a new login to your vault."
    │  );
    │
    ▼
[API GATEWAY]
    │  // Internal traffic typically bypasses or routes straight through the gateway
    │  // depending on network topology, but often Feign clients call across Eureka directly.
    ▼
NOTIFICATION-SERVICE (:8080+)
    └── [InternalNotificationController.java]
          @PostMapping("/api/notifications/internal/create")
          │
          ├── 1. Parses @RequestParam type as String ("ACCOUNT_ACTIVITY")
          ├── 2. Converts String to Enum:
          │      NotificationType.valueOf("ACCOUNT_ACTIVITY")
          │      (Catches IllegalArgumentException if invalid string passed)
          │
          └── 3. Passes to [NotificationService.createNotification()]
                   │
                   ├── userClient.getUserDetailsByUsername(username)
                   │     └── (Synchronous Feign call to user-service, see Flow #2)
                   │
                   ├── Creates new Notification Entity via Builder:
                   │     - userId: extracted from userClient response
                   │     - notificationType: ACCOUNT_ACTIVITY
                   │     - title: "New Vault Entry"
                   │     - message: "You just added a new login to your vault."
                   │     - isRead: false
                   │     - createdAt: LocalDateTime.now()
                   │
                   └── notificationRepository.save(notification)
                         └── MYSQL: INSERT INTO notifications (user_id, notification_type, ...)
    │
    ▼
[RESPONSE 200 OK] (Returns to Vault Service)
```

---

## 2. Infrastructure Flow: Fetching User Details via OpenFeign

### What it does
The `notification-service` only stores the `user_id` in its table to maintain normalization, but the frontend requests come with a `username` (via the API Gateway JWT injection). Therefore, the `notification-service` must synchronously ask the `user-service` to convert the `username` into a `user_id` before saving or retrieving data.

### Complete ASCII Flow Diagram

```text
NOTIFICATION-SERVICE
    │
    │  userClient.getUserDetailsByUsername(username)
    │
    ▼
[UserClient.java] (OpenFeign Interface)
    │  @FeignClient(name = "user-service")
    │  @GetMapping("/api/users/internal/username/{username}")
    │
    ├── 1. Looks up `user-service` IP address in Eureka Registry
    ├── 2. Constructs HTTP GET request
    └── 3. Dispatches synchronous request over the network
    │
    ▼
USER-SERVICE
    └── [InternalUserController.java]
          ├── userRepository.findByUsername(username)
          └── Returns: { id: 105, username: "Reddy", email: "..." }
    │
    ▼
[UserClient.java]
    │  Deserializes JSON back into `UserVaultDetails` DTO
    ▼
NOTIFICATION-SERVICE
    └── Uses `user.getId()` for JPA database operations
```

> **Performance Note:** Because this call is synchronous, the `notification-service` blocks until `user-service` replies. If `user-service` is down, notification requests will fail (throw 500 exceptions).

---

## 3. Infrastructure Configuration: OpenAPI & Swagger UI

### What it does
It provides a UI to document and test the REST controllers without needing an Angular frontend or Postman. Since the `/api/notifications/**` endpoints require a JWT, the Swagger UI must be configured to pass a `Bearer` token.

### Complete ASCII Flow Diagram

```text
DEVELOPER (Browser)
    │
    │  GET http://localhost:8080/swagger-ui.html
    │
    ▼
[OpenApiConfig.java]
    ├── 1. Defines `SecurityScheme` named "bearerAuth"
    │         - Type: HTTP
    │         - Scheme: "bearer"
    │         - Format: "JWT"
    ├── 2. Applies `SecurityRequirement("bearerAuth")` to all endpoints globally
    └── 3. Sets API Metadata (Title: "Notification Service API", Version: "1.0")
    │
    ▼
SWAGGER UI Renders
    ├── Shows "Authorize" 🔒 padlock button
    └── Developer pastes JWT Token -> Swagger injects generic "Authorization: Bearer <token>" into headers
```

---

## 4. Supported Notification Types (Detailed Explanation)

The `NotificationType` Enum acts as a strict contract defining exactly what events the system can broadcast. 

| Enum Value | Triggered By | Example Use Case |
| :--- | :--- | :--- |
| `PASSWORD_EXPIRY` | Scheduled CRON job (user-service) | "Your master password is 85 days old. Please consider updating it soon." |
| `SECURITY_ALERT` | `SecurityService` (Risk scoring) | "A high-risk login was detected. Adaptive Auth forced an OTP verification." |
| `BACKUP_REMINDER` | `UserService` or Scheduled job | "You haven't generated backup recovery codes yet. Secure your account now." |
| `SYSTEM_UPDATE` | Admin/Internal broadside event | "The vault will undergo maintenance on Sunday between 2AM and 4AM." |
| `BREACH_DETECTED` | Periodic breach scanning logic | "One of your saved passwords was found in a recent data breach." |
| `ACCOUNT_ACTIVITY` | `VaultService` / `UserService` | "You created a new 'Bank' folder in your vault.", "You changed your master password." |

### Why an Enum?
1. **Consistency:** Prevents typos when `vault-service` tries to alert the user (e.g., sending `ACTVITY` instead of `ACCOUNT_ACTIVITY`).
2. **Frontend Styling:** Allows the Angular frontend to map a specific icon and color to a type.
   - `SECURITY_ALERT` → Red Warning Icon ⚠️
   - `ACCOUNT_ACTIVITY` → Blue Info Icon ℹ️
3. **Filtering:** Allows future features where a user could "Mute all `ACCOUNT_ACTIVITY` notifications but keep `SECURITY_ALERT`s active."

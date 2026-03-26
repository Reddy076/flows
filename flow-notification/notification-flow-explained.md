# Notification Management — Complete Flow Document
> Part of the `notification-service` deep-dive series.

This document explains the full end-to-end flow of every Notification feature, from the frontend HTTP request all the way through to the database — including every file involved, every dependency used, and ASCII diagrams for each flow.

---

## Files Involved in Notification Service

| File | Package | Role |
| :--- | :--- | :--- |
| `NotificationController.java` | `controller` | Entry point — receives all frontend HTTP requests |
| `InternalNotificationController.java` | `controller` | Entry point — receives internal service-to-service HTTP requests |
| `NotificationService.java` | `service` | Main orchestrator for creating, reading, and managing alerts |
| `NotificationRepository.java` | `repository` | DB queries for Notification entity (Spring Data JPA) |
| `Notification.java` | `model` | Database entity mapping for the `notifications` table |
| `NotificationDTO.java` | `dto` | Data Transfer Object for sending notification data to the frontend |
| `UserClient.java` | `client` | OpenFeign client — calls `user-service` to resolve user details (IDs) |
| `OpenApiConfig.java` | `config` | Configures Swagger/OpenAPI with JWT Bearer Authentication |
| `ResourceNotFoundException.java` | `exception` | Thrown when a notification does not exist or user doesn't own it |

---

## How Every Request Travels — FRONTEND → API Gateway → notification-service

```text
FRONTEND (Angular)
    │
    │  Authorization: Bearer eyJ...  
    │
    ▼
API GATEWAY (:8080)
    ├── 1. RateLimitGlobalFilter
    │       All requests    → max 100 req/min
    │       If over limit → 429 Too Many Requests
    │
    ├── 2. JwtAuthFilter
    │       /api/notifications/** → JWT required → validates signature + expiry
    │           Injects: X-User-Name: <username> as a routed HTTP header
    │           If invalid → 401 Unauthorized (stops here)
    │
    └── 3. Routes to notification-service
    │
    ▼
NOTIFICATION-SERVICE
    └── Controller
            Reads @RequestHeader("X-User-Name")
```

---

## Feature 1 — Creating a System Notification (Internal)

### What it does
When an important action occurs in another service (like the `vault-service` recording a new password or the `security-service` detecting a new IP), that service makes a synchronous `POST` request to the `notification-service` to generate an in-app alert for the user. 

### ASCII Flow Diagram

```text
[VAULT-SERVICE] or [SECURITY-SERVICE]
    |
    | POST /api/notifications/internal/create
    | Parameters: ?username=Reddy&type=ACCOUNT_ACTIVITY&title=...&message=...
    |
    v
[InternalNotificationController.java]
    | -- Converts `type` string to `NotificationType` Enum
    | -- Catches IllegalArgumentException if type is unknown
    v
[NotificationService.createNotification()]
    |-- 1. userClient.getUserDetailsByUsername(username)
    |         --> Synchronous OpenFeign HTTP call to `user-service`
    |         --> Retrieves the user's database `id` (userId)
    |-- 2. Notification.builder()
    |         --> Maps type, title, message, sets isRead = false, computes createdAt
    |-- 3. notificationRepository.save(notification)
    |         --> MYSQL: INSERT INTO notifications table
    v
[RESPONSE 200 OK]
    | Body: (empty, void return)
    v
[CALLING SERVICE]
```

### Dependencies Used
- `spring-cloud-starter-openfeign` — to fetch user details from `user-service`
- Spring Data JPA — saving the notification to the MySQL database
- `Lombok` — for `@Builder` instantiation

---

## Feature 2 — Fetching Notifications & Unread Count

### What it does
The frontend fetches the user's notification history and polls for the total number of unread alerts to show a badge icon (e.g., a little red bubble with "3" in the UI).

### ASCII Flow Diagram

```text
FRONTEND (Angular)
    |
    | GET /api/notifications (or /api/notifications/unread-count)
    | Headers: Authorization: Bearer <token>
    |
    v
[API GATEWAY] → RateLimitFilter → JwtAuthFilter → Injects X-User-Name
    |
    v
[NotificationController.java]
    | -- extracts @RequestHeader("X-User-Name") username
    v
[NotificationService.getNotifications() / getUnreadCount()]
    |-- 1. userClient.getUserDetailsByUsername(username)
    |         --> Feign call to user-service to get `userId`
    |-- 2. notificationRepository.findByUserIdOrderByCreatedAtDesc(userId)
    |      (OR countByUserIdAndIsReadFalse(userId))
    |         --> MYSQL: SELECT * FROM notifications WHERE user_id = ? ORDER BY created_at DESC
    |-- 3. mapToDTO() 
    |         --> Converts `Notification` Entity to `NotificationDTO`
    v
[RESPONSE 200 OK]
    | Body: [ { id, notificationType, title, message, isRead, createdAt }, ... ]
    | (OR Body: { count: 3 })
    v
FRONTEND
```

---

## Feature 3 — Marking Notifications as Read

### What it does
When a user clicks on a single notification to open it, or clicks a "Mark all as read" button, the system updates the boolean `is_read` flag in the database so the unread badge counter decreases.

### ASCII Flow Diagram

```text
FRONTEND
    |
    | PUT /api/notifications/{id}/read  (or /api/notifications/mark-all-read)
    | Headers: Authorization: Bearer <token>
    |
    v
[API GATEWAY] → JwtAuthFilter → Injects X-User-Name
    |
    v
[NotificationController.java]
    | -- extracts @RequestHeader("X-User-Name") username
    | -- extracts @PathVariable Long id
    v
[NotificationService.markAsRead() / markAllAsRead()]
    |-- 1. userClient.getUserDetailsByUsername(username)
    |         --> Feign call to user-service to get `userId`
    |-- 2. notificationRepository.findById(notificationId)
    |         --> MYSQL: Optional<Notification> lookup
    |-- 3. Check Ownership:
    |         --> If notification.getUserId() != userId -> throw IllegalArgumentException
    |-- 4. notification.setRead(true)
    |-- 5. notificationRepository.save(notification)
    |         --> MYSQL: UPDATE notifications SET is_read=1 WHERE id=?
    v
[RESPONSE 200 OK]
    | Body: { message: "Notification marked as read" }
    v
FRONTEND
```

---

## Feature 4 — Deleting a Notification

### What it does
Users can permanently remove a notification from their history. The system ensures they can only delete their own notifications.

### ASCII Flow Diagram

```text
FRONTEND
    |
    | DELETE /api/notifications/{id}
    | Headers: Authorization: Bearer <token>
    |
    v
[API GATEWAY] → JwtAuthFilter → Injects X-User-Name
    |
    v
[NotificationController.java]
    | -- extracts @RequestHeader("X-User-Name") username
    | -- extracts @PathVariable Long id
    v
[NotificationService.deleteNotification()]
    |-- 1. userClient.getUserDetailsByUsername(username)
    |         --> Feign call to user-service to get `userId`
    |-- 2. notificationRepository.findById(notificationId)
    |         --> MYSQL lookup. Throws ResourceNotFoundException if missing.
    |-- 3. Check Ownership:
    |         --> If notification.getUserId() != userId -> throw IllegalArgumentException
    |-- 4. notificationRepository.delete(notification)
    |         --> MYSQL: DELETE FROM notifications WHERE id=?
    v
[RESPONSE 200 OK]
    | Body: { message: "Notification deleted" }
    v
FRONTEND
```

---

## Dependencies Summary

| Dependency | Used For |
| :--- | :--- |
| `spring-boot-starter-web` | Core REST controllers and HTTP request mapping |
| `spring-boot-starter-data-jpa` | ORM, Repositories, and Database operations |
| `mysql-connector-j` | Connecting the service to the MySQL instance |
| `spring-cloud-starter-openfeign` | `UserClient` synchronous HTTP calls to `user-service` |
| `spring-cloud-starter-netflix-eureka-client` | Service discovery (finding `user-service`) |
| `springdoc-openapi-starter-webmvc-ui` | Swagger UI generation for API documentation |
| `Lombok` | `@RequiredArgsConstructor`, `@Builder`, `@Data` to reduce boilerplate |

---

## Database Tables Used

| Table | Written By | Read By |
| :--- | :--- | :--- |
| `notifications` | `NotificationService` (creation, marking read, deletion) | `NotificationService` (fetching, unread counting) |

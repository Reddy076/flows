# Notification Service — Complete Feature List

This document lists every feature implemented and exposed by the `notification-service`.

---

## 1. Notification Management (`/api/notifications`)
Managed by `NotificationController` → `NotificationService`

| Feature | Method | Endpoint |
| :--- | :--- | :--- |
| Get all notifications for user | `GET` | `/api/notifications` |
| Get unread notification count | `GET` | `/api/notifications/unread-count` |
| Mark a specific notification as read | `PUT` | `/api/notifications/{id}/read` |
| Mark all notifications as read | `PUT` | `/api/notifications/mark-all-read` |
| Delete a specific notification | `DELETE` | `/api/notifications/{id}` |

---

## 2. Internal / Service-to-Service (`/api/notifications/internal`)
Managed by `InternalNotificationController` — called by other microservices only, not the frontend.

| Feature | Used By |
| :--- | :--- |
| Create a new notification | `vault-service` |

---

## 3. Cross-Cutting / Infrastructure
These features apply across the service footprint and aren't tied to a single frontend API.

| Feature | Class |
| :--- | :--- |
| Fetch User vault details via synchronous OpenFeign | `UserClient` |
| JWT Bearer Auth configuration for Swagger UI | `OpenApiConfig` |

---

## 4. Supported Notification Types
The service supports the following predefined types of notifications (stored as an Enum):

- `PASSWORD_EXPIRY`
- `SECURITY_ALERT`
- `BACKUP_REMINDER`
- `SYSTEM_UPDATE`
- `BREACH_DETECTED`
- `ACCOUNT_ACTIVITY`

*(Note: The service currently handles internal, in-app notifications only. Email notifications are not enabled in this service's current configuration).*


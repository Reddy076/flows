# 1. General Security & Audit — Complete Flow Document
> Part of the `security-service` deep-dive series.

This document explains the end-to-end flow for the general security and audit
endpoints where a user views their own personal security history.

---

## How Requests Reach the Security Service

```
FRONTEND (Angular :4200)
    │
    │  GET /api/security/audit-logs
    │  Authorization: Bearer eyJ...
    │
    ▼
API GATEWAY (:8080)
    ├── 1. RateLimitGlobalFilter → Rate check (429 if over limit)
    │
    ├── 2. JwtAuthFilter
    │       /api/security/** → PROTECTED route → JWT explicitly required
    │       Validates JWT signature + expiry
    │       Injects into forwarded request:
    │           X-User-Name: john    ← SecurityController reads this
    │           X-Is-Duress: false
    │
    └── 3. Routes to security-service (:8083)
    │
    ▼
SECURITY-SERVICE (:8083)
    ├── CorsConfig              → Validates Origin
    ├── GatewayAuthFilter       → Reads X-User-Name → sets SecurityContext
    └── SecurityConfig          → Requires authenticated() for /api/security/**
                 (Controller methods extract X-User-Name directly from headers)
```

---

## Files Involved

| File | Package | Role |
| :--- | :--- | :--- |
| `SecurityController.java` | `controller` | HTTP entry point handling `GET /api/security/**` |
| `AuditLogService.java` | `service` | Fetches audit trail directly from DB |
| `LoginAttemptService.java` | `service` | Fetches login history; auto-creates alerts on too many failed logins |
| `SecurityAlertService.java` | `service` | Fetches, reads, and deletes security alerts; sends emails |
| `UserClient.java` | `client` | Feign — resolves the username header to a continuous numeric `userId` |
| `NotificationClient.java`| `client` | Feign — sends emails for new Security Alerts |
| `AuditLogRepository.java` | `repository` | JPA queries for the `audit_logs` table |
| `LoginAttemptRepository.java`| `repository` | JPA queries for the `login_attempts` table |
| `SecurityAlertRepository.java`| `repository` | JPA queries for the `security_alerts` table |

### DTOs

| Class | Role |
| :--- | :--- |
| `AuditLogResponse` | `id, action, details, ipAddress, timestamp` |
| `LoginHistoryResponse` | `id, successful, ipAddress, deviceInfo, timestamp` |
| `SecurityAlertDTO` | `id, alertType, title, message, severity, isRead, createdAt` |

---

## Feature 1a — Get All Audit Logs

### What it does
Returns a chronological list of every security, vault, and authentication action the user
has taken. This includes logins, vault creations, password views, and sharing events.

### ASCII Flow Diagram

```
FRONTEND
    |
    | GET /api/security/audit-logs
    | Headers: Authorization: Bearer <token>
    |
    v
[API GATEWAY] → validates JWT → X-User-Name: john → security-service
    |
    v
[SecurityController.getAuditLogs(username="john")]
    v
[AuditLogService.getAuditLogs("john")]
    |
    |-- STEP 1: RESOLVE USER ID
    |   userClient.getUserDetailsByUsername("john")
    |       → Feign: GET http://user-service/api/users/internal/by-username/john
    |       → Returns { id: 1, email: "john@example.com", ... }
    |
    |-- STEP 2: FETCH LOGS FROM DB
    |   auditLogRepository.findByUserIdOrderByTimestampDesc(1)
    |       → MYSQL: SELECT * FROM audit_logs
    |                WHERE user_id = 1 ORDER BY timestamp DESC
    |       → Returns List<AuditLog>
    |
    |-- STEP 3: MAPPING
    |   Maps domain objects to AuditLogResponse DTOs
    v
[RESPONSE 200 OK]
    | Body: [
    |   { id: 142, action: "ENTRY_CREATED", details: "Created entry: Netflix", ipAddress: "192.168.1.10", timestamp: "..." },
    |   { id: 141, action: "LOGIN", details: "Successful login (IP: 192.168.1.10)", ipAddress: "192.168.1.10", timestamp: "..." }
    | ]
```

---

## Feature 1b — Get Login History & Failed Login Lockout Trigger

### What it does
Returns a historical list of login attempts (both successful and failed) along with the
device/user-agent used and IP address.

**Background protection:** When recording login attempts internally, the system checks
if the user has had >5 failed logins in 15 minutes. If so, a high-severity alert is fired.

### ASCII Flow Diagram — Fetching Login History

```
FRONTEND
    |
    | GET /api/security/login-history
    | Headers: Authorization: Bearer <token>
    |
    v
[API GATEWAY] → validates JWT → X-User-Name: john → security-service
    |
    v
[SecurityController.getLoginHistory(username="john")]
    v
[LoginAttemptService.getLoginHistory("john")]
    |
    |-- STEP 1: RESOLVE USER ID
    |   userClient.getUserDetailsByUsername("john") → returns userId = 1
    |
    |-- STEP 2: FETCH LOGINS FROM DB
    |   loginAttemptRepository.findByUserIdOrderByTimestampDesc(1)
    |       → MYSQL: SELECT * FROM login_attempts
    |                WHERE user_id = 1 ORDER BY timestamp DESC
    |
    |-- STEP 3: MAPPING
    |   Maps domain objects to LoginHistoryResponse DTOs
    v
[RESPONSE 200 OK]
    | Body: [
    |   { id: 88, successful: true, ipAddress: "192.168.1.10", deviceInfo: "Chrome 114 (Windows)", timestamp: "..." },
    |   { id: 87, successful: false, ipAddress: "45.22.11.9", deviceInfo: "Unknown Bot", timestamp: "..." }
    | ]
```

### ASCII Flow Diagram — Automatic Failed Login Alert
*(Triggered internally behind the scenes, not by a frontend GET request)*

```
[user-service] logs a failed password attempt
    |
    v
[security-service: LoginAttemptService.recordLoginAttempt(successful=false)]
    |
    |-- Saves failure to login_attempts DB
    |
    |-- checkFailedAttempts(userId=1):
    |   lockoutThreshold = now() - 15 minutes
    |   count = loginAttemptRepository.countByUserIdAndIsSuccessfulFalseAndTimestampAfter
    |   If count >= 5 (MAX_FAILED_ATTEMPTS):
    |       securityAlertService.createAlert(
    |           username: "john",
    |           type: MULTIPLE_FAILED_LOGINS,
    |           severity: HIGH
    |       )
```

---

## Feature 1c — Security Alerts (Fetch, Read, Delete)

### What it does
Allows users to view critical security notifications (e.g., "New device logged in",
"Master password changed", "Multiple failed logins"). Users can mark them as read or
delete them. **When an alert is created, an email is automatically sent via notification-service.**

### ASCII Flow Diagram — Creating an Alert (Internal Trigger)

```
[Any Service (user-service or vault-service)]
    | Needs to warn the user about something dangerous
    v
[SecurityAlertService.createAlert("john", "MULTIPLE_FAILED_LOGINS", "Suspicious Login", "...", HIGH)]
    |
    |-- userClient.getUserDetailsByUsername("john") → userId = 1
    |
    |-- Saves to DB
    |   securityAlertRepository.save(alert)
    |
    |-- NOTIFICATION FEIGN CALL (Email dispatched)
    |   notificationClient.sendNotification(
    |       NotificationRequest(user="john", type="SECURITY_ALERT", title="...")
    |   )
    |   → HTTP POST http://notification-service/api/notifications/internal/send
```

### ASCII Flow Diagram — Fetch & Mark Read (Frontend)

```
FRONTEND
    |
    | GET /api/security/alerts
    |
    v
[API GATEWAY] → security-service
    |
    v
[SecurityAlertService.getAlerts("john")]
    |-- userClient.getUserDetailsByUsername("john") → userId = 1
    |-- database fetch: findByUserIdOrderByCreatedAtDesc(1)
    |   → Returns 2 alerts (1 unread, 1 read)
    v
[RESPONSE 200 OK — Array of Alerts]

FRONTEND
    | User clicks on the unread alert to view the message
    | PUT /api/security/alerts/99/read
    v
[API GATEWAY] → security-service
    |
    v
[SecurityAlertService.markAsRead("john", alertId=99)]
    |-- userClient.getUserDetailsByUsername("john") → userId = 1
    |-- FETCH ALERT: securityAlertRepository.findById(99)
    |-- SECURITY CHECK: Does alert.getUserId() == 1?
    |   (Throws IllegalArgumentException if user tries to read someone else's alert)
    |-- alert.setRead(true)
    |-- securityAlertRepository.save(alert)
    v
[RESPONSE 200 OK] { "message": "Alert marked as read" }
```

---

## Security Design Notes

| Decision | Why |
| :--- | :--- |
| `userId` resolved dynamically via Feign | The gateway sets the username in headers, but the database schema uses numeric user IDs as foreign keys. Fetching from `user-service` guarantees syncing between microservices. |
| Automatic Emailing on Alerts | Security alerts are critical (e.g. account takeover attempt). Sending an email immediately via `notification-service` ensures the user knows even if they aren't looking at the dashboard. |
| Ownership check on Read/Delete | `markAsRead` and `deleteAlert` both verify `alert.getUserId().equals(userId)` to prevent Insecure Direct Object Reference (IDOR) attacks. |
| Transaction Propagation `REQUIRES_NEW` | `AuditLogService.logAction` suspends current transactions to guarantee that audit logs persist *even if the main operation fails and rolls back*. |

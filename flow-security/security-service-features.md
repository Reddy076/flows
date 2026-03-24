# Security Service — Complete Feature List
> This document lists all features, endpoints, and background tasks handled by `security-service`. 
> Reference this to understand the full scope of what the security-service microservice does.

---

## 1. General Security & Audit (`/api/security`)

Provides users with their personal security history, audit logs, and security alerts.

| Feature | Method | Endpoint | Service Class |
| :--- | :--- | :--- | :--- |
| Get all audit logs | `GET` | `/api/security/audit-logs` | `AuditLogService` |
| Get login history | `GET` | `/api/security/login-history` | `LoginAttemptService` |
| Get security alerts | `GET` | `/api/security/alerts` | `SecurityAlertService` |
| Mark alert as read | `PUT` | `/api/security/alerts/{id}/read` | `SecurityAlertService` |
| Delete alert | `DELETE` | `/api/security/alerts/{id}` | `SecurityAlertService` |

**Key Design Notes:**
- Returns data for the currently authenticated user (via `X-User-Name` header).
- Relies on the database tables: `audit_logs`, `login_attempts`, `security_alerts`.

---

## 2. Password Health & Auditing (`/api/security`)

Provides detailed analysis of the user's vault entries, identifying weak, reused, or old passwords.

| Feature | Method | Endpoint | Service Class |
| :--- | :--- | :--- | :--- |
| Get full audit report | `GET` | `/api/security/audit-report` | `SecurityAuditService` |
| Get weak passwords | `GET` | `/api/security/weak-passwords` | `SecurityAuditService` |
| Get reused passwords | `GET` | `/api/security/reused-passwords` | `SecurityAuditService` |
| Get old passwords | `GET` | `/api/security/old-passwords` | `SecurityAuditService` |
| Trigger vault analysis | `POST` | `/api/security/analyze-vault` | `SecurityAuditService` |

**Key Design Notes:**
- "Weak" is determined by the `strengthScore` calculated during entry creation.
- "Reused" checks for duplicate password hashes/strings across different entries.
- "Old" checks the `lastAnalyzed` or creation date against a configured threshold (e.g., 90 days).

---

## 3. Security Dashboard Metrics (`/api/dashboard`)

Provides aggregated stats and scores for the frontend security dashboard.

| Feature | Method | Endpoint | Service Class |
| :--- | :--- | :--- | :--- |
| Get overall security score | `GET` | `/api/dashboard/security-score` | `PasswordStrengthDashboardService` |
| Get password health summary | `GET` | `/api/dashboard/password-health` | `PasswordStrengthDashboardService` |
| Get reused passwords stats | `GET` | `/api/dashboard/reused-passwords` | `PasswordStrengthDashboardService` |
| Get password age stats | `GET` | `/api/dashboard/password-age` | `PasswordStrengthDashboardService` |
| Get security trends (30 days) | `GET` | `/api/dashboard/trends` | `PasswordStrengthDashboardService` |
| Get weak passwords list | `GET` | `/api/dashboard/passwords/weak` | `PasswordStrengthDashboardService` |
| Get old passwords list | `GET` | `/api/dashboard/passwords/old` | `PasswordStrengthDashboardService` |

**Key Design Notes:**
- Calculates an overall "Security Score" (0-100) based on password strength, 2FA status, and unresolved alerts.
- Uses `SecurityMetricsHistory` to provide trend data over the last 30 days.
- Designed specifically to populate UI charts and summary tiles efficiently.

---

## 4. Internal Service-to-Service — Vault Events (`/api/security/internal`)

Endpoints called by `vault-service` via Feign clients. Not exposed to the frontend.

| Feature | Method | Endpoint | Service Class |
| :--- | :--- | :--- | :--- |
| Log vault audit event | `POST` | `/api/security/internal/audit` | `AuditLogService` |
| Analyze single password | `POST` | `/api/security/internal/analyze` | `InternalSecurityController` |

**Key Design Notes:**
- **Audit:** Called when a vault entry is created, viewed, shared, deleted, etc.
- **Analyze:** Called immediately after a user creates or updates a vault entry. Calculates the `strengthScore` (0-100) and specific `issues` (e.g., "Add uppercase letters"), then saves to `password_analysis` table.

---

## 5. Internal Service-to-Service — User Events (`/internal`)

Endpoints called by `user-service` via Feign clients. Not exposed to the frontend.

| Feature | Method | Endpoint | Service Class |
| :--- | :--- | :--- | :--- |
| Log user audit event | `POST` | `/internal/audit` | `AuditLogService` |
| Create security alert | `POST` | `/internal/alerts` | `SecurityAlertService` |

**Key Design Notes:**
- **Audit:** Logs authentication events: logins, logouts, password changes, 2FA enablement, etc. Includes IP address tracking.
- **Alerts:** Creates high-priority security alerts for events like failed logins, new device logins, or master password changes. Stores them in the `security_alerts` table.

---

## 6. Cross-Cutting / Security Infrastructure

Features that apply globally within the `security-service`.

| Feature | Class | Role |
| :--- | :--- | :--- |
| Feign client to user-service | `UserClient` | Resolves `username` to user details if needed. |
| JWT trust filter | `GatewayAuthFilter` | Reads `X-User-Name` set by gateway and populates `SecurityContext`. |

**Key Design Notes:**
- Like the other microservices, it trusts the API Gateway for JWT validation and reads the `X-User-Name` header.
- The internal endpoints (`/api/security/internal/**` and `/internal/**`) are `permitAll()` and trust Docker network isolation, expecting to be called directly by other microservices.

---

## Database Tables Owned (`security_db`)

| Table | Purpose |
| :--- | :--- |
| `audit_logs` | Chronological record of every action taken by every user across all services. |
| `login_attempts` | History of successful and failed logins, including IP addresses and timestamps. |
| `password_analysis` | Stores the calculated strength score (0-100) and list of issues for each vault entry. |
| `security_alerts` | Notifications for suspicious activity or important security events (e.g., new device login). |
| `security_metrics_history` | Snapshots of a user's security score over time, used to generate trend charts. |

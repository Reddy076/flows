# User Service — Complete Feature List

This document lists every feature implemented and exposed by the `user-service`.

---

## 1. Authentication (`/api/auth`)
Managed by `AuthController` → `AuthenticationService`

| Feature | Method | Endpoint |
| :--- | :--- | :--- |
| Register new user | `POST` | `/api/auth/register` |
| Verify email via OTP | `POST` | `/api/auth/verify-email` |
| Resend email verification OTP | `POST` | `/api/auth/resend-verification-otp` |
| Standard Login | `POST` | `/api/auth/login` |
| Logout (invalidate session) | `POST` | `/api/auth/logout` |
| Refresh expired JWT | `POST` | `/api/auth/refresh-token` |
| Validate a JWT token | `GET` | `/api/auth/validate-token` |
| Verify master password (re-auth) | `POST` | `/api/auth/verify-master-password` |
| Send OTP to email | `POST` | `/api/auth/send-otp` |
| Resend OTP to email | `POST` | `/api/auth/resend-otp` |
| Verify OTP (2FA / high-risk login) | `POST` | `/api/auth/verify-otp` |
| Forgot password (send recovery link) | `POST` | `/api/auth/forgot-password` |
| Reset password via security questions | `POST` | `/api/auth/reset-password` |
| Verify security question answers | `POST` | `/api/auth/verify-security-questions` |
| Get security questions for a user | `GET` | `/api/auth/security-questions/{username}` |
| Verify CAPTCHA token | `POST` | `/api/auth/verify-captcha` |
| Duress login (emergency mode) | `POST` | `/api/auth/duress-login` |
| Set duress password | `POST` | `/api/auth/set-duress-password` |
| Get password hint | `GET` | `/api/auth/password-hint/{username}` |
| Set password hint | `PUT` | `/api/auth/password-hint` |

---

## 2. User Profile & Account (`/api/users`)
Managed by `UserController` → `UserService`, `AccountDeletionService`

| Feature | Method | Endpoint |
| :--- | :--- | :--- |
| Get user profile | `GET` | `/api/users/profile` |
| Update user profile | `PUT` | `/api/users/profile` |
| Change master password | `PUT` | `/api/users/change-password` |
| Get dashboard stats (vault counts) | `GET` | `/api/users/dashboard` |
| Get security questions | `GET` | `/api/users/security-questions` |
| Update security questions | `PUT` | `/api/users/security-questions` |
| Toggle vault read-only mode | `PUT` | `/api/users/read-only-mode` |
| Schedule account deletion (30 days) | `DELETE` | `/api/users/account` |
| Cancel scheduled account deletion | `POST` | `/api/users/account/cancel-deletion` |

---

## 3. Two-Factor Authentication (`/api/2fa`)
Managed by `TwoFactorController` → `TwoFactorService`

| Feature | Method | Endpoint |
| :--- | :--- | :--- |
| Get 2FA status (enabled/disabled) | `GET` | `/api/2fa/status` |
| Initialize 2FA setup (get QR code) | `POST` | `/api/2fa/setup` |
| Verify and enable 2FA | `POST` | `/api/2fa/verify-setup` |
| Disable 2FA | `POST` | `/api/2fa/disable` |
| Get backup recovery codes | `GET` | `/api/2fa/backup-codes` |
| Regenerate backup recovery codes | `POST` | `/api/2fa/regenerate-codes` |

---

## 4. Session Management (`/api/sessions`)
Managed by `SessionController` → `SessionService` (backed by Redis)

| Feature | Method | Endpoint |
| :--- | :--- | :--- |
| List all active sessions | `GET` | `/api/sessions` |
| Get current session details | `GET` | `/api/sessions/current` |
| Extend current session | `POST` | `/api/sessions/extend` |
| Terminate a specific session | `DELETE` | `/api/sessions/{sessionId}` |
| Terminate ALL sessions (global logout) | `DELETE` | `/api/sessions` |

---

## 5. User Settings (`/api/settings`)
Managed by `UserSettingsController` → `UserSettingsService`

| Feature | Method | Endpoint |
| :--- | :--- | :--- |
| Get user settings | `GET` | `/api/settings` |
| Update user settings | `PUT` | `/api/settings` |

---

## 6. Internal / Service-to-Service (`/api/internal`)
Managed by `InternalUserController` — called by other microservices only, not the frontend.

| Feature | Used By |
| :--- | :--- |
| Fetch user details by username | `vault-service`, `security-service` |
| Fetch user details by ID | `vault-service` |

---

## 7. Background / Automated Features
These run on a schedule with no API endpoint.

| Feature | Frequency | Class |
| :--- | :--- | :--- |
| Permanently delete accounts scheduled for deletion | Daily at midnight | `AccountDeletionScheduler` |

---

## 8. Cross-Cutting / Security Infrastructure
These features apply to every request and are not tied to a single endpoint.

| Feature | Class |
| :--- | :--- |
| JWT token generation & validation | `JwtTokenProvider` |
| JWT filter (validates token on every request) | `JwtAuthenticationFilter` |
| Login session tracking (Redis) | `SessionService` |
| Adaptive risk scoring (IP, device, time) | `AdaptiveAuthService` |
| Login attempt tracking & account lockout | `LoginAttemptService` |
| CAPTCHA enforcement after failed logins | `CaptchaService` |
| Device fingerprinting | `DeviceFingerprintUtil` |
| IP-based geolocation | `GeoLocationService` |
| AOP method-level logging | `LoggingAspect` |
| CORS policy for frontend | `CorsConfig` |
| JWT Bearer Auth in Swagger UI | `OpenApiConfig` |

# Two-Factor Authentication (2FA) — Complete Flow Document
> Part of the `user-service` deep-dive series.

This document explains the full end-to-end flow of every 2FA feature, from the frontend
HTTP request all the way through to the database — including every file involved,
every dependency used, and ASCII diagrams for each flow.

---

## What is TOTP?

TOTP stands for **Time-Based One-Time Password**. It is the same technology used
by Google Authenticator, Microsoft Authenticator, and Authy.

```
HOW IT WORKS:
    1. Server generates a random secret key (e.g. "JBSWY3DPEHPK3PXP")
    2. User scans QR code → secret is stored in their authenticator app
    3. Every 30 seconds, both the app AND the server compute:
           HMAC-SHA1(secret + current_time_in_30sec_windows)
           → truncate to 6 digits
    4. If the user types a code that matches what the server computed → verified
    5. The code changes every 30 seconds and can never be reused
```

This means even if someone steals your password, they still can't log in without
the constantly-changing 6-digit code from the physical device.

---

## Files Involved

| File | Package | Role |
| :--- | :--- | :--- |
| `TwoFactorController.java` | `controller` | Entry point for all `/api/2fa/**` endpoints |
| `TwoFactorService.java` | `service/auth` | All 2FA business logic — setup, verify, disable, backup codes |
| `TOTPUtil.java` | `util` | Generates secrets, creates QR codes, verifies TOTP codes |
| `OtpService.java` | `service/auth` | Used for email OTP during high-risk 2FA login step |
| `SessionService.java` | `service/auth` | Reads session to identify user from JWT token |
| `SecurityClient.java` | `client` | Fires security alerts when 2FA is enabled/disabled |
| `TwoFactorAuthRepository.java` | `repository` | DB queries for `two_factor_auth` table |
| `UserRepository.java` | `repository` | Updates `is_2fa_enabled` flag on the `users` table |
| `JwtAuthFilter.java` | `api-gateway` | First-line JWT check at gateway |
| `RateLimitGlobalFilter.java` | `api-gateway` | Rate limiting before request reaches user-service |
| `CorsConfig.java` | `config` | CORS rules for browser preflight requests |
| `SecurityConfig.java` | `config` | Marks all `/api/2fa/**` routes as JWT-protected |
| `JwtAuthenticationFilter.java` | `security` | Second JWT check inside user-service |

### External Library
| Library | Used For |
| :--- | :--- |
| `dev.samstevens.totp` | Secret generation, QR code creation, TOTP code verification |
| `ZXing (Zebra Crossing)` | QR code image rendering (used internally by samstevens library) |

---

## How Every 2FA Request is Received (Gateway + CORS)

All 2FA endpoints are **protected** — a valid JWT is required. The flow through
the API Gateway is identical to all other protected routes:

```
FRONTEND (Angular)
    |
    | POST /api/2fa/setup   (or /status, /verify-setup, etc.)
    | Headers: Authorization: Bearer eyJ...
    | Origin: http://localhost:4200
    |
    v
[API GATEWAY - port 8080]
    |-- RateLimitGlobalFilter: 100 req/min per IP (general limit, not auth)
    |-- JwtAuthFilter:
    |       Path is /api/2fa/** (NOT /api/auth/**) --> JWT IS required
    |       Validates JWT signature with shared secret
    |       Extracts username from "sub" claim
    |       Adds headers: X-User-Name: "john", X-Is-Duress: "false"
    |       Forwards to user-service:8081
    v
[USER-SERVICE - Spring Security Filter Chain]
    |-- CorsConfig: validates Origin header (allows localhost:4200)
    |-- JwtAuthenticationFilter: second JWT validation (defence in depth)
    |       Validates token + checks session is still active in DB
    |       Populates SecurityContextHolder
    |-- SecurityConfig: /api/2fa/** is .anyRequest().authenticated() → JWT required
    v
[TwoFactorController.java]
    | -- getUserBySession(request): extracts username from SecurityContext
    | -- userRepository.findByUsername(username) --> MYSQL SELECT
```

---

## Feature 1 — Get 2FA Status

### What it does
Returns whether the logged-in user currently has 2FA enabled or disabled.

### ASCII Flow Diagram

```
FRONTEND
    |
    | GET /api/2fa/status
    | Headers: Authorization: Bearer <token>
    |
    v
[API GATEWAY] --> [JwtAuthenticationFilter] --> [TwoFactorController.getStatus()]
    |
    |-- getUserBySession(request)
    |       SecurityContextHolder.getContext().getAuthentication().getPrincipal()
    |       → extracts username from Spring Security context
    |       userRepository.findByUsername() --> MYSQL SELECT
    |
    |-- user.is2faEnabled()
    |       → returns the boolean flag stored on the User entity
    |
    v
[RESPONSE 200 OK]
    | Body: { is2faEnabled: false }
```

---

## Feature 2 — Setup 2FA (Get QR Code)

### What it does
Initializes the 2FA setup for a user. Generates a random **secret key**, stores it in the
database (not yet active), and returns a **QR code data URI** the user scans into
their authenticator app. 2FA is NOT yet enabled at this step.

### ASCII Flow Diagram

```
FRONTEND
    |
    | POST /api/2fa/setup
    | Headers: Authorization: Bearer <token>
    |
    v
[TwoFactorController.setup2FA()]
    |-- getUserBySession() --> get User from DB
    v
[TwoFactorService.setup2FA(user)]
    |
    |-- STEP 1: FIND OR CREATE TwoFactorAuth RECORD
    |   twoFactorAuthRepository.findByUser(user)  --> MYSQL SELECT
    |   If exists: reuse it (user is re-setting up)
    |   If not:   create new TwoFactorAuth entity
    |
    |-- STEP 2: GENERATE SECRET KEY
    |   totpUtil.generateSecret()
    |       --> DefaultSecretGenerator.generate()
    |       --> Generates a random Base32-encoded string e.g. "JBSWY3DPEHPK3PXP"
    |       --> This is the shared secret between the server and the authenticator app
    |
    |-- STEP 3: SAVE TO DB (not yet enabled)
    |   twoFactorAuth.setSecretKey(secretKey)
    |   twoFactorAuth.setEnabled(false)    <-- IMPORTANT: not active yet
    |   twoFactorAuthRepository.save()  --> MYSQL INSERT or UPDATE
    |
    |-- STEP 4: GENERATE QR CODE
    |   totpUtil.getQrCodeUrl(secretKey, user.getEmail())
    |       --> Builds QrData:
    |             label: "john@example.com"
    |             secret: "JBSWY3DPEHPK3PXP"
    |             issuer: "Rev-PasswordManager"
    |             algorithm: SHA1
    |             digits: 6
    |             period: 30 seconds
    |       --> ZxingPngQrGenerator.generate(data) --> PNG byte array
    |       --> getDataUriForImage() --> "data:image/png;base64,iVBORw0..."
    |
    v
[RESPONSE 200 OK]
    | Body: {
    |   secretKey: "JBSWY3DPEHPK3PXP",
    |   qrCodeUrl: "data:image/png;base64,..."
    | }
    v
FRONTEND
    | -- Renders the QR code as an <img> tag
    | -- User opens Google Authenticator and scans it
    | -- App stores the secret and starts generating 6-digit codes every 30s
```

---

## Feature 3 — Verify Setup & Enable 2FA

### What it does
The user types the first 6-digit code from their authenticator app to confirm the secret
was scanned correctly. On success: 2FA is **activated**, 10 backup recovery codes are
generated, and a security alert is sent.

### ASCII Flow Diagram

```
FRONTEND
    | (user scanned QR code and has a 6-digit code ready)
    |
    | POST /api/2fa/verify-setup?code=123456
    | Headers: Authorization: Bearer <token>
    |
    v
[TwoFactorController.verifySetup(code)]
    |-- getUserBySession() --> get User from DB
    v
[TwoFactorService.verifySetup(user, code)]
    |
    |-- STEP 1: FETCH THE TwoFactorAuth RECORD
    |   twoFactorAuthRepository.findByUser(user) --> MYSQL SELECT
    |   If not found --> throw "2FA setup not initiated" (must call /setup first)
    |
    |-- STEP 2: VERIFY THE TOTP CODE
    |   totpUtil.verifyCode(twoFactorAuth.getSecretKey(), code)
    |       --> DefaultCodeVerifier.isValidCode(secret, "123456")
    |       --> Computes: HMAC-SHA1(secret, floor(now/30))
    |       --> Truncates to 6 digits
    |       --> Checks submitted code matches computed code
    |       --> Also checks: current window AND ±1 window (accounts for clock drift)
    |   If WRONG --> throw "Invalid 2FA code"
    |
    |-- STEP 3: ACTIVATE 2FA
    |   twoFactorAuth.setEnabled(true)  <-- now active!
    |
    |-- STEP 4: GENERATE 10 BACKUP RECOVERY CODES
    |   generateRecoveryCodes()
    |       --> 10x generateRecoveryCode()
    |           --> SecureRandom picks 10 chars from BASE32_ALPHABET
    |           --> e.g. "ABCD12EFGH", "WXYZ34MNOP", ...
    |   twoFactorAuth.setRecoveryCodes(newCodes)
    |   twoFactorAuthRepository.save()  --> MYSQL UPDATE
    |
    |-- STEP 5: UPDATE USER ENTITY
    |   user.set2faEnabled(true)
    |   userRepository.save()  --> MYSQL UPDATE users SET is_2fa_enabled = true
    |
    |-- STEP 6: FIRE SECURITY ALERT
    |   securityClient.createAlert(username, "TWO_FA_ENABLED", "LOW")
    |       --> HTTP call to security-service
    |
    v
[RESPONSE 200 OK]
    | Body: {
    |   success: true,
    |   message: "2FA enabled successfully",
    |   recoveryCodes: [
    |     "ABCD12EFGH",
    |     "WXYZ34MNOP",
    |     ... (10 codes total)
    |   ]
    | }
    v
FRONTEND
    | -- Shows user the 10 recovery codes with a WARNING:
    | -- "Save these codes somewhere safe. Each can only be used once."
```

---

## Feature 4 — 2FA During Login

### What it does
When a user with 2FA enabled successfully enters their password, login is **paused**.
They must also provide their 6-digit TOTP code before JWT tokens are issued.

### ASCII Flow Diagram

```
FRONTEND
    |
    | POST /api/auth/login
    | Body: { username: "john", password: "Pass123!" }
    |
    v
[AuthenticationService.authenticate()]
    |-- Validates credentials (BCrypt match) ✅
    |-- Checks: user.is2faEnabled() == true
    |
    |-- Returns EARLY (no token yet):
    |   { requiresTwoFactor: true, username: "john" }
    |
FRONTEND
    | -- Sees requiresTwoFactor: true
    | -- Shows 2FA code input screen
    |
    | POST /api/auth/verify-otp
    | Body: { username: "john", otpCode: "482910" }
    |
    v
[AuthenticationService.verifyLoginOtp()]
    |
    |-- userRepository.findByUsername() --> MYSQL SELECT
    |
    |-- IF user.is2faEnabled() == true:
    |   twoFactorService.verifyCode(user, "482910")
    |       --> twoFactorAuthRepository.findByUser() --> MYSQL SELECT
    |       --> totpUtil.verifyCode(secretKey, "482910")
    |           --> HMAC-SHA1 verification (same as Feature 3 Step 2)
    |
    |-- IF TOTP fails, try recovery code:
    |   twoFactorAuth.getRecoveryCodes().contains("482910")?
    |       YES --> remove that code (one-time use) --> MYSQL UPDATE
    |       NO  --> throw "Invalid 2FA code"
    |
    |-- ON SUCCESS: generate JWT + create session
    |   (same as normal login flow after credential verification)
    |
    v
[RESPONSE 200 OK]
    | Body: { accessToken: "eyJ...", refreshToken: "eyJ..." }
```

---

## Feature 5 — Disable 2FA

### What it does
Removes 2FA from the account. The `TwoFactorAuth` record is permanently deleted
and the `users` table flag is set back to false. A HIGH severity alert is sent
because this is a potentially dangerous action.

### ASCII Flow Diagram

```
FRONTEND
    |
    | POST /api/2fa/disable
    | Headers: Authorization: Bearer <token>
    |
    v
[TwoFactorController.disable2FA()]
    |-- getUserBySession() --> get User from DB
    v
[TwoFactorService.disable2FA(user)]
    |
    |-- STEP 1: FIND TwoFactorAuth RECORD
    |   twoFactorAuthRepository.findByUser(user) --> MYSQL SELECT
    |   If not found --> throw "2FA not set up for this user"
    |
    |-- STEP 2: UPDATE USER FLAG
    |   user.set2faEnabled(false)
    |   userRepository.save()  --> MYSQL UPDATE users SET is_2fa_enabled = false
    |
    |-- STEP 3: DELETE THE 2FA RECORD
    |   twoFactorAuthRepository.delete(twoFactorAuth)
    |       --> MYSQL DELETE FROM two_factor_auth WHERE user_id = ?
    |       --> Secret key and all backup codes are permanently wiped
    |
    |-- STEP 4: HIGH SEVERITY ALERT
    |   securityClient.createAlert(username, "TWO_FA_DISABLED", "HIGH")
    |       --> "If you did not make this change, secure your account immediately."
    |
    v
[RESPONSE 200 OK]
    | Body: { message: "2FA disabled successfully" }
```

---

## Feature 6 — Get Backup Codes

### What it does
Returns the user's current list of unused backup recovery codes.
These codes can be used instead of the TOTP code when the user
does not have access to their authenticator app (e.g. lost phone).

### ASCII Flow Diagram

```
FRONTEND
    |
    | GET /api/2fa/backup-codes
    | Headers: Authorization: Bearer <token>
    |
    v
[TwoFactorController.getBackupCodes()]
    |-- getUserBySession() --> get User from DB
    v
[TwoFactorService.getRecoveryCodes(user)]
    |-- twoFactorAuthRepository.findByUser(user) --> MYSQL SELECT
    |   If not found --> throw "2FA not set up"
    |-- return twoFactorAuth.getRecoveryCodes()
    |       --> List<String> stored as semicolon-delimited string in DB
    |       --> Deserialized by StringListConverter.java (JPA @Converter)
    v
[RESPONSE 200 OK]
    | Body: ["ABCD12EFGH", "WXYZ34MNOP", ...] (remaining unused codes)
```

> **How StringListConverter works:**
> The backup codes are stored in a single DB column as `"ABCD12EFGH;WXYZ34MNOP;..."`.
> JPA automatically converts this to/from `List<String>` using `StringListConverter.java`
> which implements `AttributeConverter<List<String>, String>`.

---

## Feature 7 — Regenerate Backup Codes

### What it does
Generates a fresh set of 10 backup codes, replacing all old ones. Useful when the
user has used most of their codes or suspects they were compromised.

### ASCII Flow Diagram

```
FRONTEND
    |
    | POST /api/2fa/regenerate-codes
    | Headers: Authorization: Bearer <token>
    |
    v
[TwoFactorController.regenerateCodes()]
    |-- getUserBySession() --> get User from DB
    v
[TwoFactorService.regenerateRecoveryCodes(user)]
    |
    |-- twoFactorAuthRepository.findByUser(user) --> MYSQL SELECT
    |   If not found --> throw "2FA not set up"
    |-- Check: twoFactorAuth.isEnabled() == true
    |   If not enabled --> throw "Cannot regenerate codes for unverified setup"
    |
    |-- generateRecoveryCodes()
    |       --> 10x: SecureRandom picks 10 Base32 chars
    |       --> ["NEWCODE1AB", "NEWCODE2CD", ...] (all old codes now invalid)
    |
    |-- twoFactorAuth.setRecoveryCodes(newCodes)
    |-- twoFactorAuthRepository.save() --> MYSQL UPDATE
    |       StringListConverter joins: "NEWCODE1AB;NEWCODE2CD;..."
    |
    |-- securityClient.createAlert(username, "PASSWORD_CHANGED", "MEDIUM")
    |       --> "Your old backup codes will no longer work."
    |
    v
[RESPONSE 200 OK]
    | Body: {
    |   success: true,
    |   message: "Backup codes regenerated successfully",
    |   codes: ["NEWCODE1AB", "NEWCODE2CD", ...]
    | }
```

---

## TOTP Technical Deep Dive

### How TOTPUtil.java Works Internally

```
totpUtil.generateSecret()
    Uses: DefaultSecretGenerator (samstevens library)
    Output: Random Base32 string e.g. "JBSWY3DPEHPK3PXP"
    Entropy: ~80 bits (cryptographically secure)

totpUtil.getQrCodeUrl(secret, email)
    Builds "otpauth://totp/john@example.com?secret=JBSWY...&issuer=Rev-PasswordManager"
    ZxingPngQrGenerator encodes this URL as a QR code PNG
    Returns Base64 data URI → frontend renders it as <img>

totpUtil.verifyCode(secret, code)
    DefaultCodeVerifier.isValidCode(secret, submittedCode)
        1. Gets current time: System.currentTimeMillis() / 1000
        2. Computes time window: floor(time / 30)  e.g. window = 57,234,321
        3. Computes HMAC-SHA1(secretKey, window_as_8_bytes)
        4. Truncates HMAC to dynamic offset → 4 bytes
        5. Converts to integer, takes last 6 digits
        6. Compares to submitted code
        7. Also checks window-1 and window+1 (±30 sec tolerance for clock drift)
```

### Why 30 Seconds?
```
Window = floor(Unix epoch seconds / 30)
    = floor(1711213500 / 30)
    = 57040450

Same window computed on both:
    - User's phone (via app)
    - Your server (via TOTPUtil)

If clocks are within 30 seconds → same window → same code → match!
```

---

## Database Tables Used

| Table | Written By | Read By |
| :--- | :--- | :--- |
| `two_factor_auth` | `TwoFactorService` | `TwoFactorService` |
| `users` | `TwoFactorService` (sets `is_2fa_enabled`) | `TwoFactorController`, `AuthenticationService` |

### two_factor_auth Table Structure

| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | BIGINT | Primary key |
| `user_id` | BIGINT FK | References `users.id` |
| `secret_key` | VARCHAR | Base32 TOTP secret (e.g. "JBSWY3DPEHPK3PXP") |
| `is_enabled` | BOOLEAN | False until `verify-setup` succeeds |
| `recovery_codes` | VARCHAR | Semicolon-delimited list of 10 backup codes |

---

## Dependencies Summary

| Dependency | Used For |
| :--- | :--- |
| `dev.samstevens.totp` | Secret generation, QR code, TOTP verification |
| `ZXing (Zebra Crossing)` | QR image rendering inside samstevens library |
| `spring-data-jpa` | All DB reads and writes |
| `spring-cloud-openfeign` | `SecurityClient` for security alerts |
| `spring-security-crypto` | BCrypt (for password re-auth in login flow) |
| `java.security.SecureRandom` | Generating backup recovery codes |
| `Lombok` | `@RequiredArgsConstructor`, `@Builder` |

---

## Security Notes

| Risk | How It Is Mitigated |
| :--- | :--- |
| Secret key exposed | Never returned after initial setup — only the QR code is shown once |
| Backup code reuse | Each code is removed from the list after a single use |
| Clock drift between user and server | `DefaultCodeVerifier` allows ±1 time window (±30 seconds) |
| 2FA disabled without user's knowledge | HIGH severity security alert is fired immediately |
| Brute force on 6-digit code | API Gateway rate limiter (100 req/min) blocks automated attempts |

# Authentication Feature — Complete Flow Document
> Part of the `user-service` deep-dive series.

This document explains the full end-to-end flow of every Authentication feature, from the frontend HTTP request all the way through to the database — including every file involved, every dependency used, and ASCII diagrams for each flow.

---

## Files Involved in Authentication

| File | Package | Role |
| :--- | :--- | :--- |
| `AuthController.java` | `controller` | Entry point — receives all HTTP requests |
| `AuthenticationService.java` | `service/auth` | Main orchestrator for login, logout, token refresh |
| `RegistrationService.java` | `service/auth` | Handles new user registration |
| `OtpService.java` | `service/auth` | Generates and validates 6-digit OTP codes |
| `AccountRecoveryService.java` | `service/auth` | Password reset via security questions |
| `SecurityQuestionService.java` | `service/auth` | Saves and verifies security Q&A pairs |
| `SessionService.java` | `service/auth` | Tracks active sessions in the database |
| `LoginAttemptService.java` | `service/security` | Records every login attempt (success + failure) |
| `AdaptiveAuthService.java` | `service/security` | Calculates risk score based on IP, device, time |
| `DuressService.java` | `service/security` | Handles emergency "duress" login mode |
| `GeoLocationService.java` | `service/security` | Converts IP address to a readable location |
| `CaptchaService.java` | `service/security` | Validates CAPTCHA tokens after failed logins |
| `EmailService.java` | `service/email` | Sends OTP and verification emails |
| `JwtTokenProvider.java` | `security` | Generates and validates JWT tokens |
| `JwtAuthenticationFilter.java` | `security` | Validates JWT on every incoming request |
| `CustomUserDetailsService.java` | `security` | Loads user from DB for Spring Security |
| `SecurityConfig.java` | `config` | Defines which routes are protected |
| `JwtConfig.java` | `config` | Holds JWT secret key and expiry times |
| `VaultClient.java` | `client` | Feign client — calls vault-service on registration |
| `SecurityClient.java` | `client` | Feign client — logs actions, creates security alerts |
| `UserRepository.java` | `repository` | DB queries for User |
| `OtpTokenRepository.java` | `repository` | DB queries for OTP tokens |
| `UserSessionRepository.java` | `repository` | DB queries for active sessions |
| `LoginAttemptRepository.java` | `repository` | DB queries for login history |

---

## Feature 1 — User Registration

### What it does
A new user signs up by providing their email, username, master password, and 3 security questions. The system validates inputs, creates the user, sets up their vault, and sends a verification email.

### ASCII Flow Diagram

```
FRONTEND (Angular)
    |
    | POST /api/auth/register
    | Body: { email, username, masterPassword, securityQuestions[] }
    |
    v
[API GATEWAY]
    | -- Routes to user-service (no JWT needed, public route)
    v
[AuthController.java]
    | -- Receives request, triggers @Valid bean validation
    | -- RegistrationRequest: @NotBlank, @Email, @Size(min=12) all checked here
    v
[RegistrationService.java]
    |-- 1. userRepository.existsByEmail()  --> MYSQL: check duplicate email
    |-- 2. userRepository.existsByUsername() --> MYSQL: check duplicate username
    |-- 3. Check: passwordHint cannot contain masterPassword
    |-- 4. passwordEncoder.encode(masterPassword) --> BCrypt hash
    |-- 5. userRepository.save(newUser)  --> MYSQL: INSERT into users table
    |-- 6. vaultClient.createDefaultFoldersAndCategories(userId)
    |         --> HTTP call to vault-service (creates Login, Social, Payment folders)
    |-- 7. securityQuestionService.saveSecurityQuestions(user, questions)
    |         --> MYSQL: INSERT into security_questions table
    |-- 8. otpService.generateOtp(user, "EMAIL_VERIFICATION")
    |         --> 6-digit SecureRandom code
    |         --> MYSQL: INSERT into otp_tokens table (expires in 15min)
    |-- 9. emailService.sendOtpEmail(email, otpCode)
    |         --> SMTP: sends email with verification code
    v
[RESPONSE 201 Created]
    | Body: { id, email, username, createdAt }
    v
FRONTEND
```

### Dependencies Used
- `BCryptPasswordEncoder` — hashes the master password
- `SecureRandom` — generates the 6-digit OTP
- Spring `@Valid` — validates `RegistrationRequest` DTO annotations
- `VaultClient` (Feign) — HTTP call to `vault-service`
- JavaMail (`EmailService`) — sends SMTP email
- Spring Data JPA — all DB writes

---

## Feature 2 — Email Verification

### What it does
After registration, the user enters the 6-digit code from their email. The system marks their email as verified, allowing them to log in.

### ASCII Flow Diagram

```
FRONTEND
    |
    | POST /api/auth/verify-email
    | Body: { username, otpCode }
    |
    v
[AuthController.java]
    v
[RegistrationService.verifyEmail()]
    |-- 1. userRepository.findByUsername() or findByEmail()  --> MYSQL lookup
    |-- 2. Check: is email already verified? --> throw if yes
    |-- 3. otpService.validateOtp(user, otpCode, "EMAIL_VERIFICATION")
    |         --> MYSQL: find OTP token by code
    |         --> Check: same user? correct type? not used? not expired?
    |         --> Mark OTP as used --> MYSQL: UPDATE otp_tokens SET is_used=true
    |-- 4. user.setEmailVerified(true)
    |-- 5. userRepository.save(user) --> MYSQL: UPDATE users
    v
[RESPONSE 200 OK]
```

---

## Feature 3 — Login (Standard & Adaptive)

### What it does
This is the most complex flow. The user submits credentials, the system validates them, calculates a risk score based on device/IP/time, decides if extra verification is needed, then issues JWT tokens and creates a session.

### ASCII Flow Diagram

```
FRONTEND
    |
    | POST /api/auth/login
    | Body: { username, password }
    | Headers: User-Agent, X-Device-Fingerprint (optional)
    |
    v
[API GATEWAY]
    | -- Public route, no JWT check
    v
[AuthController.java]
    v
[AuthenticationService.authenticate()]
    |
    |-- STEP 1: LOCKOUT CHECK
    |   loginAttemptService.getRecentFailedAttempts(username, 15min)
    |   If >= 5 failures --> throw "Account temporarily locked"
    |   captchaService: if failures >= 3, require CAPTCHA token to be present
    |
    |-- STEP 2: CREDENTIAL VALIDATION
    |   authenticationManager.authenticate(UsernamePasswordAuthenticationToken)
    |       --> CustomUserDetailsService.loadUserByUsername()
    |           --> userRepository.findByUsername() or findByEmail() --> MYSQL
    |           --> returns UserDetails with masterPasswordHash as password
    |       --> BCrypt.matches(rawPassword, hash)
    |       --> If wrong --> throw BadCredentialsException
    |
    |-- STEP 3: EMAIL VERIFICATION CHECK
    |   If user.emailVerified == false --> throw "Email not verified"
    |
    |-- STEP 4: DEVICE & LOCATION
    |   clientIpUtil.getClientIpAddress(request)
    |   deviceFingerprintUtil.generateFingerprint(request)
    |   geoLocationService.getLocation(ipAddress)
    |       --> Converts IP to "Mumbai, India"
    |
    |-- STEP 5: ADAPTIVE RISK SCORING
    |   adaptiveAuthService.calculateRiskScore(username, ip, device, fingerprint)
    |       --> loginAttemptRepository.findByUsernameAttemptedOrderByTimestampDesc()
    |           --> MYSQL: check login history
    |       Scoring:
    |         +30 if first-ever login (no history)
    |         +40 if IP never seen before --> creates security alert via SecurityClient
    |         +25 if device fingerprint never seen before
    |         +30 if 3+ failures in last hour
    |         +20 if unusual time (e.g. 3am)
    |       Max score: 100
    |
    |-- STEP 6: IS HIGH RISK? (score >= 50)
    |   YES --> Send OTP email, return { requiresOtp: true }
    |   NO  --> Continue to token generation
    |
    |-- STEP 7: JWT TOKEN GENERATION (low risk path)
    |   jwtTokenProvider.generateAccessToken(authentication)
    |       --> JwtConfig: reads secret key from config-server
    |       --> Jwts.builder().subject(username).expiration().signWith(HMAC-SHA256)
    |       --> Returns signed JWT string
    |   jwtTokenProvider.generateRefreshToken(authentication)
    |       --> Same but longer expiry
    |
    |-- STEP 8: SESSION TRACKING
    |   sessionService.createSession(user, token, request, location, fingerprint)
    |       --> UserSession entity: ip, device, location, expiresAt(+7days)
    |       --> userSessionRepository.save() --> MYSQL: INSERT into user_sessions
    |
    |-- STEP 9: RECORD LOGIN ATTEMPT
    |   loginAttemptService.recordLoginAttempt(username, true, null, ip, device, ...)
    |       --> loginAttemptRepository.save() --> MYSQL: INSERT into login_attempts
    |
    |-- STEP 10: LOG ACTION
    |   securityClient.logAction(username, "USER_LOGIN", "...")
    |       --> HTTP call to security-service
    |
    v
[RESPONSE 200 OK]
    | Body: {
    |   accessToken: "eyJ...",
    |   refreshToken: "eyJ...",
    |   username: "john",
    |   is2faEnabled: true/false,
    |   requiresOtp: false
    | }
    v
FRONTEND
    | -- Stores accessToken in memory / localStorage
    | -- All future requests: Authorization: Bearer <token>
```

### Risk Score Table

| Condition | Points Added |
| :--- | :--- |
| First-ever login (no history) | +30 |
| New IP address never seen before | +40 |
| New device fingerprint | +25 |
| 3+ failures in the last hour | +30 |
| Login at an unusual time (late night) | +20 |
| **Total cap** | **100** |
| **Threshold for OTP requirement** | **>= 50** |

---

## Feature 4 — OTP Verification (High-Risk Login)

### What it does
When risk score ≥ 50, login is paused and an OTP is sent. The user must submit the code to complete login and receive tokens.

### ASCII Flow Diagram

```
FRONTEND
    | (received requiresOtp: true from login)
    |
    | POST /api/auth/verify-otp
    | Body: { username, otpCode }
    |
    v
[AuthController.java]
    v
[AuthenticationService.verifyLoginOtp()]
    |-- 1. userRepository.findByUsername() --> MYSQL
    |-- 2. otpService.validateOtp(user, otpCode, "LOGIN_OTP")
    |        --> Check: exists, correct user, not used, not expired(15min)
    |        --> Mark as used --> MYSQL UPDATE
    |-- 3. Generate JWT tokens (same as Step 7 in login flow)
    |-- 4. Create session (same as Step 8 in login flow)
    |-- 5. Record successful login attempt
    v
[RESPONSE 200 OK]
    | Body: { accessToken, refreshToken, ... }
```

---

## Feature 5 — 2FA Login Verification

### What it does
If the user has 2FA enabled, after credentials are validated they must also provide a 6-digit TOTP code from their authenticator app (e.g. Google Authenticator).

### ASCII Flow Diagram

```
FRONTEND
    | POST /api/auth/login
    | (credentials valid, 2FA enabled)
    |
    v
[AuthenticationService]
    |-- user.is2faEnabled() == true
    |-- Returns { requiresTwoFactor: true } WITHOUT tokens
    |
FRONTEND
    | -- Shows 2FA code input screen
    | POST /api/auth/verify-otp (with TOTP code)
    |
    v
[AuthenticationService.verifyLoginOtp()]
    |-- otpService or totpUtil.verifyCode(secret, submittedCode)
    |       --> Validates HMAC-SHA1 time-based code (30 second windows)
    |-- On success --> issue JWT + create session
    v
[RESPONSE 200 OK with tokens]
```

---

## Feature 6 — Token Refresh

### What it does
Access tokens expire (short-lived). When expired, the frontend sends the refresh token to get a new access token without requiring re-login.

### ASCII Flow Diagram

```
FRONTEND
    | (API call returns 401 Unauthorized)
    |
    | POST /api/auth/refresh-token
    | Body: { refreshToken: "eyJ..." }
    |
    v
[AuthController.java]
    v
[AuthenticationService.refreshToken()]
    |-- 1. jwtTokenProvider.validateToken(refreshToken)
    |        --> Parses and checks signature against secret key
    |        --> Checks expiry date
    |-- 2. jwtTokenProvider.getUsernameFromToken(refreshToken)
    |        --> Extracts "sub" claim from JWT payload
    |-- 3. sessionService.isSessionActive(refreshToken)
    |        --> MYSQL: check user_sessions table → is_active = true
    |-- 4. userRepository.findByUsername() --> MYSQL
    |-- 5. jwtTokenProvider.generateAccessToken() --> New access token
    v
[RESPONSE 200 OK]
    | Body: { accessToken: "eyJ...(new)" }
```

---

## Feature 7 — Logout

### What it does
Invalidates the current session so the token can never be used again, even before it expires.

### ASCII Flow Diagram

```
FRONTEND
    |
    | POST /api/auth/logout
    | Headers: Authorization: Bearer <token>
    |
    v
[AuthController.java]
    v
[AuthenticationService.logout()]
    |-- 1. Extract token from Authorization header
    |-- 2. sessionService.terminateSessionByToken(token)
    |        --> userSessionRepository.findFirstByToken()  --> MYSQL
    |        --> session.setActive(false)
    |        --> userSessionRepository.save()  --> MYSQL UPDATE
    |-- 3. securityClient.logAction(username, "USER_LOGOUT", ...)
    |        --> HTTP call to security-service
    v
[RESPONSE 200 OK]
    | Body: { message: "Logged out successfully" }
    |
FRONTEND
    | -- Clears tokens from storage
```

---

## Feature 8 — Password Recovery (Forgot Password)

### What it does
A user who forgot their master password can reset it by correctly answering their 3 security questions. All active sessions are terminated after reset.

### ASCII Flow Diagram

```
FRONTEND
    |
    | STEP 1: GET /api/auth/security-questions/{username}
    |   --> Returns the 3 question texts (without answers)
    |
    | STEP 2: POST /api/auth/verify-security-questions
    | Body: { username, securityAnswers: [{question, answer}] }
    |
    v
[AuthController.java]
    v
[AccountRecoveryService.verifySecurityQuestions()]
    |-- 1. userRepository.findByUsername() --> MYSQL
    |-- 2. securityQuestionService.verifySecurityAnswers(user, answers)
    |        --> BCrypt.matches(submittedAnswer, storedHash) for each question
    |        --> All 3 must match
    v
[RESPONSE 200 OK - questions verified]
    |
FRONTEND
    | STEP 3: POST /api/auth/reset-password
    | Body: { username, securityAnswers[], newMasterPassword }
    |
    v
[AccountRecoveryService.resetPassword()]
    |-- 1. Verify security answers again (security double-check)
    |-- 2. passwordEncoder.encode(newMasterPassword) --> BCrypt hash
    |-- 3. user.setMasterPasswordHash(newHash)
    |-- 4. userRepository.save(user)  --> MYSQL UPDATE
    |-- 5. sessionService.terminateAllUserSessions(username)
    |        --> MYSQL: UPDATE all sessions SET is_active=false
    v
[RESPONSE 200 OK]
```

---

## Feature 9 — Duress Login

### What it does
The user logs in with a special "duress password" (set in advance) when they are under coercion. The system issues a real JWT but marks it with a `duress: true` claim. The vault then shows fake/decoy data instead of the real vault.

### ASCII Flow Diagram

```
FRONTEND
    |
    | POST /api/auth/duress-login
    | Body: { username, duressPassword }
    |
    v
[AuthController.java]
    v
[AuthenticationService.duressLogin() / DuressService]
    |-- 1. Validate duress password (BCrypt match against duress hash)
    |-- 2. jwtTokenProvider.generateDuressToken(authentication)
    |        --> Same as regular JWT but adds claim: { "duress": true }
    |-- 3. Create session normally
    v
[RESPONSE 200 OK]
    | Body: { accessToken: "eyJ...(duress marked)" }
    |
FRONTEND --> vault-service
    | --> vault-service reads X-Is-Duress header set by api-gateway
    | --> DuressService.generateDummyVault() returns fake entries
    | --> Real vault is never touched
```

---

## Feature 10 — Every Request (JWT Validation Filter)

### What it does
On **every** protected API request, before reaching any controller, the JWT filter validates the token and populates the Spring Security context.

### ASCII Flow Diagram

```
FRONTEND
    |
    | ANY request to /api/users/**, /api/2fa/**, /api/sessions/**
    | Headers: Authorization: Bearer eyJ...
    |
    v
[JwtAuthenticationFilter.java]   <-- Spring Security Filter Chain
    |-- 1. Extract token from "Authorization: Bearer ..." header
    |-- 2. jwtTokenProvider.validateToken(token)
    |        --> Checks HMAC-SHA256 signature using secret from JwtConfig
    |        --> Checks expiry date
    |-- 3. sessionService.isSessionActive(token)
    |        --> MYSQL: SELECT from user_sessions WHERE token=? AND is_active=true
    |        --> If session was manually logged out → rejects here
    |-- 4. jwtTokenProvider.getUsernameFromToken(token)
    |-- 5. customUserDetailsService.loadUserByUsername(username)
    |        --> MYSQL: SELECT from users table
    |-- 6. SecurityContextHolder.setAuthentication(...)
    |        --> Spring now knows who the user is for this request
    |
    v
[Controller / Service]
    | -- Can now call SecurityContextHolder.getContext().getAuthentication().getName()
    | -- Returns the authenticated username
```

---

## Dependencies Summary

| Dependency | Used For |
| :--- | :--- |
| `spring-boot-starter-security` | SecurityConfig, Filter Chain |
| `spring-security-crypto` | BCryptPasswordEncoder |
| `jjwt` (io.jsonwebtoken) | JWT generation & validation |
| `spring-data-jpa` | All database operations |
| `spring-boot-starter-mail` | Sending OTP emails |
| `spring-cloud-openfeign` | VaultClient, SecurityClient HTTP calls |
| `dev.samstevens.totp` | TOTP 2FA code generation/verification |
| `Jakarta Validation` | @Valid, @NotBlank, @Size, @Email on DTOs |
| `Lombok` | @RequiredArgsConstructor, @Builder, @Slf4j |
| `Config Server` | JWT secret, expiration values via JwtConfig |

---

## Database Tables Used

| Table | Written By | Read By |
| :--- | :--- | :--- |
| `users` | `RegistrationService` | All services |
| `otp_tokens` | `OtpService` | `OtpService` |
| `user_sessions` | `SessionService` | `SessionService`, `JwtAuthenticationFilter` |
| `login_attempts` | `LoginAttemptService` | `LoginAttemptService`, `AdaptiveAuthService` |
| `security_questions` | `SecurityQuestionService` | `SecurityQuestionService` |
| `two_factor_auth` | `TwoFactorService` | `TwoFactorService` |

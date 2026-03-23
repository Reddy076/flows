# Background & Automated Features — Complete Flow Document
> Part of the `user-service` deep-dive series.

This document explains every **background** and **automated** mechanism in the
`user-service` — things that run silently without any direct HTTP request from
the frontend. These are the invisible engines that keep the application secure,
clean, and observable.

---

## Overview — What Runs in the Background?

```
user-service background mechanisms:

┌─────────────────────────────────────────────────────────┐
│  1. AccountDeletionScheduler   → Runs at midnight daily  │
│     Permanently deletes accounts scheduled for deletion  │
│                                                         │
│  2. LoggingAspect              → Runs on EVERY method    │
│     AOP: auto-logs all method entry/exit and exceptions  │
│                                                         │
│  3. AdaptiveAuthService        → Runs on every LOGIN     │
│     Calculates risk score from login history + patterns  │
│                                                         │
│  4. LoginAttemptService        → Runs on every LOGIN     │
│     Records every success/failure to login_attempts DB   │
│                                                         │
│  5. JwtAuthenticationFilter    → Runs on EVERY REQUEST   │
│     Validates JWT + DB session check on every HTTP call  │
└─────────────────────────────────────────────────────────┘
```

---

## Background Feature 1 — Account Deletion Scheduler

### Files Involved

| File | Package | Role |
| :--- | :--- | :--- |
| `AccountDeletionScheduler.java` | `scheduler` | The scheduled task — executes automatically at midnight |
| `UserRepository.java` | `repository` | Queries for users whose deletion date has passed |
| `@EnableScheduling` | (on main app class) | Activates Spring's scheduling engine at startup |

### What it does
Runs **every night at midnight (00:00:00)** without any human trigger.
Finds all user accounts where `deletion_scheduled_at` is in the past,
then permanently deletes them from the database.

### How it is triggered

```
NOBODY triggers this — Spring's TaskScheduler fires it automatically.
The @Scheduled annotation tells Spring: "run this method at this cron expression."

Cron: "0 0 0 * * ?"
       │ │ │ │ │ │
       │ │ │ │ │ └─ Any day of week
       │ │ │ │ └─── Any month
       │ │ │ └───── Any day of month
       │ │ └─────── Hour = 0 (midnight)
       │ └───────── Minute = 0
       └─────────── Second = 0
```

### ASCII Flow Diagram

```
[SPRING TASK SCHEDULER] — 00:00:00 every day
    |
    | Triggers: @Scheduled(cron = "0 0 0 * * ?")
    v
[AccountDeletionScheduler.deleteExpiredAccounts()]
    |
    |-- LOG: "Running scheduled account deletion task..."
    |
    |-- userRepository.findByDeletionScheduledAtBefore(LocalDateTime.now())
    |       --> MYSQL: SELECT * FROM users
    |                  WHERE deletion_scheduled_at IS NOT NULL
    |                  AND deletion_scheduled_at < NOW()
    |
    |-- Returns: List<User> (could be 0, 1, or many users)
    |
    |-- For each expired user:
    |       LOG: "Permanently deleting user: john"
    |       userRepository.delete(user)
    |           --> MYSQL: DELETE FROM users WHERE id = ?
    |           --> Cascade deletes (JPA @OnDelete or orphanRemoval):
    |               - user_sessions    (all sessions)
    |               - login_attempts   (all login history)
    |               - otp_tokens       (all OTP records)
    |               - security_questions (all Q&A pairs)
    |               - two_factor_auth  (2FA secret + backup codes)
    |               - user_settings    (@OneToOne → deleted with user)
    |
    |-- If any users were deleted:
    |       LOG: "Deleted N expired accounts."
    |
    v
[DONE — sleeps until next midnight]
```

### User Perspective

```
DAY 0:  User requests account deletion
            → deletionScheduledAt = NOW() + 30 days

DAYS 1-29:  User can still log in, cancel deletion at any time
            → JwtAuthenticationFilter still grants access
            → /api/users/account/cancel-deletion resets the timestamp to null

DAY 30:    AccountDeletionScheduler runs at midnight
            → finds user in query (deletionScheduledAt < NOW())
            → permanently deleted — cannot be recovered
```

---

## Background Feature 2 — AOP Logging Aspect

### Files Involved

| File | Package | Role |
| :--- | :--- | :--- |
| `LoggingAspect.java` | `aspect` | Auto-intercepting logger via Spring AOP |
| `@EnableAspectJAutoProxy` | (auto-enabled by Spring Boot) | Activates AOP proxy weaving |

### What it does
An **Aspect-Oriented Programming (AOP)** component that silently wraps every method
in every `@Repository`, `@Service`, and `@RestController` class. It auto-logs:
- Method entry with arguments (in DEBUG mode)
- Method exit with return value (in DEBUG mode)
- Every exception — automatically, without any `try/catch` in business code

### How AOP Works (Conceptually)

```
NORMAL code (without AOP):
    AuthenticationService.authenticate() {
        // business logic only
    }

WITH AOP (what actually runs):
    LoggingAspect intercepts → logs "Enter: authenticate() with args=[...]"
        → AuthenticationService.authenticate() runs
        → if exception: LoggingAspect logs "Exception in authenticate() with cause=..."
    LoggingAspect intercepts → logs "Exit: authenticate() with result=..."
```

The business classes themselves have **no logging boilerplate** — the aspect adds it
transparently to all of them at once.

### ASCII Flow Diagram

```
[ANY HTTP REQUEST RECEIVED]
    |
    v
[Spring AOP Proxy (invisible wrapper around every bean)]
    |
    |-- LoggingAspect.logAround() enters
    |       Pointcut 1: within(@Repository *) OR within(@Service *) OR within(@RestController *)
    |       Pointcut 2: within(com.revature.user..*)
    |       Both must match → aspect activates
    |
    |-- IF log.isDebugEnabled():
    |       log.debug("Enter: UserService.updateProfile() with argument[s] = [john, UpdateProfileRequest{...}]")
    |
    |-- joinPoint.proceed()
    |       ← the actual method executes here →
    |
    |-- IF log.isDebugEnabled():
    |       log.debug("Exit: UserService.updateProfile() with result = UserResponse{...}")
    |
    |-- IF exception thrown:
    |   LoggingAspect.logAfterThrowing() catches it
    |       log.error("Exception in UserService.updateProfile() with cause = ConstraintViolationException")
    |       (the exception is NOT swallowed — it re-throws normally after logging)
    v
[RESPONSE returned to controller / exception propagated]
```

### Which Classes Are Covered

```
COVERED (both pointcuts match):
    ✅ All @Service classes       (AuthenticationService, UserService, SessionService, etc.)
    ✅ All @Repository classes    (UserRepository, UserSessionRepository, etc.)
    ✅ All @RestController classes (UserController, AuthController, etc.)

NOT COVERED:
    ❌ @Component classes         (filters, utils, config)
    ❌ Classes outside com.revature.user.*  (Spring Security internals, Feign, etc.)
```

### Log Levels

```
DEBUG level (only in dev/test — controlled by application.yml logging config):
    → Method entry with all arguments
    → Method exit with return value

ERROR level (always active, even in production):
    → Every exception with class name and cause
    → Every IllegalArgumentException with the bad argument values
```

---

## Background Feature 3 — Adaptive Risk Scoring (per login)

### Files Involved

| File | Package | Role |
| :--- | :--- | :--- |
| `AdaptiveAuthService.java` | `service/security` | Calculates risk score from login history patterns |
| `LoginAttemptRepository.java` | `repository` | Queries all historical login attempts for the user |
| `DateTimeUtil.java` | `util` | Checks whether current time is "unusual" (late night) |
| `SecurityClient.java` | `client` | Fires "New IP login" alert when unknown IP detected |

### What it does
Runs **automatically on every login attempt**, invisible to the user. Analyses login
history to decide if this login is suspicious — and if so, requires OTP before
granting tokens.

### ASCII Flow Diagram

```
[Login attempt arrives: POST /api/auth/login]
    |
    v
[AuthenticationService] → after credentials pass BCrypt check
    |
    v
[AdaptiveAuthService.calculateRiskScore(username, ip, deviceInfo, fingerprint)]
    |
    |-- FETCH FULL LOGIN HISTORY:
    |   loginAttemptRepository.findByUsernameAttemptedOrderByTimestampDesc(username)
    |       --> MYSQL: SELECT * FROM login_attempts
    |                  WHERE username_attempted = 'john'
    |                  ORDER BY timestamp DESC
    |
    |-- START: riskScore = 0
    |
    |-- CHECK 1: Is this the first-ever login?
    |   recentAttempts.isEmpty() == true?
    |       YES → riskScore += 30  (no history to compare against)
    |       NO  → continue with checks below
    |
    |-- CHECK 2: Is this IP address new?
    |   attempts.stream().anyMatch(a -> a.ipAddress.equals(currentIp))
    |       Not found → riskScore += 40
    |       Also: securityClient.createAlert("NEW_LOCATION_LOGIN", "MEDIUM")
    |               --> HTTP call to security-service (non-blocking try/catch)
    |
    |-- CHECK 3: Is this device fingerprint new?
    |   attempts.stream().anyMatch(a -> a.deviceInfo.contains(fingerprint))
    |       Not found → riskScore += 25
    |
    |-- CHECK 4: Recent failed attempts?
    |   attempts.filter(a -> !a.successful && a.timestamp.isAfter(now - 1 hour))
    |       count >= 3 → riskScore += 30
    |
    |-- CHECK 5: Unusual login time?
    |   dateTimeUtil.isUnusualTime(LocalDateTime.now())
    |       e.g. between 1am and 5am → riskScore += 20
    |
    |-- APPLY CAP: Math.min(100, riskScore)
    |
    v
[RESULT: riskScore = 0-100]
    |
    |-- requiresAdditionalVerification(riskScore):
    |       return riskScore >= 50
    |
    |-- IF score < 50 → JWT issued immediately (low risk login)
    |-- IF score >= 50 → OTP sent, login paused, { requiresOtp: true } returned
```

### Risk Score Table

| Condition | Points |
| :--- | :--- |
| First-ever login (no history at all) | +30 |
| IP address never seen before | +40 |
| Device fingerprint never seen before | +25 |
| 3+ failed attempts in last hour | +30 |
| Unusual login time (e.g. 2am) | +20 |
| **Maximum possible score** | **100** |
| **OTP threshold** | **≥ 50** |

---

## Background Feature 4 — Login Attempt Recording (per login)

### Files Involved

| File | Package | Role |
| :--- | :--- | :--- |
| `LoginAttemptService.java` | `service/security` | Records every login (success + failure) to DB |
| `LoginAttemptRepository.java` | `repository` | Writes and reads `login_attempts` table |

### What it does
Every login attempt — whether it succeeds or fails — is recorded in the
`login_attempts` table **automatically** by `AuthenticationService`. This data
feeds both the Adaptive Risk Scoring (Feature 3) and the Login History view.

### ASCII Flow Diagram

```
[POST /api/auth/login — credentials FAIL]
    |
    v
[AuthenticationService]
    |-- BadCredentialsException thrown
    |-- loginAttemptService.recordLoginAttempt(
    |       username: "john",
    |       successful: false,
    |       failureReason: "Invalid credentials",
    |       ipAddress: "192.168.1.100",
    |       deviceInfo: "Chrome on Windows",
    |       location: "Mumbai, India",
    |       riskScore: 40
    |   )
    v
[LoginAttemptService.recordLoginAttempt()]
    |
    |-- userRepository.findByUsername("john") → MYSQL SELECT
    |   (user may be null if username doesn't exist — still recorded!)
    |
    |-- LoginAttempt.builder()
    |       .user(userOrNull)
    |       .usernameAttempted("john")  ← always stored, even if user not found
    |       .ipAddress("192.168.1.100")
    |       .deviceInfo("Chrome on Windows")
    |       .location("Mumbai, India")
    |       .successful(false)
    |       .failureReason("Invalid credentials")
    |       .riskScore(40)
    |       .timestamp(LocalDateTime.now())
    |       .build()
    |
    |-- loginAttemptRepository.save(attempt)
    |       --> MYSQL INSERT INTO login_attempts (...)
    |
    v
[DONE — attempt recorded, exception re-thrown to controller]

---

[POST /api/auth/login — credentials PASS]
    |
    v
[AuthenticationService — after JWT issued + session created]
    |-- loginAttemptService.recordLoginAttempt(
    |       successful: true,
    |       failureReason: null
    |   )
    v
[MYSQL INSERT — same flow as above, successful = true]
```

### Why record even failed attempts for non-existent users?

```
If attacker uses username "admin" (which doesn't exist):
    → user = null (user not found in DB)
    → LoginAttempt is still saved with usernameAttempted = "admin"
    → getRecentFailedAttempts("admin", 15) → still counts correctly
    → Lockout still works even for usernames that don't exist
    → Prevents username enumeration via timing attack
```

### How it feeds the lockout check

```
EVERY login → LoginAttemptService.getRecentFailedAttempts(username, 15min)
    --> MYSQL: SELECT COUNT(*) FROM login_attempts
               WHERE username_attempted = ?
               AND successful = false
               AND timestamp > NOW() - 15 minutes
    --> If count >= 5 → AuthenticationService throws "Account temporarily locked"
    --> If count >= 3 → CAPTCHA required
```

---

## Background Feature 5 — JWT Filter (per request, every time)

### Files Involved

| File | Package | Role |
| :--- | :--- | :--- |
| `JwtAuthenticationFilter.java` | `security` | Intercepts every HTTP request before the controller |
| `JwtTokenProvider.java` | `security` | Validates JWT signature + extracts claims |
| `SessionService.java` | `service/auth` | Checks if the session is still active in MySQL |
| `CustomUserDetailsService.java` | `security` | Loads UserDetails from MySQL for Spring Security |

### What it does
Automatically runs on **every single HTTP request** that reaches the user-service
(even before your controller method is called). It's a Spring Security filter —
you don't call it, Spring calls it for you.

### ASCII Flow Diagram

```
[ANY HTTP REQUEST → user-service]
    |
    v
[Spring Security Filter Chain]
    |
    v
[JwtAuthenticationFilter.doFilterInternal()] ← runs BEFORE controller
    |
    |-- Extract Authorization header
    |   If null or not "Bearer ..." → skip (let request continue unauthenticated)
    |
    |-- jwt = authHeader.substring(7)
    |
    |-- GATE 1: jwtTokenProvider.validateToken(jwt)
    |       → Jwts.parser().verifyWith(secret).parseSignedClaims(jwt)
    |       → Checks HMAC-SHA256 signature
    |       → Checks expiry date
    |       If fails → authentication NOT set (request gets 401 from SecurityConfig)
    |
    |-- GATE 2: sessionService.isSessionActive(jwt)
    |       → userSessionRepository.findFirstByTokenOrderByIdDesc(jwt)
    |           --> MYSQL SELECT (1 query per request)
    |       → session.isActive() == true?
    |       If false → authentication NOT set (logged-out session rejected immediately)
    |
    |-- BOTH GATES PASS:
    |   username = jwtTokenProvider.getUsernameFromToken(jwt)
    |   userDetails = customUserDetailsService.loadUserByUsername(username)
    |       --> MYSQL SELECT FROM users WHERE username = ?
    |   authToken = UsernamePasswordAuthenticationToken(userDetails, jwt, authorities)
    |   SecurityContextHolder.setAuthentication(authToken)
    |       → Spring now knows who this request belongs to
    |
    |-- filterChain.doFilter() → passes request to next filter / controller
    v
[Controller can now call SecurityContext.getAuthentication().getName()]
```

---

## How All 5 Background Features Interact

```
A single login attempt triggers features 3, 4, and 5 together:

POST /api/auth/login
    │
    ├─→ JwtAuthenticationFilter (Feature 5)  ← public route, skips JWT check
    │
    ├─→ AuthenticationService validates credentials
    │
    ├─→ AdaptiveAuthService.calculateRiskScore() (Feature 3)
    │       Reads ALL historical login_attempts from DB
    │       Returns riskScore = 65 (HIGH)
    │       Creates "NEW_LOCATION_LOGIN" alert via SecurityClient
    │
    ├─→ OTP required → return { requiresOtp: true }
    │
    └─→ LoginAttemptService.recordLoginAttempt() (Feature 4)
            Writes this attempt to login_attempts table
            (successful=false because OTP not yet verified)

Meanwhile, at midnight every night:
    AccountDeletionScheduler (Feature 1) deletes expired accounts

On every method call in every service:
    LoggingAspect (Feature 2) logs entry/exit/exceptions automatically
```

---

## Database Tables Used

| Background Feature | Table | Operation | When |
| :--- | :--- | :--- | :--- |
| AccountDeletionScheduler | `users` | DELETE | 00:00:00 daily |
| AdaptiveAuthService | `login_attempts` | SELECT ALL | Every login |
| LoginAttemptService | `login_attempts` | INSERT | Every login attempt |
| LoginAttemptService | `users` | SELECT | Every login attempt |
| JwtAuthenticationFilter | `user_sessions` | SELECT (1 row) | Every HTTP request |
| JwtAuthenticationFilter | `users` | SELECT (1 row) | Every HTTP request |
| LoggingAspect | *(none)* | — | Every method call |

---

## Dependencies Summary

| Dependency | Used By |
| :--- | :--- |
| `spring-scheduling` | `@Scheduled` in `AccountDeletionScheduler` |
| `spring-aop` + `aspectjweaver` | `@Aspect`, `@Around`, `@AfterThrowing` in `LoggingAspect` |
| `spring-security` | `JwtAuthenticationFilter`, `SecurityContextHolder` |
| `jjwt` | `JwtTokenProvider.validateToken()` in filter |
| `spring-data-jpa` | All background DB reads and writes |
| `spring-cloud-openfeign` | `SecurityClient` alert in `AdaptiveAuthService` |
| `slf4j` / `logback` | All logging in `LoggingAspect`, `LoginAttemptService` |

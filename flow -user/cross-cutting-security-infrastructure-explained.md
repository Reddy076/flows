# Cross-Cutting / Security Infrastructure — Complete Flow Document
> Part of the `user-service` deep-dive series.

This document explains every **cross-cutting security mechanism** in the `user-service`.
These classes don't belong to a single feature — they apply to **every request, every
login, and every method call** across the entire service.

---

## Overview — The 11 Cross-Cutting Components

```
Every HTTP Request passes through:
┌────────────────────────────────────────────────┐
│  CorsConfig              → Is this origin allowed?         │
│  RateLimitGlobalFilter   → Is this IP rate-limited?        │
│  JwtAuthFilter (gateway) → Is the JWT signature valid?     │
│  JwtAuthenticationFilter → DB session still active?        │
│  LoggingAspect           → Auto-log every method call      │
└────────────────────────────────────────────────┘

Every Login attempt additionally runs:
┌────────────────────────────────────────────────┐
│  ClientIpUtil         → What is the real IP?              │
│  DeviceFingerprintUtil→ What device is this?              │
│  GeoLocationService   → Where is this IP from?            │
│  AdaptiveAuthService  → What is the risk score?           │
│  CaptchaService       → Is CAPTCHA required?              │
│  LoginAttemptService  → Record the attempt to DB          │
└────────────────────────────────────────────────┘

Token operations use:
┌────────────────────────────────────────────────┐
│  JwtConfig            → Secret key + expiry config        │
│  JwtTokenProvider     → Generate & validate JWTs          │
│  OpenApiConfig        → Swagger UI auth integration       │
└────────────────────────────────────────────────┘
```

---

## Component 1 — JwtConfig

### File: `config/JwtConfig.java`

### What it does
A simple `@ConfigurationProperties` bean that reads JWT settings from the
**Config Server** (not hardcoded in the application). All other JWT components
inject `JwtConfig` to get the secret and expiry durations.

```java
@Configuration
@ConfigurationProperties(prefix = "jwt")   // reads jwt.* from config-server
@Data
public class JwtConfig {
    private String secret;                  // Base64-encoded HMAC-SHA256 key
    private long accessTokenExpiration;     // milliseconds (e.g. 900000 = 15 min)
    private long refreshTokenExpiration;    // milliseconds (e.g. 604800000 = 7 days)
}
```

### Config Server YAML (what it reads from):

```yaml
jwt:
  secret: 5367566B59703373367639792F423F4528482B4D6251655468576D5A71347437
  access-token-expiration: 900000      # 15 minutes
  refresh-token-expiration: 604800000  # 7 days
```

### Why Config Server (not application.yml)?

```
The JWT secret is the MASTER KEY of the entire authentication system.
If it is in application.yml → it ends up in Git → publicly exposed!
Config Server allows: separate secrets per environment (dev/staging/prod)
                       rotation of secrets without code deployments
                       centralized secret management
```

---

## Component 2 — JwtTokenProvider

### File: `security/JwtTokenProvider.java`

### What it does
Creates and validates all JWT tokens in the system. Has 4 token generation methods
and 4 reading/validation methods.

### Token Structure

```
A JWT has 3 parts separated by dots: header.payload.signature

Header:  { "alg": "HS256", "typ": "JWT" }
Payload: {
    "sub": "john",              ← username (subject)
    "iat": 1711213500,          ← issued at (Unix timestamp)
    "exp": 1711214400,          ← expiry (iat + accessTokenExpiration)
    "duress": true              ← ONLY on duress tokens (optional extra claim)
}
Signature: HMAC-SHA256(base64(header) + "." + base64(payload), secretKey)
```

### Methods Explained

```
generateAccessToken(authentication)
    → Short-lived token. No extra claims. Used for all API calls.
    → Expiry: from JwtConfig.accessTokenExpiration (15 min)

generateRefreshToken(authentication)
    → Same structure, longer expiry (7 days).
    → Only used to get new access tokens, not for API calls.

generateDuressToken(authentication)
    → Same as access token BUT adds: { "duress": true } to payload
    → vault-service sees this claim and returns fake/decoy vault data

validateToken(String token)
    → Jwts.parser().verifyWith(secret).parseSignedClaims(token)
    → Returns TRUE only if: signature valid AND not expired
    → Swallows all exceptions and returns false (never throws)
    → Exception types caught:
        MalformedJwtException  → tampered or invalid structure
        ExpiredJwtException    → token past expiry date
        UnsupportedJwtException→ wrong JWT type
        IllegalArgumentException → null/empty token
        JwtException           → any other JWT error

getUsernameFromToken(String token)
    → extractClaim(token, Claims::getSubject)
    → Returns the "sub" claim (username string)

isDuressToken(String token)
    → Reads the "duress" boolean claim
    → Returns false if claim absent or on any error

getExpirationDateFromToken(String token)
    → Returns the "exp" claim as java.util.Date
```

### ASCII Flow: Token Generation

```
[AuthenticationService - after successful login]
    |
    v
[JwtTokenProvider.generateAccessToken(authentication)]
    |
    |-- authentication.getPrincipal() → UserDetails (loaded from DB)
    |-- generateToken(emptyMap, userDetails, jwtConfig.getAccessTokenExpiration())
    |       jwtConfig.getSecret() → Base64-encoded secret string
    |       Decoders.BASE64.decode(secret) → byte[]
    |       Keys.hmacShaKeyFor(keyBytes) → SecretKey (HMAC-SHA256)
    |
    |-- Jwts.builder()
    |       .claims({})                         ← no extra claims for access token
    |       .subject("john")                    ← from userDetails.getUsername()
    |       .issuedAt(new Date(now))
    |       .expiration(new Date(now + 900000)) ← now + 15 minutes
    |       .signWith(secretKey)                ← HMAC-SHA256 signature
    |       .compact()                          ← serialize to "eyJ..." string
    v
Returns: "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJqb2huIiwiaWF0Ij..."
```

---

## Component 3 — JwtAuthenticationFilter

### File: `security/JwtAuthenticationFilter.java`

### What it does
A Spring Security filter that runs **before every controller method**. It is the
gatekeeper — if it doesn't populate the `SecurityContext`, the request will be
rejected as 401 Unauthorized.

### ASCII Flow

```
[ANY HTTP REQUEST → user-service]
    |
    v
[JwtAuthenticationFilter.doFilterInternal()]  ← Spring calls this automatically
    |
    |-- Extract "Authorization" header
    |   Null or not "Bearer ..." → skip (pass through un-authenticated)
    |
    |-- jwt = header.substring(7)  ← strips "Bearer "
    |
    |-- GATE 1: jwtTokenProvider.validateToken(jwt)
    |       Validates HMAC-SHA256 signature against JwtConfig.secret
    |       Checks token has not expired
    |       IF FAILS → pass through (SecurityConfig will reject with 401)
    |
    |-- GATE 2: sessionService.isSessionActive(jwt)
    |       userSessionRepository.findFirstByTokenOrderByIdDesc(token)
    |           → MYSQL SELECT FROM user_sessions WHERE token = ?
    |       session.isActive() == true?
    |       IF FALSE → pass through (logged out or revoked session)
    |
    |-- BOTH GATES PASS, SecurityContext is empty:
    |   username = jwtTokenProvider.getUsernameFromToken(jwt)
    |   userDetails = userDetailsService.loadUserByUsername(username)
    |       → MYSQL SELECT FROM users WHERE username = ?
    |   authToken = UsernamePasswordAuthenticationToken(userDetails, jwt, authorities)
    |   SecurityContextHolder.setAuthentication(authToken)
    |       ← Spring Security now knows who this request is
    |
    v
[Passes to next filter → controller]
```

### Why TWO JWT checks (Gateway + user-service)?

```
API Gateway JwtAuthFilter:
    ✅ First line of defence — rejects bad tokens immediately
    ✅ Adds X-User-Name header to forward username downstream
    ❌ Does NOT check DB session (no DB access at gateway level)

user-service JwtAuthenticationFilter:
    ✅ Second validation (defence in depth)
    ✅ Checks DB session → enables immediate revocation on logout
    ✅ Loads full UserDetails from DB for Spring Security context
```

---

## Component 4 — ClientIpUtil

### File: `util/ClientIpUtil.java`

### What it does
Extracts the **real client IP address** from the incoming request. This is non-trivial
because in a production environment, requests pass through load balancers and proxies
that mask the original IP.

### IP Header Priority Order

```
[BROWSER] → [Load Balancer] → [API Gateway] → [user-service]

Without proxy headers: request.getRemoteAddr() = Load Balancer's IP ← WRONG!
With proxy headers:    X-Forwarded-For: "203.0.113.42, 10.0.0.1"
                                          ↑ real client   ↑ proxy

ClientIpUtil checks these headers in priority order:
    1. X-Forwarded-For            (most common, set by most proxies)
    2. X-Real-IP                  (set by Nginx)
    3. Proxy-Client-IP            (set by Apache)
    4. WL-Proxy-Client-IP         (WebLogic)
    5. HTTP_X_FORWARDED_FOR
    6. HTTP_X_FORWARDED
    7. HTTP_X_CLUSTER_CLIENT_IP
    8. HTTP_CLIENT_IP
    9. HTTP_FORWARDED_FOR
    10. HTTP_FORWARDED
    11. REMOTE_ADDR (last resort — request.getRemoteAddr())

X-Forwarded-For may contain multiple IPs (e.g. "203.0.113.42, 10.0.0.1, 172.16.0.1")
→ ClientIpUtil takes ip.split(",")[0].trim() → always first = original client
```

### isLocalhost() — Private Network Detection

```
Returns true for:
    127.0.0.1              (IPv4 loopback)
    ::1 / 0:0:0:0:0:0:0:1  (IPv6 loopback)
    192.168.*.*            (RFC 1918 private)
    10.*.*.*               (RFC 1918 private)
    172.16-31.*.*          (RFC 1918 private)
    fc00: / fe80:          (IPv6 private)

Used by GeoLocationService → returns "Local Network" instead of fake city
```

---

## Component 5 — DeviceFingerprintUtil

### File: `util/DeviceFingerprintUtil.java`

### What it does
Generates a short **device identifier** from request headers. The fingerprint is
used to detect when the same user logs in from a new/unknown device — which adds
+25 to the risk score.

### How the Fingerprint is Generated

```java
// Input: 3 request headers concatenated with "|" separator
String raw = userAgent    + "|" +   // "Mozilla/5.0 (Windows NT 10.0) Chrome/121"
             acceptLang   + "|" +   // "en-US,en;q=0.9"
             acceptEncoding;         // "gzip, deflate, br"

// e.g: "Mozilla/5.0 (Windows)|en-US|gzip, deflate"

// Output: SHA-256 hash of raw string, Base64url-encoded (no padding)
MessageDigest.getInstance("SHA-256").digest(raw.getBytes(UTF_8))
→ Base64.getUrlEncoder().withoutPadding().encodeToString(hash)

// e.g: "aB3xKp9-mN2qR7sT4wVy8uL1oQ6jH0cF5nG"  (43 chars, URL-safe)
```

### Limitations (Current Implementation)

```
GOOD: Quick, deterministic, no client-side JS needed
LIMITATION: Only uses 3 HTTP headers → not truly unique across devices
            If same browser/OS → same fingerprint even on different machines
            If VPN changes User-Agent → different fingerprint on same machine

For production-grade fingerprinting you'd add:
    - Canvas fingerprint (JS)
    - Time zone
    - Screen resolution
    - Font list
```

---

## Component 6 — GeoLocationService

### File: `service/security/GeoLocationService.java`

### What it does
Converts an IP address to a human-readable location string like `"Mumbai, India"`.
This location is stored in `login_attempts` and `user_sessions` tables.

### Current Implementation (Simulated)

```java
public String getLocationFromIp(String ipAddress) {
    if (clientIpUtil.isLocalhost(ipAddress))   → return "Local Network"
    if (ipAddress == null or "unknown")         → return "Unknown"
    // Production would call a real GeoIP API here (e.g. MaxMind GeoLite2)
    return "City, Country (Simulated)"
}
```

```
Current state: Simulated — always returns "City, Country (Simulated)" for public IPs
Production extension: Would query MaxMind GeoLite2 database or ip-api.com
                      to return real city/country from IP
```

---

## Component 7 — AdaptiveAuthService

### File: `service/security/AdaptiveAuthService.java`

### What it does
Calculates a **risk score 0-100** on every login by analysing the user's login history.
Automatically triggers OTP verification when the score reaches 50.

### Risk Scoring Logic

```
Input: (username, ipAddress, deviceInfo, deviceFingerprint)

Fetch: ALL historical login_attempts for this username (no time limit)

riskScore = 0

IF no history at all:           riskScore += 30  (first login ever)

ELSE compare to history:
    IF ipAddress never seen:    riskScore += 40  (new location)
                                → also fires "NEW_LOCATION_LOGIN" security alert
    IF fingerprint never seen:  riskScore += 25  (new device)
    IF 3+ failures in last 1hr: riskScore += 30  (brute force pattern)
    IF isUnusualTime():         riskScore += 20  (odd login hour)

riskScore = Math.min(100, riskScore)   ← cap at 100

requiresAdditionalVerification(score): return score >= 50
```

---

## Component 8 — CaptchaService

### File: `service/security/CaptchaService.java`

### What it does
Decides if a CAPTCHA token is required before a login attempt is processed.
Requires CAPTCHA after a configurable number of failed attempts in the last 30 minutes.

### ASCII Flow

```
[POST /api/auth/login received]
    |
    v
[AuthenticationService]
    |
    |-- CaptchaService.isCaptchaRequired(username)
    |       IF captchaConfig.isEnabled() == false → return false (skip in dev)
    |       loginAttemptService.getRecentFailedAttempts(username, 30min)
    |           → MYSQL: COUNT(*) FROM login_attempts
    |                    WHERE username = ? AND successful = false
    |                    AND timestamp > NOW() - 30 minutes
    |       return count >= captchaConfig.getFailedAttemptsThreshold()
    |           (e.g. threshold = 3)
    |
    |-- IF required AND captchaToken is null:
    |       throw exception → "CAPTCHA required"
    |
    |-- CaptchaService.verifyCaptcha(captchaToken)
    |       IF captchaConfig.isEnabled() == false → return true
    |       IF token is null or blank → return false
    |       [Currently: accepts any non-blank token]
    |       [Production: POST to Google reCAPTCHA /siteverify API]
```

### Note on Current Implementation

```
The current verifyCaptcha() accepts any non-blank token:
    return true;  ← always passes if token is present

Production would call:
    POST https://www.google.com/recaptcha/api/siteverify
         ?secret=<serverSecret>
         &response=<captchaToken>
    → Parse JSON: { "success": true/false }
```

---

## Component 9 — LoginAttemptService

### File: `service/security/LoginAttemptService.java`

### What it does
Records every login attempt to the `login_attempts` table and provides
query methods used by both lockout logic and adaptive risk scoring.

### Key Methods

```
recordLoginAttempt(username, successful, failureReason, ip, device, location, riskScore)
    → Writes one row to login_attempts table
    → User link is NULLABLE (still records if username doesn't exist)
    → Wrapped in try/catch → never throws (logging failure must not block login)

getRecentFailedAttempts(username, minutes)
    → MYSQL: COUNT WHERE username = ? AND successful = false AND timestamp > now-N-min
    → Used by: AuthenticationService (lockout), CaptchaService (threshold)

getLoginHistory(username)
    → MYSQL: SELECT all for user ORDER BY timestamp DESC
    → Exposed via GET /api/users/login-history endpoint

isNewDevice(username, deviceInfo)
    → MYSQL: EXISTS WHERE user_id = ? AND deviceInfo = ? AND successful = true
    → Returns true if device has NEVER had a successful login
```

---

## Component 10 — LoggingAspect

### File: `aspect/LoggingAspect.java`

### What it does
An AOP (Aspect-Oriented Programming) component that auto-wraps every method in
every `@Repository`, `@Service`, and `@RestController` to add logging with
**zero boilerplate** in the actual business classes.

### Pointcut — What It Covers

```
springBeanPointcut():
    within(@Repository *) OR within(@Service *) OR within(@RestController *)

applicationPackagePointcut():
    within(com.revature.user..*)              ← all classes in the app package

BOTH must match → aspect fires

Combined: all Repositories/Services/Controllers inside com.revature.user.*
```

### Two Advices

```
@Around("applicationPackagePointcut() && springBeanPointcut()")
logAround(ProceedingJoinPoint joinPoint):
    IF DEBUG enabled:
        BEFORE: log.debug("Enter: UserService.authenticate() with args=[john, ***]")
    joinPoint.proceed()   ← run the actual method
    IF DEBUG enabled:
        AFTER: log.debug("Exit: UserService.authenticate() with result=AuthResponse{...}")
    IF IllegalArgumentException:
        log.error("Illegal argument: [args] in UserService.authenticate()")
        re-throw

@AfterThrowing("applicationPackagePointcut() && springBeanPointcut()")
logAfterThrowing(JoinPoint joinPoint, Throwable e):
    log.error("Exception in UserService.authenticate() with cause=NullPointerException")
    (does NOT swallow — exception continues propagating)
```

---

## Component 11 — CorsConfig

### File: `config/CorsConfig.java`

### What it does
Registers CORS (Cross-Origin Resource Sharing) rules with Spring MVC so the
Angular frontend at `localhost:4200` can make API calls to the backend at `localhost:8080`.

```java
registry.addMapping("/api/**")
    .allowedOrigins("http://localhost:4200", "http://localhost", "http://frontend")
    .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS", "PATCH")
    .allowedHeaders("*")               // any request header allowed
    .exposedHeaders("Authorization")   // frontend can READ this response header
    .allowCredentials(true)            // cookies/auth headers are included
    .maxAge(3600);                     // browser caches preflight for 1 hour
```

The origins are loaded from config:
```yaml
cors:
  allowed:
    origins: http://localhost:4200,http://localhost,http://frontend
```

---

## Component 12 — OpenApiConfig (Swagger)

### File: `config/OpenApiConfig.java`

### What it does
Configures Swagger UI (`/swagger-ui.html`) to show a **Bearer token input** at the
top of the page, so developers can log in once and test all protected endpoints
without manually setting headers in Postman.

```java
SecurityScheme bearerAuth = new SecurityScheme()
    .type(HTTP)          // HTTP auth, not apiKey
    .scheme("bearer")    // "Authorization: Bearer ..."
    .bearerFormat("JWT") // tells Swagger it's a JWT token

// Applies this scheme globally to all endpoints → all require JWT by default
openAPI.addSecurityItem(new SecurityRequirement().addList("bearerAuth"))
```

### How Developers Use It

```
1. Open http://localhost:8081/swagger-ui.html
2. Call POST /api/auth/login with credentials
3. Copy the accessToken from response
4. Click "Authorize" button at top of Swagger UI
5. Paste token → all subsequent API calls send "Authorization: Bearer <token>"
6. No more manual header setting in every request
```

---

## How All 11 Components Interact on a Login

```
FRONTEND → POST /api/auth/login

[Gateway: RateLimitGlobalFilter]     ← checks IP request count
[Gateway: JwtAuthFilter]             ← /api/auth/** → SKIPPED (public route)
[user-service: CorsConfig]           ← validates Origin header
[user-service: LoggingAspect]        ← "Enter: AuthController.login()"
[user-service: JwtAuthenticationFilter] ← public route → skipped

[AuthController → AuthenticationService]
    ├─ [ClientIpUtil.getClientIpAddress()]      ← real IP from X-Forwarded-For
    ├─ [DeviceFingerprintUtil.generateFingerprint()] ← SHA-256 of headers
    ├─ [GeoLocationService.getLocationFromIp()]  ← "Local Network" or "City, Country"
    ├─ [LoginAttemptService.getRecentFailedAttempts()] ← lockout/captcha check
    ├─ [CaptchaService.isCaptchaRequired()]      ← 3+ failures → captcha required
    ├─ [BCrypt password validation]
    ├─ [AdaptiveAuthService.calculateRiskScore()]← 0-100 risk check
    │    └─ [SecurityClient.createAlert()]       ← "New IP login" alert (async)
    ├─ [JwtTokenProvider.generateAccessToken()]  ← Builds JWT with JwtConfig
    ├─ [JwtTokenProvider.generateRefreshToken()]
    ├─ [SessionService.createSession()]          ← Writes to user_sessions
    └─ [LoginAttemptService.recordLoginAttempt()] ← Writes to login_attempts

[LoggingAspect]                      ← "Exit: AuthController.login()"
[RESPONSE 200 OK → { accessToken, refreshToken }]
```

---

## Dependencies Summary

| Component | Key Dependency |
| :--- | :--- |
| `JwtConfig` | `spring-boot-configuration-processor`, Config Server |
| `JwtTokenProvider` | `jjwt` (io.jsonwebtoken) |
| `JwtAuthenticationFilter` | `spring-security`, `jjwt` |
| `ClientIpUtil` | `jakarta.servlet` |
| `DeviceFingerprintUtil` | `java.security.MessageDigest` (SHA-256) |
| `GeoLocationService` | `ClientIpUtil` (internal) |
| `AdaptiveAuthService` | `LoginAttemptRepository`, `SecurityClient` (Feign) |
| `CaptchaService` | `LoginAttemptService`, `CaptchaConfig` |
| `LoginAttemptService` | `spring-data-jpa`, `LoginAttemptRepository` |
| `LoggingAspect` | `spring-aop`, `aspectjweaver`, `slf4j` |
| `CorsConfig` | `spring-webmvc` (WebMvcConfigurer) |
| `OpenApiConfig` | `springdoc-openapi-starter-webmvc-ui` |

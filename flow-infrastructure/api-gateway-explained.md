# API Gateway — Complete Deep-Dive Flow
> Part of the infrastructure deep-dive series.

This document provides a highly detailed, comprehensive breakdown of everything the **API Gateway** does. It acts as the single entry point (the "front door") for all traffic coming from the Angular frontend into the microservice ecosystem.

---

## 1. Architectural Role & File Structure

The API Gateway handles three massive responsibilities so that the downstream microservices (like `user-service` or `vault-service`) don't have to:
1. **Dynamic Routing & Load Balancing:** Sending requests to the correct service instance.
2. **Global Rate Limiting:** Stopping DDoS attacks and brute-force attempts.
3. **Authentication (JWT Verification):** Verifying tokens and establishing identity before letting traffic into the private network.

### Key Files Involved
1. `ApiGatewayApplication.java` — The bootstrap class (`@EnableDiscoveryClient`).
2. `RateLimitGlobalFilter.java` — A *GlobalFilter* (applied to **all** requests) that limits traffic per IP address.
3. `JwtAuthFilter.java` — A *GatewayFilterFactory* (applied only to **protected** routes) that verifies JWTs and mutates the request.
4. `api-gateway.yml` (in Config Server) — The central routing rulebook defining paths and filters.

---

## 2. Global Rate Limiting (`RateLimitGlobalFilter.java`)

Before a request is even allowed to be routed or its JWT checked, it passes through the **Global Rate Limiter**. 

### How It Works internally:
The gateway uses Google Guava `LoadingCache` to keep track of IP addresses in memory.
- **Cache Expiry:** Entries expire exactly 1 minute after their last write.
- **IP Resolution:** It intelligently checks the `X-Forwarded-For` header first (in case it is sitting behind an Nginx proxy or AWS ALB), falling back to the raw `RemoteAddress`.

### The Limits:
- **Authentication Endpoints (`/api/auth/**`):** Maximum **10 requests per minute** per IP. 
  *(Strictly protects against brute-forcing passwords or spamming OTP emails).*
- **All Other Endpoints:** Maximum **60 requests per minute** per IP.
  *(Protects backend database connections and third-party LLM API limits).*

### The Rejection Flow:
If an IP address exceeds the limit, the filter immediately halts the chain and returns:
- **HTTP 429 Too Many Requests**
- Body: `{"error":"Too Many Requests","message":"Rate limit exceeded. Please try again later."}`

### The Success Flow:
If the request is allowed through, the filter injects a helpful HTTP header into the response going back to the frontend: `X-RateLimit-Remaining: 59`.

---

## 3. The Routing Mechanism (`api-gateway.yml`)

The API Gateway pulls its routing configurations from the Config Server on startup.
It uses **Spring Cloud Gateway Predicates and Filters**.

Here is exactly how the backend is carved up:

| Route ID | Path Predicates (`/api/...`) | Destination | Needs JWT? |
| :--- | :--- | :--- | :--- |
| `user-service-public` | `/auth/**` | `lb://user-service` | ❌ No |
| `user-service-protected` | `/users/**`, `/2fa/**`, `/sessions/**`, `/settings/**` | `lb://user-service` | ✅ Yes |
| `vault-service` | `/vault/**`, `/categories/**`, `/folders/**`, `/shares/**`, `/timeline/**` | `lb://vault-service` | ✅ Yes |
| `generator-service` | `/generator/**` | `lb://generator-service` | ❌ No |
| `security-service` | `/security/**`, `/dashboard/**` | `lb://security-service` | ✅ Yes |
| `notification-service` | `/notifications/**` | `lb://notification-service` | ✅ Yes |
| `ai-service` | `/ai/**` | `lb://ai-service` | ✅ Yes |

> **Notice:** The Password Generator service does *not* require a JWT. It is completely public.

> **What does `lb://` mean?** It stands for LoadBalancer. When the gateway matches `/api/vault`, it queries Eureka for all IPs of `vault-service`, picks one using Round-Robin, and routes the traffic there.

---

## 4. JWT Authentication & Request Mutation (`JwtAuthFilter.java`)

If a route has `filters: - JwtAuthFilter` configured in the YAML, the request is intercepted. The downstream services (Vault, User, Security) **completely trust** the Gateway to handle validation.

### The Complete Authentication Flow

```
FRONTEND (Angular)
    |
    | GET /api/vault
    | Authorization: Bearer eyJhbG...
    v
[API GATEWAY]
    |
    |-- 1. Rate Limiter (Global Filter, Order: -2)
    |      Checks IP cache -> Allowed.
    |
    |-- 2. Route Matching
    |      Matched Predicate: /api/vault/**
    |      Requires Filter: JwtAuthFilter
    |
    |-- 3. JwtAuthFilter executes:
           |
           |-- A. Extracts "Authorization" header. (Rejects 401 if missing).
           |-- B. Strips "Bearer " prefix.
           |-- C. Validates cryptographic signature using `jwt.secret`. (Rejects 401 if tampered/expired).
           |-- D. Extracts Claims:
                  username = claims.getSubject()  -> "reddy123"
                  isDuress = claims.get("duress") -> false
           |
           |-- E. REQUEST MUTATION (The Magic Step)
                  The gateway takes the incoming HTTP request and attaches custom headers:
                  X-User-Name: reddy123
                  X-Is-Duress: false
    |
    |-- 4. Load Balancing
    |      Queries Eureka for `lb://vault-service`
    |      Forwards the MUTATED request to Vault-Service (e.g., 10.0.0.8:8082).
    v
[VAULT-SERVICE]
```

### Why Request Mutation?
Because the gateway validates the JWT, the `vault-service` doesn't need to import heavy JWT deciphering libraries, and it doesn't need to know the `jwt.secret`. 
The `vault-service` simply reads the `X-User-Name` header that the Gateway securely injected.

**Security Guarantee:** The outside world cannot bypass the Gateway and send HTTP requests directly to `vault-service:8082` with fake `X-User-Name` headers because the microservices are isolated inside a private Docker bridge network. The API Gateway is the only container with an exposed port (`8080`).

---

## 5. End-to-End Example Flow: "Viewing the Vault"

Here is every single step that happens when a user clicks "My Vault" in the frontend:

1. **Frontend** sends `GET https://myapp.com/api/vault` with `Authorization: Bearer <token>`.
2. **Gateway** receives the request on port `8080`.
3. **Gateway (RateLimitGlobalFilter)** resolves the IP. Checks the Guava cache for general traffic. Counter increments to 1/60. Allowed.
4. **Gateway (Router)** looks at the URL `/api/vault`. Matches it to the `vault-service` route config pulled from `config-server`. Sees that `JwtAuthFilter` must be applied.
5. **Gateway (JwtAuthFilter)** parses the token using the secret key. Confirms it's valid. Extracts `sub="reddy"`.  
6. **Gateway (Mutation)** builds a copy of the incoming request, but adds `X-User-Name: reddy` to the HTTP headers.
7. **Gateway (LoadBalancer)** looks up `vault-service` in its local Eureka cache. Discovers the internal IP is `172.18.0.5:8082`.
8. **Gateway (Netty Client)** forwards the mutated HTTP request to `http://172.18.0.5:8082/api/vault`.
9. **Vault-Service (GatewayAuthFilter)** receives the request. Reads `X-User-Name` header. Tells Spring Security "Reddy is authenticated."
10. **Vault-Service (Controller)** queries MySQL for Reddy's encrypted entries.
11. **Vault-Service (EncryptionUtil)** queries `user-service` via Feign for Reddy's salt to derive the AES decryption key, decrypts the passwords.
12. **Vault-Service** returns the decrypted JSON to the Gateway.
13. **Gateway** injects the `X-RateLimit-Remaining: 59` header into the response.
14. **Gateway** forwards the final JSON back to the Angular Frontend.

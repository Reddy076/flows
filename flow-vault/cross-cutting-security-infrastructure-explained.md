# Cross-Cutting & Security Infrastructure — Complete Flow Document
> Part of the `vault-service` deep-dive series.

This document explains the infrastructure features that apply globally to the
entire `vault-service` — components that are not tied to a single specific endpoint,
but affect every request that comes through the service.

---

## 1. JWT Trust Filter & Security Configuration
(`GatewayAuthFilter.java`, `SecurityConfig.java`)

### The Architecture Problem
In a microservices architecture, you do not want every single service to re-validate
the JWT cryptographic signature and re-query the session database. That causes massive
redundancy and database load.

### The Solution: Gateway Trust
1. The **API Gateway** handles the heavy lifting: validates the JWT signature, parsing
   it, and checking expiry.
2. The Gateway injects headers into the request before forwarding it:
   `X-User-Name: john` and `X-Is-Duress: false`
3. The **vault-service** trusts the internal API Gateway network. It skips full JWT
   validation and simply reads those headers to establish the active user.

### Flow Diagram

```
[API GATEWAY]
    |-- Receives POST /api/vault
    |-- Validates JWT "eyJ..." → Valid!
    |-- Extracts "sub" claim → "john"
    |-- Extracts "duress" claim → "false"
    |-- Forwards request with HTTP Headers added:
    |       X-User-Name: john
    |       X-Is-Duress: false
    v
[VAULT-SERVICE: GatewayAuthFilter.java]
    |
    |-- request.getHeader("X-User-Name")  → "john"
    |
    |-- Create Authentication object:
    |       UsernamePasswordAuthenticationToken auth =
    |           new UsernamePasswordAuthenticationToken(username, null, new ArrayList<>());
    |
    |-- Set Spring Security Context:
    |       SecurityContextHolder.getContext().setAuthentication(auth);
    |
    |-- Continues filter chain
    v
[VAULT-SERVICE: SecurityConfig.java]
    |
    |-- Requires authentication for all non-internal endpoints:
    |   .authorizeHttpRequests(auth -> auth
    |       .requestMatchers("/internal/**").permitAll()
    |       .anyRequest().authenticated()
    |   )
    v
[VAULT-SERVICE: Controllers]
    | Can safely call: SecurityContextHolder.getContext().getAuthentication().getName()
    | or read @RequestHeader("X-User-Name") directly.
```

---

## 2. Cryptography Engine & Key Derivation
(`EncryptionUtil.java`)

The `EncryptionUtil` class handles all symmetric encryption for user passwords
stored in the vault, as well as the secure sharing keys.

### Key Derivation Function (KDF)
Because you cannot safely use a human-readable password directly as a 256-bit AES key,
the system uses **PBKDF2 (Password-Based Key Derivation Function 2)** to stretch it.

```
VaultService calls: encryptionUtil.deriveKey(masterPasswordHash, salt)

Inside EncryptionUtil.java:
    factory = SecretKeyFactory.getInstance("PBKDF2WithHmacSHA256")
    iterations = 100000
    keyLength = 256

    spec = PBEKeySpec(masterPasswordHash, salt, 100000, 256)
    keyBytes = factory.generateSecret(spec).getEncoded()
    return new SecretKeySpec(keyBytes, "AES")
```

> **Why `masterPasswordHash` and not the raw master password?**
> The `vault-service` never sees the raw master password during normal operations.
> It fetches the BCrypt hash from `user-service`. By generating the key from the
> hash + salt, the vault-service can decrypt entries without needing the user to type
> their master password on every single request.

### Encryption / Decryption Algorithm
The engine uses **AES/GCM/NoPadding** (Advanced Encryption Standard in Galois/Counter Mode).
GCM is an authenticated encryption mode — it provides both confidentiality and data
integrity (prevents tampering).

**Encryption Flow:**
```
Input: "MyNetflixPass123!", SecretKey
    |
    |-- 1. Generate random IV (Initialization Vector)
    |      byte[12] iv; SecureRandom.nextBytes(iv);
    |
    |-- 2. Init Cipher
    |      Cipher.getInstance("AES/GCM/NoPadding")
    |      cipher.init(ENCRYPT_MODE, key, GCMParameterSpec(tagLength=128, iv))
    |
    |-- 3. Encrypt
    |      byte[] cipherText = cipher.doFinal("MyNetflixPass123!")
    |
    |-- 4. Combine IV + CipherText
    |      byte[] combined = new byte[12 + cipherText.length]
    |      (copy IV into first 12 bytes, then cipherText)
    |
    |-- 5. Base64 Encode
    |      return Base64.getEncoder().encodeToString(combined)
```

**Decryption Flow:**
```
Input: "ab7F2j... (Base64)", SecretKey
    |
    |-- 1. Base64 Decode
    |      byte[] decoded = Base64.getDecoder().decode(encryptedData)
    |
    |-- 2. Split IV and CipherText
    |      byte[12] iv = first 12 bytes
    |      byte[] cipherText = everything after 12 bytes
    |
    |-- 3. Init Cipher
    |      Cipher.getInstance("AES/GCM/NoPadding")
    |      cipher.init(DECRYPT_MODE, key, GCMParameterSpec(128, iv))
    |
    |-- 4. Decrypt
    |      byte[] plainText = cipher.doFinal(cipherText)
    |
    |-- 5. Return String
    |      return new String(plainText, UTF_8)
```

---

## 3. Distributed Service Integration
(`UserClient.java`, `SecurityClient.java`, `FeignConfig.java`)

`vault-service` depends heavily on other services. All synchronous microservice
communication is handled by **Spring Cloud OpenFeign**.

### Flow for a Feign Call (e.g., getting User Details)

```
[VaultService.java]
    |
    |-- userClient.getUserDetailsByUsername("john")
    v
[UserClient.java (Feign Interface)]
    | @GetMapping("/api/users/internal/by-username/{username}")
    v
[Spring Cloud OpenFeign]
    |-- Looks up "USER-SERVICE" in Eureka Service Registry
    |   (Gets host:port e.g., 10.0.0.5:8081)
    |-- Serializes parameters
    |-- Makes HTTP GET request over internal Docker network
    v
[USER-SERVICE (:8081)]
    |-- Processes request via its UserController
    |-- Returns JSON: { "id": 1, "masterPasswordHash": "$2a...", "salt": "..." }
    v
[Spring Cloud OpenFeign (vault-service)]
    |-- Deserializes JSON into UserVaultDetails record
    v
[VaultService.java]
    | Resumes execution with UserVaultDetails object
```

---

## 4. API Documentation
(`OpenApiConfig.java`)

Swagger/OpenAPI UI requires explicit configuration to know that API paths expect
a JWT Bearer token, even though the Gateway actually does the validation.

```java
@Configuration
public class OpenApiConfig {
    // Configures the Swagger UI to show the "Authorize" padlock button
    // and injects the "Bearer eyJ..." header into Swagger requests.
    @Bean
    public OpenAPI vaultOpenAPI() { ... }
}
```

---

## 5. Cross-Origin Resource Sharing
(`CorsConfig.java`)

Even though the API Gateway handles initial CORS requests, `vault-service` also defines
`CorsConfig` as defence-in-depth and for direct `localhost` testing without the gateway.

```
BROWSER (OPTIONS preflight /api/vault)
    |
    v
[CorsConfig.java (WebMvcConfigurer)]
    |-- Checks allowed origins (http://localhost:4200)
    |-- Checks allowed methods (GET, POST, PUT, DELETE, OPTIONS, PATCH)
    |-- Returns 200 OK with Access-Control-* headers
```

---

## 6. Global Exception Handling
(`GlobalExceptionHandler.java`)

Standardizes all error responses from API endpoints into a consistent format,
preventing raw stack traces from leaking to the frontend.

```
[Any Controller/Service]
    |
    |-- throws ResourceNotFoundException("Entry not found")
    v
[GlobalExceptionHandler.java]
    |
    |-- @ExceptionHandler(ResourceNotFoundException.class)
    |-- Logs the error internally
    |-- Builds standardized JSON:
    |   {
    |     "timestamp": "2026-03-24T01:30:00",
    |     "status": 404,
    |     "error": "Not Found",
    |     "message": "Entry not found",
    |     "path": "/api/vault/5"
    |   }
    v
[RESPONSE 404 NOT FOUND]
```

## Security Summary

| Component | Responsibility | Protection Against |
| :--- | :--- | :--- |
| `GatewayAuthFilter` | Establishing user identity | Unauthenticated requests |
| `EncryptionUtil (GCM)` | Authenticated encryption | Password theft from DB; Ciphertext tampering |
| `EncryptionUtil (PBKDF2)` | Key stretching | Brute forcing keys |
| `GlobalExceptionHandler` | Standardized errors | Information / Stack trace leakage |
| `FeignClients` | Internal HTTP routing | Bypassing API Gateway |

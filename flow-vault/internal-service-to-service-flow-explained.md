# Internal / Service-to-Service — Complete Flow Document
> Part of the `vault-service` deep-dive series.

This document explains the internal endpoints in `vault-service` that are called by
**other microservices** via Feign clients — these endpoints are **never exposed to the
browser** and do not go through the API Gateway.

---

## How Internal Calls Work — No API Gateway, No JWT

```
NORMAL (browser) FLOW:
    Frontend → API Gateway (:8080) → vault-service (:8082)
                  [rate limit + JWT check + X-User-Name injection]

INTERNAL (service-to-service) FLOW:
    security-service (:8083)  ──────────────────────► vault-service (:8082)
    user-service (:8081)      ──Feign (HTTP GET)────► /internal/vault/**
                                  No gateway hop.
                                  No JWT required.
                                  No X-User-Name header.
                                  userId passed directly in the URL path.
```

### Why No Authentication?

```
SecurityConfig (vault-service):
    .authorizeHttpRequests(auth -> auth
        .requestMatchers("/internal/**").permitAll()   ← no token required
        .anyRequest().authenticated()
    )

Protection is provided by Docker network isolation instead:
    - /internal/** endpoints are NOT routed by the API Gateway
    - Only containers on the same Docker network can reach vault-service:8082 directly
    - A browser on the public internet cannot reach port 8082 — only port 8080 (gateway)
```

---

## Files Involved

| File | Package | Role |
| :--- | :--- | :--- |
| `VaultInternalController.java` | `controller` | The only file — handles both internal endpoints |
| `VaultEntryRepository.java` | `repository` | Fetches entries + counts for both endpoints |
| `VaultTrashRepository.java` | `repository` | Counts trash entries for stats endpoint |
| `UserClient.java` | `client` | Feign — fetches `masterPasswordHash + salt` from user-service |
| `EncryptionUtil.java` | `util` | Derives AES-256 key; decrypts each vault entry password |

### Caller-Side (in other services)

| File | Service | Role |
| :--- | :--- | :--- |
| `VaultAnalysisClient.java` | `security-service` | Feign client that calls `/internal/vault/{userId}/decrypted-entries` |
| `VaultClient.java` | `user-service` | Feign client that calls `/internal/vault/stats/{userId}` |

### DTOs (static inner classes in `VaultInternalController.java`)

| Class | Fields |
| :--- | :--- |
| `DecryptedVaultEntryDTO` | `entryId, title, websiteUrl, categoryId, categoryName, isFavorite, decryptedPassword, createdAt, updatedAt` |
| `VaultStatsResponse` | `vaultCount, favoriteCount, trashCount` |

---

## Feature 1 — Get Decrypted Vault Entries (for AI/Security Analysis)

### What it does
Called by `security-service` when it needs to **analyse password strength** for all of a
user's vault entries. Since passwords are stored AES-256 encrypted, this endpoint
derives the per-user AES key and returns the entries with plaintext passwords.

This is the **only place in the entire system** where decrypted passwords leave the
vault-service — and only to another internal microservice, never to the browser.

### Who calls it
```
security-service → VaultAnalysisClient (Feign)
    → GET http://vault-service:8082/internal/vault/{userId}/decrypted-entries
```

### ASCII Flow Diagram

```
SECURITY-SERVICE
    |
    | Needs to analyse password strength for userId=1
    | VaultAnalysisClient.getDecryptedEntries(userId=1)
    |
    v
[Feign HTTP GET]
    http://vault-service:8082/internal/vault/1/decrypted-entries
    (No Authorization header — internal call, no JWT)
    |
    v
[VaultInternalController.getDecryptedEntries(userId=1)]
    |
    |-- STEP 1: FETCH ALL ACTIVE ENTRIES FOR USER
    |   vaultEntryRepository.findByUserIdAndIsDeletedFalse(1)
    |       → MYSQL SELECT * FROM vault_entries
    |                WHERE user_id = 1 AND is_deleted = false
    |       → Returns e.g. 15 VaultEntry objects (passwords still encrypted)
    |
    |-- STEP 2: FETCH USER'S ENCRYPTION KEY MATERIAL
    |   userClient.getUserDetailsById(1)
    |       → Feign: GET http://user-service:8081/api/users/internal/by-id/1
    |       → Returns UserVaultDetails {
    |             id: 1,
    |             masterPasswordHash: "$2a$10$...",   ← BCrypt hash
    |             salt: "a1b2c3d4e5f6..."             ← hex salt
    |         }
    |   If userClient call fails (user-service down) → key = null
    |       → All decryptedPassword fields will be empty string ""
    |       → Method still returns entries (graceful degradation)
    |
    |-- STEP 3: DERIVE AES-256 DECRYPTION KEY
    |   encryptionUtil.deriveKey(masterPasswordHash, salt)
    |       → PBKDF2WithHmacSHA256(password=masterPasswordHash, salt=salt)
    |       → 256-bit AES SecretKey
    |       (Same derivation the user-service does when changing master password)
    |
    |-- STEP 4: DECRYPT EACH ENTRY'S PASSWORD
    |   For each VaultEntry in the list:
    |       encryptionUtil.decrypt(entry.getPassword(), decryptionKey)
    |           → AES/GCM/NoPadding decryption
    |           → cipher text (stored in DB) → plain text password
    |       If decryption fails for one entry → log.warn() + decryptedPassword = ""
    |           (one bad entry does not fail the whole request)
    |
    |-- STEP 5: BUILD DTOs (no filtering, all active entries)
    |   DecryptedVaultEntryDTO {
    |       entryId, title, websiteUrl, categoryId, categoryName,
    |       isFavorite, decryptedPassword,   ← ← ← THE PLAINTEXT PASSWORD
    |       createdAt, updatedAt
    |   }
    |
    v
[RESPONSE 200 OK]
    | Body: List<DecryptedVaultEntryDTO> [
    |   {
    |     entryId: 5,
    |     title: "Netflix",
    |     websiteUrl: "netflix.com",
    |     categoryId: 2,
    |     categoryName: "Entertainment",
    |     isFavorite: true,
    |     decryptedPassword: "MyNetflixPass123!",   ← actual plaintext
    |     createdAt: "2024-01-10T09:00",
    |     updatedAt: "2026-03-01T14:22"
    |   },
    |   {
    |     entryId: 3,
    |     title: "Gmail",
    |     decryptedPassword: "",   ← empty if decryption failed
    |     ...
    |   },
    |   ... (all 15 active entries)
    | ]
    |
    v
SECURITY-SERVICE
    | Receives plaintext passwords
    | Scores each password (length, complexity, breach check via HIBP)
    | Stores strength scores back in security_service DB
    | NEVER stores or logs the plaintext passwords themselves
```

### The Decryption Math

```
STORING (at vault entry creation):
    user types: "MyNetflixPass123!"
    vault-service derives AES key:
        PBKDF2WithHmacSHA256(masterPasswordHash + salt) → 256-bit AES key
    vault-service encrypts:
        AES/GCM encrypt("MyNetflixPass123!", key)
        → base64(IV + ciphertext) stored in vault_entries.password column

RETRIEVING (this endpoint):
    vault-service fetches: masterPasswordHash + salt from user-service
    vault-service re-derives the SAME AES key:
        PBKDF2WithHmacSHA256(masterPasswordHash + salt) → same 256-bit key
    vault-service decrypts:
        AES/GCM decrypt(stored_ciphertext, key) → "MyNetflixPass123!"

KEY INSIGHT:
    The AES key is NEVER stored anywhere.
    It is derived on-demand from masterPasswordHash + salt.
    If the user changes their master password, the key changes too —
    which is why changeMasterPassword re-encrypts every vault entry.
```

---

## Feature 2 — Get Vault Stats (for User Dashboard)

### What it does
Called by `user-service` when building the dashboard response. Returns 3 aggregate
counts so the user-service can include vault tile counts without querying the vault DB
itself.

### Who calls it
```
user-service → VaultClient (Feign)
    → GET http://vault-service:8082/internal/vault/stats/{userId}
```

### ASCII Flow Diagram

```
USER-SERVICE
    |
    | UserController.getDashboard()
    | needs vault counts for the dashboard tile display
    |
    v
[VaultClient.getDashboardStats(userId=1)]
    | Feign HTTP GET
    | http://vault-service:8082/internal/vault/stats/1
    |
    v
[VaultInternalController.getDashboardStats(userId=1)]
    |
    |-- COUNT 1: ACTIVE VAULT ENTRIES
    |   vaultEntryRepository.countByUserIdAndIsDeletedFalse(1)
    |       → MYSQL SELECT COUNT(*) FROM vault_entries
    |                WHERE user_id = 1 AND is_deleted = false
    |       → vaultCount = 15
    |
    |-- COUNT 2: FAVORITES
    |   vaultEntryRepository.countByUserIdAndIsFavoriteTrue(1)
    |       → MYSQL SELECT COUNT(*) FROM vault_entries
    |                WHERE user_id = 1 AND is_favorite = true
    |       → favoriteCount = 3
    |
    |-- COUNT 3: TRASH
    |   vaultTrashRepository.countByUserIdAndIsDeletedTrue(1)
    |       → MYSQL SELECT COUNT(*) FROM vault_entries
    |                WHERE user_id = 1 AND is_deleted = true
    |       → trashCount = 2
    |
    v
[RESPONSE 200 OK]
    | Body: VaultStatsResponse {
    |   vaultCount: 15,
    |   favoriteCount: 3,
    |   trashCount: 2
    | }
    |
    v
USER-SERVICE
    | VaultClient returns VaultStatsResponse
    | UserController builds DashboardResponse:
    |   {
    |     totalVaultEntries: 15,
    |     totalFavorites: 3,
    |     trashCount: 2
    |   }
    | → Returns to frontend

NOTE: If vault-service is unreachable (Feign error):
    VaultClient catches the exception → returns { 0, 0, 0 }
    Dashboard still loads — graceful degradation
```

---

## Security Design Notes

| Decision | Why |
| :--- | :--- |
| `permitAll()` for `/internal/**` in SecurityConfig | These endpoints receive no JWT — they trust Docker network isolation instead |
| AES key derived on-demand, never stored | Even if the vault DB is compromised, passwords cannot be decrypted without the masterPasswordHash + salt from user-service |
| `log.warn()` on decryption failure, not exception | One corrupted entry does not break the entire analysis batch |
| `decryptedPassword = ""` on failure | security-service can still score the other entries; it skips blank passwords |
| No `X-User-Name` header used | userId is a path param — caller knows who to request for; no gateway hop |
| `VaultStatsResponse` is a static inner class | Keeps the DTO co-located with the only controller that uses it |

---

## Database Tables Used

| Table | Operation | Endpoint |
| :--- | :--- | :--- |
| `vault_entries` | `SELECT * WHERE user_id = ? AND is_deleted = false` | decrypted-entries |
| `vault_entries` | `COUNT(*) WHERE user_id = ? AND is_deleted = false` | stats |
| `vault_entries` | `COUNT(*) WHERE user_id = ? AND is_favorite = true` | stats |
| `vault_entries` | `COUNT(*) WHERE user_id = ? AND is_deleted = true` | stats (trash) |

---

## Dependencies Summary

| Dependency | Used For |
| :--- | :--- |
| `spring-data-jpa` | `VaultEntryRepository`, `VaultTrashRepository` — all DB reads |
| `spring-cloud-openfeign` | `UserClient` — fetch masterPasswordHash + salt from user-service |
| `javax.crypto` | AES/GCM decryption via `EncryptionUtil` |
| `Lombok` | `@RequiredArgsConstructor` |
| `Slf4j` | Warn logging when decryption or user-fetch fails |

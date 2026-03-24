# Secure Sharing — Complete Flow Document
> Part of the `vault-service` deep-dive series.

This document explains the full end-to-end flow of every Secure Sharing feature,
including the cryptographic design that makes sharing safe without revealing the vault key.

---

## How Every Request Travels — FRONTEND → API Gateway → vault-service

```
FRONTEND (Angular :4200)
    │
    │  HTTP Request
    │  Authorization: Bearer eyJ...   ← JWT (authenticated routes only)
    │
    ▼
API GATEWAY (:8080)
    ├── 1. RateLimitGlobalFilter
    │       429 Too Many Requests if over limit
    │
    ├── 2. JwtAuthFilter
    │       PROTECTED routes (/api/shares, /api/shares/{id} DELETE, GET /api/shares/received):
    │           validates JWT signature + expiry
    │           injects: X-User-Name: john, X-Is-Duress: false
    │           if invalid → 401 Unauthorized
    │       PUBLIC route (GET /api/shares/{token}):
    │           skipps JWT validation entirely
    │           passes through with no X-User-Name header
    │
    └── 3. Routes to vault-service (:8082)
    │
    ▼
VAULT-SERVICE (:8082)
    ├── CorsConfig         → validates Origin
    ├── GatewayAuthFilter  (reads X-User-Name ? sets SecurityContext)
    │       reads X-User-Name ? sets SecurityContext
    ├── SecurityConfig
    │       /api/shares/{token} GET → permitAll()  (public)
    │       /api/shares/**      → authenticated()  (owner-only)
    └── SecureShareController → SecureShareService
```

---

## The Core Problem — How Do You Share a Password Safely?

```
NAIVE (WRONG) approach:
    Share the vault URL with the password in it?
        → The vault AES key is derived from the master password hash + salt
        → If the recipient gets the key, they can decrypt ALL entries in the vault
        → Master password compromise affects EVERYTHING

SECURE approach (what this system does):
    1. Decrypt the one password that should be shared
    2. Re-encrypt it with a BRAND NEW one-time AES-256-GCM key
    3. Store ONLY the ciphertext + IV in the database (never the key)
    4. Put the new key in the URL fragment (#keyBase64)
       → URL fragment is NEVER sent to the server (browser-only)
    5. Recipient opens the URL → browser reads key from fragment → decrypts locally
    6. Server never sees the one-time key → even a DB breach can't reveal the password
```

### The Two-Key Architecture

```
VAULT KEY (per user):
    Derived from: masterPasswordHash + salt  →  PBKDF2 → 256-bit AES key
    Used for:     ALL entries in the vault
    Never stored: computed fresh on every request
    Exposure risk: compromises the ENTIRE vault

SHARE KEY (per share):
    Generated:    fresh AES-256 KeyGenerator for EACH share
    Used for:     THIS ONE PASSWORD only
    Stored:       NEVER — returned to caller in response, embedded in URL fragment
    Exposure risk: compromises only this ONE shared password
```

---

## Files Involved

| File | Package | Role |
| :--- | :--- | :--- |
| `SecureShareController.java` | `controller` | HTTP entry point for all `/api/shares/**` endpoints |
| `SecureShareService.java` | `service/sharing` | All sharing business logic |
| `ShareEncryptionService.java` | `service/sharing` | One-time AES-256-GCM encrypt/decrypt for share keys |
| `ShareTokenGenerator.java` | `service/sharing` | Generates collision-free UUID-based share tokens |
| `ShareExpirationService.java` | `service/sharing` | Validity checks + bulk expiry + cleanup logic |
| `SecureShareCleanupScheduler.java` | `scheduler` | Scheduled background job — calls `ShareExpirationService` |
| `SecureShareRepository.java` | `repository` | All DB queries for `secure_shares` table |
| `UserClient.java` | `client` | Feign — resolve username → user details + vault key materials |
| `SecurityClient.java` | `client` | Audit log for SHARE_CREATED, SHARE_ACCESSED, SHARE_REVOKED |
| `NotificationClient.java` | `client` | In-app notification sent to recipient (if registered user) |
| `EncryptionUtil.java` | `util` | PBKDF2 vault key derivation + vault password decryption |
| `EncryptionService.java` | `service/vault` | Wrapper for decrypt — reads vault entry's encrypted password |

---

## SharePermission Enum — The 3 Permission Modes

| Mode | Max Views | Time Limit | Behaviour |
| :--- | :--- | :--- | :--- |
| `VIEW_ONCE` | 1 | Yes (default 24h) | Link dies after the first access |
| `VIEW_MULTIPLE` | N (e.g. 5) | Yes | Link dies after N views or expiry, whichever is first |
| `TEMPORARY_ACCESS` | ∞ (unlimited) | Yes | Link works unlimited times until `expiresAt` passes |

---

## SecureShare — Database Table Structure

| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | BIGINT PK | Auto-generated |
| `vault_entry_id` | BIGINT FK | The entry being shared (`vault_entries.id`) |
| `owner_id` | BIGINT | The user who created the share |
| `recipient_email` | VARCHAR | Optional — who the share was sent to |
| `share_token` | VARCHAR UNIQUE | 32-char UUID-based token used in the URL |
| `encrypted_password` | TEXT | AES-GCM ciphertext of the plaintext password |
| `encryption_iv` | VARCHAR | Base64-encoded 12-byte IV for decryption |
| `expires_at` | DATETIME | Hard deadline — link stops working after this |
| `max_views` | INT | Maximum allowed view count |
| `view_count` | INT | How many times the link has been accessed |
| `permission` | ENUM | `VIEW_ONCE`, `VIEW_MULTIPLE`, `TEMPORARY_ACCESS` |
| `is_revoked` | BOOLEAN | Owner manually killed the link |
| `created_at` | DATETIME | Auto timestamp |

**What is NOT stored:**
- The one-time AES share key → never persisted, only in the URL fragment
- Plaintext password → only the ciphertext is stored

---

## ShareTokenGenerator — URL Token Generation

```java
// Generates a collision-safe, URL-clean, cryptographically random token

UUID.randomUUID()                    → "550e8400-e29b-41d4-a716-446655440000"
.toString().replace("-", "")         → "550e8400e29b41d4a716446655440000" (32 chars)

// Collision check just to be safe:
for (int attempt = 0; attempt < 5; attempt++) {
    String token = UUID.randomUUID().toString().replace("-", "");
    if (!shareRepository.existsByShareToken(token)) {
        return token;  // unique → use this
    }
}
// After 5 collisions (essentially impossible): throw IllegalStateException
```

---

## Feature 1 — Create Secure Share Link

### Request

```
POST /api/shares
Authorization: Bearer <token>
Body: {
    "vaultEntryId": 5,
    "recipientEmail": "friend@example.com",    ← optional
    "permission": "VIEW_ONCE",                  ← or VIEW_MULTIPLE / TEMPORARY_ACCESS
    "maxViews": 1,
    "expiryHours": 24                           ← link dies after 24 hours
}
```

### ASCII Flow Diagram

```
FRONTEND
    |
    | POST /api/shares
    | Authorization: Bearer <token>
    |
    v
[API GATEWAY (:8080)]
    |-- RateLimitGlobalFilter: rate check
    |-- JwtAuthFilter: /api/shares → PROTECTED → validates JWT
    |       Injects: X-User-Name: john
    v
VAULT-SERVICE (:8082)
    |-- CorsConfig: valid Origin check
    |-- GatewayAuthFilter: reads X-User-Name header ? sets SecurityContext
    |-- SecurityConfig: /api/shares → authenticated() ✓
    v
[SecureShareController.createShare(request)]
    | getCurrentUsername() → "john" (from SecurityContext)
    v
[SecureShareService.createShare("john", request)]
    |
    ├── STEP 1: GET OWNER + ENTRY
    │   userClient.getUserDetailsByUsername("john")
    │       → Feign GET http://user-service/api/users/internal/by-username/john
    │       → Returns: { id:1, masterPasswordHash, salt, ... }
    │
    │   vaultEntryRepository.findByIdAndUserId(5, 1)
    │       → MYSQL: SELECT * FROM vault_entries WHERE id=5 AND user_id=1
    │   If not found → 404 ResourceNotFoundException
    │
    ├── GUARD: Highly Sensitive entries BLOCKED from sharing
    │   if (entry.isHighlySensitive) → 400 "Entry cannot be shared"
    │   (prevents sharing entries that require extra re-auth)
    │
    ├── STEP 2: DECRYPT THE VAULT PASSWORD
    │   encryptionUtil.deriveKey(masterPasswordHash, salt)
    │       → PBKDF2WithHmacSHA256, 100k iterations → 256-bit vaultKey
    │   encryptionService.decrypt(entry.password, vaultKey)
    │       → AES-256-GCM decrypt using vault key
    │       → plainPassword = "MyP@ssw0rd!"
    │
    ├── STEP 3: RE-ENCRYPT WITH ONE-TIME SHARE KEY
    │   shareEncryptionService.encrypt("MyP@ssw0rd!")
    │       → KeyGenerator.getInstance("AES").init(256) → new random SecretKey
    │       → SecureRandom.nextBytes(iv)  → 12-byte random IV
    │       → AES-256-GCM encrypt("MyP@ssw0rd!", shareKey, iv)
    │       → Returns ShareEncryptionResult {
    │             encryptedData: "Base64(ciphertext + GCM tag)"
    │             iv:            "Base64(12-byte IV)"
    │             keyBase64:     "Base64(raw 32-byte AES key)"   ← key to return to caller
    │         }
    │
    ├── STEP 4: DETERMINE PERMISSION & MAX VIEWS
    │   permission = SharePermission.valueOf(request.permission) → VIEW_ONCE
    │   maxViews:
    │       TEMPORARY_ACCESS → Integer.MAX_VALUE (unlimited)
    │       else             → Math.max(1, request.maxViews)  → 1
    │
    ├── STEP 5: GENERATE SHARE TOKEN
    │   tokenGenerator.generate()
    │       → UUID.randomUUID().toString().replace("-","") → "550e8400e29b41d4..."
    │       → checks DB for collision → unique → returns token
    │
    ├── STEP 6: SAVE SHARE RECORD TO DB
    │   SecureShare {
    │       vaultEntry:        entry5
    │       ownerId:           1
    │       recipientEmail:    "friend@example.com"
    │       shareToken:        "550e8400e29b41d4..."
    │       encryptedPassword: "Base64(ciphertext)"   ← one-time ciphertext
    │       encryptionIv:      "Base64(iv)"
    │       expiresAt:         now + 24 hours
    │       maxViews:          1
    │       permission:        VIEW_ONCE
    │       viewCount:         0
    │       isRevoked:         false
    │   }
    │   shareRepository.save(share) → MYSQL INSERT INTO secure_shares
    │
    ├── STEP 7: NOTIFY RECIPIENT (if email provided)
    │   ┌── Try: userClient.getUserDetailsByUsername("friend@example.com")
    │   │       → If recipient is a REGISTERED user:
    │   │           notificationClient.createNotification(
    │   │               "friend", "ACCOUNT_ACTIVITY",
    │   │               "Secure Password Shared With You",
    │   │               "john has shared a secure password link with you..."
    │   │           )
    │   └── Catch: if recipient not a registered user → silently log warning, continue
    │   (Share still works even if notification fails)
    │
    ├── STEP 8: AUDIT LOG
    │   securityClient.logAction("john", "SHARE_CREATED", "Shared entry 'Netflix' token=550e...")
    │
    v
[RESPONSE 201 Created]
    | Body: ShareLinkResponse {
    |     shareId:       1
    |     shareToken:    "550e8400e29b41d4..."
    |     shareUrl:      "/api/shares/550e8400e29b41d4..."
    |     encryptionKey: "Base64(32-byte AES key)"   ← ONLY returned on create
    |     vaultEntryTitle: "Netflix"
    |     recipientEmail:  "friend@example.com"
    |     permission:      "VIEW_ONCE"
    |     maxViews:        1
    |     viewCount:       0
    |     expiresAt:       "2026-03-25T00:39:00"
    |     revoked:         false
    | }

FRONTEND BUILDS THE SHARE URL:
    shareUrl + "#" + encryptionKey
    → "/api/shares/550e8400...#Base64AESKeyHere"
    → The # fragment is NEVER sent to the server
```

---

## Feature 2 — Access Shared Password (Public — No Auth Required)

### The Cryptographic Handshake

```
SENDER sends:   /api/shares/550e8400...#SGVsbG9Xb3JsZA==
                                         ↑ Base64 AES key
                                         └── lives in URL fragment (#)
                                             NEVER sent to server

RECIPIENT's browser:
    1. Loads page: GET /api/shares/550e8400...  ← server only sees token
    2. Server returns: { encryptedPassword, encryptionIv }
    3. Browser reads key from window.location.hash → "SGVsbG9Xb3JsZA=="
    4. Browser decrypts: AES-256-GCM(encryptedPassword, key, iv) → "MyP@ssw0rd!"
    5. Shows plaintext to recipient
```

### ASCII Flow Diagram

```
RECIPIENT (no login required)
    |
    | GET /api/shares/550e8400e29b41d4...
    | No Authorization header
    |
    v
[API GATEWAY (:8080)]
    |-- RateLimitGlobalFilter: rate check
    |-- JwtAuthFilter: /api/shares/{token} GET → PUBLIC route → JWT check SKIPPED
    |       No X-User-Name header injected
    v
VAULT-SERVICE (:8082)
    |-- CorsConfig: valid Origin check
    |-- GatewayAuthFilter: SKIPPED (no X-User-Name header on public route)
    |-- SecurityConfig: /api/shares/{token} → permitAll() ✓
    v
[SecureShareController.getSharedPassword("550e8400...")]
    v
[SecureShareService.getSharedPassword("550e8400...")]
    |
    ├── STEP 1: LOOK UP SHARE BY TOKEN
    │   shareRepository.findByShareToken("550e8400...")
    │       → MYSQL: SELECT * FROM secure_shares WHERE share_token = ?
    │   If not found → 404 "Share not found or has expired"
    │
    ├── STEP 2: VALIDITY GATE (3-way check via share.isValid())
    │   ┌── CHECK A: share.isRevoked() == true?
    │   │   → throw "Share is no longer valid: revoked"
    │   ├── CHECK B: viewCount >= maxViews?
    │   │   → throw "Share is no longer valid: view limit reached"
    │   └── CHECK C: LocalDateTime.now().isAfter(expiresAt)?
    │       → throw "Share is no longer valid: expired"
    │
    │   ALL PASS → proceed
    │
    ├── STEP 3: INCREMENT VIEW COUNT
    │   share.setViewCount(viewCount + 1)   → 0 → 1
    │   shareRepository.save(share)
    │       → MYSQL UPDATE secure_shares SET view_count = 1 WHERE id = ?
    │
    │   (If VIEW_ONCE: this view pushed count to 1 = maxViews
    │    → next call to isValid() will return false → link is now dead)
    │
    ├── STEP 4: DECRYPT VAULT USERNAME (for context)
    │   ┌── Try:
    │   │   userClient.getUserDetailsById(share.ownerId)
    │   │       → Feign GET user-service for master hash + salt
    │   │   vaultKey = deriveKey(masterHash, salt)
    │   │   plainUsername = decrypt(entry.username, vaultKey)
    │   └── Catch: if fails → plainUsername = null (graceful fallback)
    │   (username is for context only — recipient knows WHICH account this is for)
    │
    ├── STEP 5: AUDIT LOG
    │   securityClient.logAction(owner.username, "SHARE_ACCESSED",
    │       "token=550e... viewCount=1")
    │
    v
[RESPONSE 200 OK]
    | Body: SharedPasswordResponse {
    |     title:             "Netflix"
    |     username:          "john@gmail.com"    ← decrypted from vault
    |     encryptedPassword: "Base64(ciphertext)" ← from secure_shares table
    |     encryptionIv:      "Base64(iv)"         ← from secure_shares table
    |     websiteUrl:        "netflix.com"
    |     permission:        "VIEW_ONCE"
    |     viewsRemaining:    0                    ← link is now dead
    |     expiresAt:         "2026-03-25T00:39:00"
    | }

RECIPIENT'S BROWSER:
    reads # fragment → key = "SGVsbG9XbXJsZA=="
    AES-GCM.decrypt(encryptedPassword, iv=decode(encryptionIv), key=decode(key))
    → "MyP@ssw0rd!"  ← shown to recipient
    Server NEVER saw the key or the plaintext
```

---

## Feature 3 — List My Active Shares

```
FRONTEND
    |
    | GET /api/shares
    | Authorization: Bearer <token>
    |
    v
[API GATEWAY] → validates JWT → X-User-Name: john → vault-service
    |
    v
[GatewayAuthFilter] → reads X-User-Name ? sets SecurityContext
    |
    v
[SecureShareController.getActiveShares()] → getCurrentUsername() → "john"
    v
[SecureShareService.getActiveShares("john")]
    |
    |-- userClient.getUserDetailsByUsername("john") → user.id = 1
    |
    |-- shareRepository.findActiveSharesByOwnerId(1, LocalDateTime.now())
    |       → MYSQL: SELECT * FROM secure_shares
    |                WHERE owner_id = 1
    |                  AND is_revoked = false
    |                  AND expires_at > NOW()
    |       Note: view_count >= max_views NOT filtered here
    |           → exhausted shares still show in the list (so owner knows)
    |
    |-- toShareLinkResponse(share, null)
    |       encryptionKey = null  ← key is NOT returned on list
    |       (key was only returned once at create time, never again)
    |
    v
[RESPONSE 200 OK]
    | Body: [
    |   {
    |     shareId: 1,
    |     shareToken: "550e8400...",
    |     shareUrl: "/api/shares/550e8400...",
    |     encryptionKey: null,         ← key never revealed again
    |     vaultEntryTitle: "Netflix",
    |     recipientEmail: "friend@example.com",
    |     permission: "VIEW_ONCE",
    |     maxViews: 1,
    |     viewCount: 1,               ← already viewed once
    |     expiresAt: "2026-03-25T00:39:00",
    |     revoked: false
    |   }
    | ]
```

---

## Feature 4 — Revoke a Share

```
FRONTEND
    | (owner clicks "Revoke" on an active share)
    |
    | DELETE /api/shares/{id}
    | Authorization: Bearer <token>
    | e.g. DELETE /api/shares/1
    |
    v
[API GATEWAY] → validates JWT → X-User-Name: john → vault-service
    |
    v
[GatewayAuthFilter] → reads X-User-Name ? sets SecurityContext
    |
    v
[SecureShareController.revokeShare(id)] → getCurrentUsername() → "john"
    v
[SecureShareService.revokeShare("john", 1)]
    |
    |-- STEP 1: GET USER
    |   userClient.getUserDetailsByUsername("john") → user.id = 1
    |
    |-- STEP 2: LOAD SHARE
    |   shareRepository.findById(1)
    |       → MYSQL: SELECT * FROM secure_shares WHERE id = 1
    |   If not found → 404
    |
    |-- STEP 3: OWNERSHIP CHECK
    |   userClient.getUserDetailsById(share.ownerId).id == user.id?
    |   If not → 400 "Share does not belong to this user"
    |   (prevents revoking another user's shares even if you know the share ID)
    |
    |-- STEP 4: REVOKE
    |   share.setRevoked(true)
    |   shareRepository.save(share)
    |       → MYSQL UPDATE secure_shares SET is_revoked = true WHERE id = 1
    |
    |-- STEP 5: AUDIT
    |   securityClient.logAction("john", "SHARE_REVOKED", "id=1 entry='Netflix'")
    |
    v
[RESPONSE 200 OK]
    | Body: ShareLinkResponse { ..., revoked: true }

IMMEDIATE EFFECT:
    Next time anyone accesses /api/shares/550e8400...
    → isValid() check: share.isRevoked() == true → throw "Share is no longer valid: revoked"
    → 400 error immediately
```

---

## Feature 5 — List Received Shares

```
FRONTEND
    | (recipient logs in and checks shares sent to them)
    |
    | GET /api/shares/received
    | Authorization: Bearer <token>
    |
    v
[API GATEWAY] → validates JWT → X-User-Name: friend → vault-service
    |
    v
[GatewayAuthFilter] → reads X-User-Name ? sets SecurityContext
    |
    v
[SecureShareController.getReceivedShares()] → getCurrentUsername() → "friend"
    v
[SecureShareService.getReceivedShares("friend")]
    |
    |-- userClient.getUserDetailsByUsername("friend") → user details
    |
    |-- shareRepository.findActiveSharesForRecipient("friend", LocalDateTime.now())
    |       → MYSQL: SELECT * FROM secure_shares
    |                WHERE recipient_email = 'friend'
    |                  AND is_revoked = false
    |                  AND expires_at > NOW()
    |
    |-- toShareLinkResponse(share, null) for each
    |       (encryptionKey is null — recipient must use the original URL with #key)
    |
    v
[RESPONSE 200 OK]
    | Body: list of active shares received by this user
```

---

## Feature 6 — Background Expiry Cleanup (Scheduler)

### ShareExpirationService — All Validity Logic

```
isValid(share):
    if share == null          → false
    if share.isRevoked()      → false
    if now > expiresAt        → false (time expired)
    if isViewLimitReached()   → false (used up)
    → true (still accessible)

isViewLimitReached(share):
    if permission == TEMPORARY_ACCESS → false (unlimited)
    else: viewCount >= maxViews → true/false

getInvalidReason(share):
    revoked         → "revoked"
    expired         → "expired"
    view limit      → "view limit reached"
    valid           → null

secondsUntilExpiry(share):
    Math.max(0, ChronoUnit.SECONDS.between(now, expiresAt))

deleteExpiredShares(cutoff):
    findByExpiresAtBeforeAndIsRevokedFalse(cutoff)
        → MYSQL: SELECT * FROM secure_shares
                 WHERE expires_at < ? AND is_revoked = false
    shareRepository.deleteAll(expired)
    → Physical hard delete from DB
```

### Auto-Cleanup Flow

```
[SecureShareCleanupScheduler] — fires on cron schedule
    |
    v
[ShareExpirationService.deleteExpiredSharesOlderThan(graceHours)]
    |
    |-- cutoff = LocalDateTime.now().minusHours(graceHours)
    |       e.g. graceHours=1 → cutoff = "2026-03-24T23:39:00"
    |
    |-- deleteExpiredShares(cutoff)
    |       SELECT WHERE expires_at < cutoff AND is_revoked = false
    |       deleteAll(expired)
    |       → MYSQL DELETE FROM secure_shares WHERE id IN (...)
    |
    |-- logs: "Deleted 3 expired secure shares (older than 2026-03-24T23:39:00)"
    |
    v
[DONE — expired shares physically removed from DB]
```

---

## Full Share Lifecycle — Combined Diagram

```
[OWNER creates share]
    │
    │  POST /api/shares → creates secure_shares row
    │  Returns: { shareUrl, encryptionKey }
    │
    ▼
[OWNER sends URL to recipient]
    │  E.g. via email: "/api/shares/550e8400...#AESKeyBase64"
    │                                              └── key in fragment
    ▼
[RECIPIENT opens URL in browser]
    │
    │  GET /api/shares/550e8400...   ← server never sees the # fragment
    │
    ▼
[Server validates: not revoked, not expired, views < maxViews]
    │  viewCount++  → saved to DB
    │
    ▼
[Server returns: encryptedPassword + iv]
    │
    ▼
[Browser: decrypt(encryptedPassword, iv, key_from_url_fragment)]
    │
    └─► plaintext password shown to recipient (never sent to server)

        ─── For VIEW_ONCE: link is now dead (viewCount = maxViews) ───

        ─── For TEMPORARY_ACCESS: link works again until expiresAt ───

[OWNER revokes share anytime]
    │  DELETE /api/shares/1 → is_revoked = true
    └─► Any future access returns: "Share is no longer valid: revoked"

[Cleanup Scheduler: runs nightly]
    └─► Deletes all shares where expires_at < now (from DB entirely)
```

---

## Security Design — Why This Approach is Safe

| Threat | Mitigation |
| :--- | :--- |
| **DB breach** — attacker reads `secure_shares` table | Ciphertext only, IV only. Key is NEVER stored. Nothing decryptable. |
| **Server logs** — sniff the encryption key | Key lives only in URL fragment (`#key`). Fragment is never sent to server, never in logs. |
| **Link forwarded** — recipient shares the URL | `VIEW_ONCE` makes link dead after first use. `TEMPORARY_ACCESS` uses expiry time. |
| **Someone guesses the token** | 32-char UUID (122 bits of entropy). Probability: ~4×10⁻³⁷. Collision-checked on generation. |
| **Sharing highly sensitive entries** | Blocked at `createShare()` — `isHighlySensitive` check throws 400. |
| **Cross-user share revocation** | Owner ID double-checked: `getUserDetailsById(share.ownerId).id == currentUserId` |
| **Stale shares after password change** | `updateEntry()` auto-revokes all active shares for the changed entry immediately |

---

## Dependencies Summary

| Dependency | Used For |
| :--- | :--- |
| `javax.crypto.KeyGenerator` | Generates fresh 256-bit AES key per share |
| `javax.crypto` (AES-GCM) | One-time encrypt/decrypt in `ShareEncryptionService` |
| `java.security.SecureRandom` | Cryptographically random IV per encryption |
| `java.util.UUID` | Share token generation (122-bit entropy) |
| `spring-data-jpa` | All `secure_shares` DB operations |
| `spring-cloud-openfeign` | `UserClient`, `SecurityClient`, `NotificationClient` |
| `spring-scheduling` | `SecureShareCleanupScheduler` background cleanup |
| `slf4j` | Every share creation/access/revoke logged at INFO |

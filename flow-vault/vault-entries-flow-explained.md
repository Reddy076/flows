# Vault Entries — Complete Flow Document
> Part of the `vault-service` deep-dive series.

This document explains the full end-to-end flow of every Vault Entries feature,
from the frontend HTTP request all the way through to the database — including every
file involved, every dependency used, and ASCII diagrams for each flow.

---

## How Every Request Travels — FRONTEND → API Gateway → vault-service

No request from the frontend ever reaches `vault-service` directly.
Every call passes through the **API Gateway** first, which validates the JWT and injects
headers that vault-service relies on.

```
FRONTEND (Angular :4200)
    │
    │  HTTP Request  (e.g. GET /api/vault)
    │  Authorization: Bearer eyJ...     ← JWT token in every request
    │
    ▼
API GATEWAY (:8080)
    ├── 1. RateLimitGlobalFilter
    │       Checks: is this IP over 100 req/min?
    │       If yes → 429 Too Many Requests (frontend never reaches vault-service)
    │
    ├── 2. JwtAuthFilter
    │       /api/vault/** → PROTECTED route (JWT required)
    │       Validates: JWT signature + expiry using shared secret
    │       Extracts: username from JWT claims
    │       Checks:   isDuressLogin claim in JWT
    │       Injects headers into forwarded request:
    │           X-User-Name: john           ← username from JWT
    │           X-Is-Duress: false          ← or true for duress logins
    │       If JWT invalid/missing → 401 Unauthorized (request stopped here)
    │
    └── 3. Routes to vault-service (:8082)
    │
    ▼
VAULT-SERVICE (:8082)  ← request arrives with injected headers
    ├── CorsConfig
    │       Validates Origin header (e.g. http://localhost:4200)
    │       If not allowed → 403 Forbidden
    │
    ├── GatewayAuthFilter  (reads X-User-Name → sets SecurityContext)
    │       Reads X-User-Name header (injected by API Gateway)
    │       Sets SecurityContextHolder.getAuthentication().getName() = "john"
    │       No JWT re-validation, no DB session check — trusts the gateway
    │       (vault-service does NOT have JwtAuthenticationFilter)
    │
    ├── SecurityConfig
    │       /api/vault/**               → authenticated() — must pass GatewayAuthFilter
    │       /api/vault/*/sensitive-view → authenticated() + body-level re-auth
    │
    └── VaultController → VaultService → EncryptionService → DB
```

### What each header does in vault-service

| Header | Set by | Used by |
| :--- | :--- | :--- |
| `Authorization: Bearer ...` | Frontend | `GatewayAuthFilter` reads X-User-Name ? sets SecurityContext |
| `X-User-Name: john` | API Gateway `JwtAuthFilter` | `BackupController` reads directly; `VaultController` uses `SecurityContextHolder` |
| `X-Is-Duress: true/false` | API Gateway `JwtAuthFilter` | `VaultService.isDuressMode()` — returns fake data if `true` |

---

## The Core Concept — Encryption-First Storage

Every vault entry stores only **AES-256-GCM encrypted ciphertext** — never plaintext.
The encryption key is derived on-the-fly from the user's master password hash and salt,
fetched from `user-service` via a Feign client on every request.

```
plaintext password: "MyP@ssw0rd!"
                         ↓
AES-256-GCM encrypt(plaintext, derivedKey)
                         ↓
stored in DB:  "aB3kXp9+mN2qR7sT4wVy8uL1oQ6j...==" (Base64 ciphertext)
```

The key is **never stored** — it is recomputed every time from:
```
PBKDF2WithHmacSHA256(masterPasswordHash, salt, 100000 iterations, 256 bits)
→ 32-byte AES-256 SecretKey
```

---

## Files Involved

| File | Package | Role |
| :--- | :--- | :--- |
| `VaultController.java` | `controller` | HTTP entry point for all `/api/vault/**` endpoints |
| `VaultService.java` | `service/vault` | All vault entry business logic |
| `VaultSnapshotService.java` | `service/vault` | Password change history — create, get, restore |
| `EncryptionUtil.java` | `util` | AES-256-GCM encrypt/decrypt + PBKDF2 key derivation |
| `EncryptionService.java` | `service/vault` | Wrapper around `EncryptionUtil` used by `VaultService` |
| `DuressService.java` | `service/vault` | Generates fake decoy vault data for duress logins |
| `PasswordStrengthCalculator.java` | `util` | Scores password strength 0-100 on every list/get |
| `TOTPUtil.java` | `util` | Verifies TOTP OTP as second factor for sensitive entries |
| `UserClient.java` | `client` | Feign client — fetches masterPasswordHash + salt from user-service |
| `SecurityClient.java` | `client` | Logs actions + triggers async password analysis |
| `NotificationClient.java` | `client` | Sends in-app notifications for create/update/delete |
| `VaultEntryRepository.java` | `repository` | All DB queries for `vault_entries` table |
| `VaultSnapshotRepository.java` | `repository` | All DB queries for `vault_snapshots` table |
| `CategoryRepository.java` | `repository` | Looks up category entity by ID |
| `FolderRepository.java` | `repository` | Looks up folder entity by ID |
| `SecureShareRepository.java` | `repository` | Auto-revokes shares when password is changed |

---

## Encryption Deep Dive — AES-256-GCM

### Key Derivation (`EncryptionUtil.deriveKey`)

```
INPUT:
    masterPasswordHash = "$2a$10$abc..."   ← BCrypt hash from user-service
    salt               = "base64saltXYZ"   ← random per-user salt from user-service

PROCESS (PBKDF2WithHmacSHA256):
    PBEKeySpec(
        password  = masterPasswordHash.toCharArray()
        salt      = salt.getBytes(UTF_8)
        iterations= 100,000               ← slows brute force by 100k factor
        keyLength = 256 bits              ← AES-256
    )
    SecretKeyFactory.generateSecret(spec) → 32-byte raw key
    new SecretKeySpec(keyBytes, "AES")    → SecretKey object

OUTPUT: a 256-bit AES SecretKey (exists only in JVM memory, never written to DB)
```

### Encryption (`EncryptionUtil.encrypt`)

```
INPUT: plaintext = "MyP@ssw0rd!", key = SecretKey

PROCESS:
    1. Generate 12-byte random IV: SecureRandom.nextBytes(iv)
       (new IV per encryption → same password encrypted twice = different ciphertext)

    2. Cipher.getInstance("AES/GCM/NoPadding")
       cipher.init(ENCRYPT_MODE, key, GCMParameterSpec(128-bit tag, iv))

    3. cipherText = cipher.doFinal("MyP@ssw0rd!".getBytes(UTF_8))
       (cipherText includes 16-byte GCM authentication tag at the end)

    4. Prepend IV to ciphertext:
       encryptedData = [12 bytes IV] + [cipherText bytes]

    5. Base64.encode(encryptedData) → "aB3kXp9+mN2qR7..."

OUTPUT: Base64 string stored in DB vault_entries.password column
```

### Decryption (`EncryptionUtil.decrypt`)

```
INPUT: encryptedData = "aB3kXp9+mN2qR7...", key = SecretKey

PROCESS:
    1. Base64.decode(encryptedData) → raw bytes
    2. Extract IV: first 12 bytes
    3. Extract cipherText: remaining bytes
    4. cipher.init(DECRYPT_MODE, key, GCMParameterSpec(128, iv))
    5. plainText = cipher.doFinal(cipherText)
       (GCM authentication tag is verified here — if tampered → exception)

OUTPUT: "MyP@ssw0rd!"
```

### Why AES-GCM (not AES-CBC)?

```
AES-CBC:  Encrypts data. Cannot detect tampering.
AES-GCM:  Encrypts data + produces authentication tag.
          If the ciphertext is modified in DB, decryption throws AEADBadTagException.
          Detects both: unauthorized modification AND corruption.
```

---

## Duress Mode — Fake Vault

Before every write operation, `VaultService` checks for the `X-Is-Duress: true` header
(set by `JwtAuthFilter` at the gateway when it detects a duress JWT token):

```
isDuressMode():
    reads request header X-Is-Duress from RequestContextHolder
    returns true if "true"

checkDuressMode():  ← called by createEntry, updateEntry, deleteEntry, etc.
    if isDuressMode() → throw "Action not permitted in duress mode"

For READ operations:
    if isDuressMode() → return DuressService.generateDummyVault(user)
                         (fake random entries — never read from real DB)
```

---

## Feature 1 — Create Vault Entry

### ASCII Flow Diagram

```
FRONTEND
    |
    | POST /api/vault
    | Authorization: Bearer <token>
    | Body: { title, username, password, websiteUrl, notes, categoryId, folderId, isFavorite, isHighlySensitive }
    |
    v
[API GATEWAY] → X-Is-Duress: false, X-User-Name: john → [vault-service]
    |
    v
[VaultController.createEntry()]
    |-- getCurrentUsername() → "john" (from SecurityContext)
    v
[VaultService.createEntry("john", request)]
    |
    |-- GUARD: checkDuressMode()
    |       isDuressMode()? → X-Is-Duress header = false → proceed
    |
    |-- STEP 1: GET USER ENCRYPTION MATERIALS
    |   userClient.getUserDetailsByUsername("john")
    |       → Feign: GET http://user-service/api/users/internal/by-username/john
    |       → Returns: { id, username, masterPasswordHash, salt, is2faEnabled, readOnlyMode }
    |
    |-- GUARD: checkReadOnlyMode(user)
    |       user.isReadOnlyMode() == false → proceed
    |
    |-- STEP 2: RESOLVE OPTIONAL CATEGORY & FOLDER
    |   categoryRepository.findById(categoryId)  → MYSQL SELECT
    |   folderRepository.findById(folderId)       → MYSQL SELECT
    |
    |-- STEP 3: DERIVE AES KEY
    |   encryptionUtil.deriveKey(masterPasswordHash, salt)
    |       → PBKDF2WithHmacSHA256, 100k iterations → 256-bit SecretKey
    |       (computed fresh, never stored)
    |
    |-- STEP 4: ENCRYPT SENSITIVE FIELDS
    |   encryptionService.encrypt(password, key)  → AES-GCM ciphertext
    |   encryptionService.encrypt(username, key)  → AES-GCM ciphertext
    |   encryptionService.encrypt(notes, key)     → AES-GCM ciphertext (if present)
    |   title, websiteUrl → stored PLAINTEXT (not sensitive)
    |
    |-- STEP 5: SAVE TO DB
    |   VaultEntry.builder()...build()
    |   vaultEntryRepository.save(entry) → MYSQL INSERT INTO vault_entries
    |
    |-- STEP 6: FIRE EVENTS (non-blocking)
    |   securityClient.logAction("ENTRY_CREATED", "Created entry: Netflix")
    |   notificationClient.createNotification("ACCOUNT_ACTIVITY", "Vault Entry Created")
    |   securityClient.analyzePasswordEntry(mapToPayload(entry))
    |       → AI password strength analysis runs asynchronously in security-service
    |
    v
[RESPONSE 201 Created]
    | Body: { id, title, websiteUrl, categoryId, isFavorite, strengthScore, strengthLabel }
    | NOTE: password and username are NEVER returned in the response
```

---

## Feature 2 — Get All Entries

### ASCII Flow Diagram

```
FRONTEND
    |
    | GET /api/vault
    | Authorization: Bearer <token>
    |
    v
[API GATEWAY] → validates JWT → X-User-Name: john, X-Is-Duress: false → vault-service
    |
    v
[GatewayAuthFilter] → reads X-User-Name ? sets SecurityContext → sets SecurityContext
    |
    v
[VaultController.getAllEntries()]
    | getCurrentUsername() → "john" (from SecurityContext)
    v
[VaultService.getAllEntries("john")]
    |
    |-- IF isDuressMode():
    |       return DuressService.generateDummyVault(user) → fake entries list
    |
    |-- userClient.getUserDetailsByUsername("john") → masterPasswordHash + salt
    |
    |-- vaultEntryRepository.findByUserIdAndIsDeletedFalse(userId)
    |       → MYSQL: SELECT * FROM vault_entries WHERE user_id = ? AND is_deleted = false
    |
    |-- getSharedKeySafely(user)
    |       → deriveKey(masterPasswordHash, salt) (single key derivation for all entries)
    |       Returns null if derivation fails (safe fallback)
    |
    |-- entries.stream().map(e -> mapToResponseWithKey(e, key))
    |       For EACH entry:
    |           encryptionService.decrypt(e.password, key) → rawPassword (NOT returned!)
    |           passwordStrengthCalculator.calculateScore(rawPassword)  → 0-100
    |           passwordStrengthCalculator.getStrengthLabel(score) → "Weak"/"Fair"/"Strong"
    |           Build VaultEntryResponse:
    |               { id, title, websiteUrl, category, folder, isFavorite,
    |                 username:"******",   ← always masked in list response
    |                 strengthScore, strengthLabel }
    |
    v
[RESPONSE 200 OK]
    | Body: [ { id, title, websiteUrl, strengthScore: 42, strengthLabel: "Fair", ... }, ... ]
    | Passwords are NEVER in the response — only strength scores
```

---

## Feature 3 — Get Single Entry (with decryption)

```
FRONTEND
    |
    | GET /api/vault/{id}
    | Authorization: Bearer <token>
    |
    v
[API GATEWAY] → validates JWT → X-User-Name: john, X-Is-Duress: false → vault-service
    |
    v
[GatewayAuthFilter] → reads X-User-Name ? sets SecurityContext
    |
    v
[VaultController.getEntry(id)]
    | getCurrentUsername() → "john"
    v
[VaultService.getEntry("john", 5)]
    |
    |-- IF isDuressMode() → return dummy entry (no real DB access)
    |
    |-- vaultEntryRepository.findByIdAndUserId(5, userId)
    |       → MYSQL: SELECT * FROM vault_entries WHERE id = 5 AND user_id = ?
    |   If not found → ResourceNotFoundException
    |
    |-- IF entry.isHighlySensitive == true:
    |       decryptedPassword = "******"   ← blocked!
    |       decryptedUsername = "******"
    |       decryptedNotes    = "******"
    |       requiresSensitiveAuth = true   ← signals frontend to show re-auth dialog
    |
    |-- ELSE:
    |   deriveKey() → AES key
    |   decryptedPassword = encryptionService.decrypt(entry.password, key) → "MyP@ssw0rd!"
    |   decryptedUsername = encryptionService.decrypt(entry.username, key) → "john@gmail.com"
    |   decryptedNotes    = encryptionService.decrypt(entry.notes, key)
    |
    v
[RESPONSE 200 OK]
    | Body: { id, title, websiteUrl, username, password, notes, strengthScore, ... }
```

---

## Feature 4 — Update Entry

```
FRONTEND
    |
    | PUT /api/vault/{id}
    | Authorization: Bearer <token>
    | Body: { title, password: "NewP@ss!", ... }
    |
    v
[API GATEWAY] → validates JWT → X-User-Name: john, X-Is-Duress: false → vault-service
    |
    v
[GatewayAuthFilter] → reads X-User-Name ? sets SecurityContext
    |
    v
[VaultController.updateEntry(id, request)]
    | getCurrentUsername() → "john"
    v
[VaultService.updateEntry("john", 5, request)]
    |
    |-- GUARD: checkDuressMode() + checkReadOnlyMode()
    |
    |-- vaultEntryRepository.findByIdAndUserId(5, userId) → ownership check
    |
    |-- Apply non-null request fields (title, websiteUrl, category, folder, isFavorite...)
    |
    |-- IF request.password != null AND password != "******":
    |       STEP A: vaultSnapshotService.createSnapshot(entry)
    |           → saves current (old) encrypted password → vault_snapshots table
    |           → creates point-in-time password history record
    |
    |       STEP B: encrypt new password
    |           encryptionService.encrypt("NewP@ss!", key) → new ciphertext
    |           entry.setPassword(newCiphertext)
    |
    |       STEP C: auto-revoke active shares for this entry
    |           secureShareRepository.findActiveSharesByOwnerId(userId, now)
    |               .filter(share -> share.vaultEntry.id == entry.id)
    |               .forEach(share -> share.setRevoked(true) → MYSQL UPDATE)
    |           (shares hold OLD ciphertext → must be revoked immediately)
    |
    |-- vaultEntryRepository.save(entry) → MYSQL UPDATE vault_entries
    |
    |-- securityClient.logAction("ENTRY_UPDATED")
    |-- notificationClient.createNotification("ACCOUNT_ACTIVITY")
    |-- securityClient.analyzePasswordEntry() → async AI re-analysis
    |
    v
[RESPONSE 200 OK] → { id, title, strengthScore, strengthLabel, ... }
```

---

## Feature 5 — Delete Entry (Soft Delete → Trash)

```
FRONTEND
    |
    | DELETE /api/vault/{id}
    | Authorization: Bearer <token>
    |
    v
[API GATEWAY] → validates JWT → X-User-Name: john, X-Is-Duress: false → vault-service
    |
    v
[GatewayAuthFilter] → reads X-User-Name ? sets SecurityContext
    |
    v
[VaultController.deleteEntry(id)]
    | getCurrentUsername() → "john"
    v
[VaultService.deleteEntry("john", 5)]
    |
    |-- GUARD: checkDuressMode() + checkReadOnlyMode()
    |-- Ownership check: findByIdAndUserId(5, userId)
    |
    |-- entry.setIsDeleted(true)
    |   entry.setDeletedAt(LocalDateTime.now())
    |   vaultEntryRepository.save(entry) → MYSQL UPDATE (soft delete)
    |       (data is NOT erased — only flagged, remains in trash)
    |
    |-- securityClient.logAction("ENTRY_DELETED")
    |-- notificationClient.createNotification("Vault Entry Deleted")
    |
    v
[RESPONSE 200 OK] → { message: "Entry deleted successfully" }
```

---

## Feature 6 — Search & Filter Entries

```
FRONTEND
    |
    | GET /api/vault/search?keyword=netflix&categoryId=2&sortBy=updatedAt&sortDir=desc
    | Authorization: Bearer <token>
    |
    v
[API GATEWAY] → validates JWT → X-User-Name: john, X-Is-Duress: false → vault-service
    |
    v
[GatewayAuthFilter] → reads X-User-Name ? sets SecurityContext
    |
    v
[VaultController.searchEntries(params)]
    | getCurrentUsername() → "john"
    v
[VaultService.searchEntries("john", "netflix", 2, null, null, null, "updatedAt", "desc")]
    |
    |-- IF isDuressMode() → return empty list
    |
    |-- userClient.getUserDetailsByUsername() → user.id
    |
    |-- vaultEntryRepository.searchEntries(userId, "netflix", categoryId=2, null, null, null)
    |       → MYSQL: SELECT * FROM vault_entries
    |                WHERE user_id = ?
    |                  AND is_deleted = false
    |                  AND (title LIKE '%netflix%' OR website_url LIKE '%netflix%')
    |                  AND category_id = 2
    |
    |-- Sort in memory (Java Stream):
    |       sortBy = "updatedAt" → Comparator.comparing(VaultEntry::getUpdatedAt)
    |       sortDir = "desc"    → .reversed()
    |
    |-- mapToResponseWithKey() for each → strength score computed
    |
    v
[RESPONSE 200 OK] → sorted, filtered list with strength scores
```

---

## Feature 7 — View Password (Decrypted)

```
FRONTEND
    | (user explicitly clicks "Show password" icon)
    |
    | POST /api/vault/entries/{id}/view-password
    | Authorization: Bearer <token>
    |
    v
[API GATEWAY] → validates JWT → X-User-Name: john, X-Is-Duress: false → vault-service
    |
    v
[GatewayAuthFilter] → reads X-User-Name ? sets SecurityContext
    |
    v
[VaultController.viewPassword(id)]
    | getCurrentUsername() → "john"
    v
[VaultService.getPassword("john", 5)]
    |
    |-- IF isDuressMode() → return "***encrypted***" (fake)
    |
    |-- findByIdAndUserId(5, userId) → ownership check
    |
    |-- IF entry.isHighlySensitive:
    |       throw "Sensitive entry requires authentication"
    |       (must use /sensitive-view endpoint instead)
    |
    |-- securityClient.logAction("PASSWORD_VIEWED", "Viewed password for: Netflix")
    |       ↑ Every password reveal is logged for audit
    |
    |-- deriveKey(masterPasswordHash, salt) → AES key
    |-- encryptionService.decrypt(entry.password, key) → "MyP@ssw0rd!"
    |
    v
[RESPONSE 200 OK] → { password: "MyP@ssw0rd!" }
```

---

## Feature 8 — Access Sensitive Entry (Double Authentication)

```
FRONTEND
    | (user has 2FA + master password required to see this entry)
    |
    | POST /api/vault/{id}/sensitive-view
    | Authorization: Bearer <token>
    | Body: { masterPassword: "My$ecret!", otpToken: "482910" }
    |
    v
[API GATEWAY] → validates JWT → X-User-Name: john, X-Is-Duress: false → vault-service
    │  (note: X-Is-Duress checked in-method below — duress gets "Invalid master password")
    |
    v
[GatewayAuthFilter] → reads X-User-Name ? sets SecurityContext
    |
    v
[VaultController.accessSensitiveEntry(id, request)]
    | getCurrentUsername() → "john"
    v
[VaultService.accessSensitiveEntry("john", 5, request)]
    |
    |-- IF isDuressMode() → throw "Invalid master password" (lies to attacker)
    |
    |-- findByIdAndUserId(5, userId) → ownership check
    |
    |-- GATE 1: MASTER PASSWORD VERIFICATION
    |   BCryptPasswordEncoder.matches(request.masterPassword, user.masterPasswordHash)
    |   If fail → throw "Invalid master password"
    |   (BCrypt compare: 2FA config error prevented storing wrong hash)
    |
    |-- GATE 2: TOTP VERIFICATION (only if user.is2faEnabled == true)
    |   user.getTwoFactorSecret() from user-service Feign response
    |   totpUtil.verifyCode(secret, request.otpToken)
    |       → HMAC-SHA1 time-window check (same as 2FA login)
    |   If fail → throw "Invalid OTP"
    |
    |-- BOTH GATES PASSED:
    |   deriveKey() → AES key
    |   decrypt(entry.password) → plaintext
    |   decrypt(entry.username) → plaintext
    |   decrypt(entry.notes)    → plaintext
    |
    v
[RESPONSE 200 OK] → full plaintext { id, title, username, password, notes, ... }
```

### Why double authentication?

```
Sensitive entries are flagged by the user as requiring extra proof.
Even though the JWT token is already validated, the master password check
ensures that:
    1. Only the account owner can see it (not someone who stole the JWT)
    2. If 2FA is enabled, the physical device (authenticator app) is also required
```

---

## Feature 9 — Toggle Favorite

```
FRONTEND
    |
    | PUT /api/vault/{id}/favorite
    | Authorization: Bearer <token>
    |
    v
[API GATEWAY] → validates JWT → X-User-Name: john → vault-service
    |
    v
[GatewayAuthFilter] → reads X-User-Name ? sets SecurityContext
    |
    v
[VaultController.toggleFavorite(id)] → getCurrentUsername() → "john"
    v
[VaultService.toggleFavorite("john", 5)]
    |
    |-- GUARD: checkDuressMode() + checkReadOnlyMode()
    |-- findByIdAndUserId → ownership check
    |-- entry.setIsFavorite(!entry.getIsFavorite())   ← simple boolean flip
    |-- vaultEntryRepository.save() → MYSQL UPDATE
    |-- notificationClient.createNotification("Favorited" or "Unfavorited")
    v
[RESPONSE 200 OK]
```

---

## Feature 10 — Toggle Sensitive Flag

```
FRONTEND
    |
    | PUT /api/vault/entries/{id}/sensitive
    | Authorization: Bearer <token>
    |
    v
[API GATEWAY] → validates JWT → X-User-Name: john → vault-service
    |
    v
[GatewayAuthFilter] → reads X-User-Name ? sets SecurityContext
    |
    v
[VaultController.toggleSensitive(id)] → getCurrentUsername() → "john"
    v
[VaultService.toggleSensitive("john", 5)]
    |
    |-- GUARD: checkDuressMode() + checkReadOnlyMode()
    |-- entry.setIsHighlySensitive(!entry.getIsHighlySensitive())
    |-- vaultEntryRepository.save() → MYSQL UPDATE
    |-- securityClient.logAction("ENTRY_UPDATED", "Toggled sensitive flag")
    v
[RESPONSE 200 OK] → entry with isHighlySensitive: true/false
```

---

## Feature 11 — Bulk Delete

```
FRONTEND
    |
    | POST /api/vault/entries/bulk-delete
    | Authorization: Bearer <token>
    | Body: [3, 7, 12, 18]   ← list of entry IDs
    |
    v
[API GATEWAY] → validates JWT → X-User-Name: john, X-Is-Duress: false → vault-service
    |
    v
[GatewayAuthFilter] → reads X-User-Name ? sets SecurityContext
    |
    v
[VaultController.bulkDelete(ids)] → getCurrentUsername() → "john"
    v
[VaultService.bulkDelete("john", [3,7,12,18])]
    |
    |-- GUARD: checkDuressMode() + checkReadOnlyMode()
    |-- vaultEntryRepository.findAllById([3,7,12,18])
    |
    |-- OWNERSHIP CHECK (critical — prevents deleting other users' entries):
    |   entries.stream().filter(e -> !e.userId.equals(currentUserId)).findAny()
    |   If ANY entry belongs to another user → throw "Access denied"
    |   (all-or-nothing check before any deletes happen)
    |
    |-- entries.forEach:
    |       e.setIsDeleted(true)
    |       e.setDeletedAt(now)
    |       securityClient.logAction("ENTRY_DELETED")
    |       notificationClient.createNotification()
    |-- vaultEntryRepository.saveAll(entries) → MYSQL batch UPDATE
    |       (single batch query instead of N individual updates)
    v
[RESPONSE 200 OK] → { message: "Entries deleted successfully" }
```

---

## Feature 12 — Password History (Snapshots)

### How Snapshots Are Created

```
Every time the password field is changed in updateEntry():
    │
    ▼
VaultSnapshotService.createSnapshot(entry)
    │
    │-- VaultSnapshot.builder()
    │       .vaultEntry(entry)         ← FK to vault_entries
    │       .password(entry.password)  ← current ENCRYPTED password (before update)
    │       .changedAt(LocalDateTime.now())
    │   vaultSnapshotRepository.save() → MYSQL INSERT INTO vault_snapshots
    │
    ▼
Now entry.password is updated with the new ciphertext
Old password is PRESERVED in vault_snapshots
```

### Viewing History

```
FRONTEND
    |
    | GET /api/vault/entries/{id}/history
    | Authorization: Bearer <token>
    |
    v
[API GATEWAY] → validates JWT → X-User-Name: john → vault-service
    |
    v
[GatewayAuthFilter] → reads X-User-Name ? sets SecurityContext
    |
    v
[VaultController.getPasswordHistory(id)] → getCurrentUsername() → "john"
    v
[VaultSnapshotService.getHistory("john", 5)]
    |
    |-- userClient.getUserDetailsByUsername() → get user.id
    |-- vaultEntryRepository.findByIdAndUserId(5, userId)  → ownership check
    |-- vaultSnapshotRepository.findByVaultEntryIdOrderByChangedAtDesc(5)
    |       → MYSQL: SELECT * FROM vault_snapshots
    |                WHERE vault_entry_id = 5
    |                ORDER BY changed_at DESC
    |
    |-- mapToResponse() for each snapshot:
    |       { id, entryTitle, changedAt, password: "******" }
    |       (password is ALWAYS masked — only timestamps shown)
    |
    v
[RESPONSE 200 OK]
    | Body: [
    |   { id: 3, entryTitle: "Netflix", changedAt: "2026-03-20T10:00:00", password: "******" },
    |   { id: 2, entryTitle: "Netflix", changedAt: "2026-03-10T08:00:00", password: "******" },
    |   { id: 1, entryTitle: "Netflix", changedAt: "2026-02-01T12:00:00", password: "******" }
    | ]
```

---

## VaultEntry — Database Table Structure

| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | BIGINT PK | Auto-generated primary key |
| `user_id` | BIGINT | Owner's user ID (from user-service) |
| `category_id` | BIGINT FK | Optional, references `categories.id` |
| `folder_id` | BIGINT FK | Optional, references `folders.id` |
| `title` | VARCHAR | **Plaintext** — used for search and display |
| `username` | TEXT | **Encrypted** AES-GCM ciphertext |
| `password` | TEXT | **Encrypted** AES-GCM ciphertext |
| `website_url` | VARCHAR | **Plaintext** — not sensitive |
| `notes` | TEXT | **Encrypted** AES-GCM ciphertext (nullable) |
| `is_favorite` | BOOLEAN | Favorite flag |
| `is_highly_sensitive` | BOOLEAN | Requires master password + OTP to view |
| `is_deleted` | BOOLEAN | Soft delete flag (trash) |
| `deleted_at` | DATETIME | When moved to trash |
| `created_at` | DATETIME | Auto-set on INSERT |
| `updated_at` | DATETIME | Auto-set on UPDATE |

---

## Dependencies Summary

| Dependency | Used For |
| :--- | :--- |
| `javax.crypto` (JDK) | AES-GCM cipher, PBKDF2 key derivation |
| `java.security.SecureRandom` | Random IV generation per encryption |
| `spring-security-crypto` | BCrypt master password verification (sensitive access) |
| `spring-data-jpa` | All DB reads and writes |
| `spring-cloud-openfeign` | `UserClient`, `SecurityClient`, `NotificationClient` |
| `Lombok` | `@Builder`, `@RequiredArgsConstructor` |

---

## Response Types — What is Returned vs Masked

| Endpoint | password field | username field | notes field |
| :--- | :--- | :--- | :--- |
| GET `/api/vault` (list) | ❌ Not returned | `"******"` | ❌ Not returned |
| GET `/api/vault/{id}` (non-sensitive) | ✅ Decrypted plaintext | ✅ Decrypted | ✅ Decrypted |
| GET `/api/vault/{id}` (sensitive) | `"******"` | `"******"` | `"******"` |
| POST `/entries/{id}/view-password` | ✅ Decrypted plaintext | — | — |
| POST `/{id}/sensitive-view` (after auth) | ✅ Decrypted plaintext | ✅ Decrypted | ✅ Decrypted |
| Duress mode (any endpoint) | `"***encrypted***"` | — | — |

# Backup & Import/Export — Complete Flow Document
> Part of the `vault-service` deep-dive series.

This document explains the full end-to-end flow of every Backup, Import/Export, and Snapshot
feature — including how plaintext passwords flow out on export, get re-encrypted on import,
and how the Factory Pattern drives third-party password manager imports.

---

## How Every Request Travels — FRONTEND → API Gateway → vault-service

```
FRONTEND (Angular :4200)
    │
    │  HTTP Request (e.g. GET /api/backup/export)
    │  Authorization: Bearer eyJ...
    │
    ▼
API GATEWAY (:8080)
    ├── 1. RateLimitGlobalFilter → 429 if over limit
    │
    ├── 2. JwtAuthFilter
    │       /api/backup/** → PROTECTED route → JWT required
    │       Validates JWT signature + expiry
    │       Injects into forwarded request:
    │           X-User-Name: john    ← BackupController reads this directly
    │           X-Is-Duress: false
    │       If invalid → 401 Unauthorized
    │
    └── 3. Routes to vault-service (:8082)
    │
    ▼
VAULT-SERVICE (:8082)  ← arrives with X-User-Name, X-Is-Duress headers
    ├── CorsConfig              → validates Origin
    ├── GatewayAuthFilter → reads X-User-Name → sets SecurityContext
    ├── SecurityConfig          → /api/backup/** → authenticated()
    └── BackupController reads X-User-Name header directly
               (unlike VaultController which uses SecurityContextHolder)
```

> **Note on BackupController:** Unlike `VaultController` which calls `SecurityContextHolder.getContext().getAuthentication().getName()`,
> `BackupController` reads the username directly from the `X-User-Name` request header
> that the API Gateway injects. Both approaches are equivalent in security terms since
> the gateway only sets this header after validating the JWT.

---

## The Core Challenge — Moving Encrypted Data In and Out

```
VAULT DATABASE:
    entry.password = "aB3kXp9+mN2qR7sT4wVy8uL1..."  ← AES-256-GCM ciphertext

EXPORT (user wants a file):
    Must DECRYPT ciphertext → plaintext → write to file
    (AES key derived from masterPasswordHash + salt via PBKDF2)

IMPORT (user uploads a file):
    Must ENCRYPT each plaintext entry → ciphertext → save to DB
    (Same key derivation — so the same user's vault key is used)

ENCRYPTED EXPORT (optional security):
    Decrypt vault → plaintext → re-encrypt file with user-provided password
    (Different from vault key — uses a password the user types at export time)
```

---

## Files Involved

| File | Package | Role |
| :--- | :--- | :--- |
| `BackupController.java` | `controller` | HTTP entry point for all `/api/backup/**` endpoints |
| `ExportService.java` | `service/backup` | Decrypt vault + serialize to JSON or CSV (+ optional file encryption) |
| `ImportService.java` | `service/backup` | Parse JSON/CSV, encrypt each entry, bulk-save to vault |
| `ThirdPartyImportService.java` | `service/backup` | Dispatches to the correct importer via `ImporterFactory` |
| `ImporterFactory.java` | `service/backup/importers` | Strategy registry — maps source name → `Importer` implementation |
| `Importer.java` | `service/backup/importers` | Interface: `getSupportedSource()` + `parse()` |
| `ChromeImporter.java` | `service/backup/importers` | Parses Chrome's exported CSV (name, url, username, password) |
| `FirefoxImporter.java` | `service/backup/importers` | Parses Firefox's exported JSON/CSV |
| `LastPassImporter.java` | `service/backup/importers` | Parses LastPass CSV export format |
| `OnePasswordImporter.java` | `service/backup/importers` | Parses 1Password CSV export format |
| `VaultSnapshotService.java` | `service/vault` | Point-in-time snapshot get + restore |
| `VaultService.java` | `service/vault` | `bulkInsert()` — batch-saves all imported entries |
| `EncryptionService.java` | `service/vault` | encrypt/decrypt/deriveKey — used in both import and export |
| `UserClient.java` | `client` | Feign — gets `masterPasswordHash` + `salt` for key derivation |
| `SecurityClient.java` | `client` | Audit log for `VAULT_EXPORTED` action |
| `VaultEntryRepository.java` | `repository` | `findByUserIdAndIsDeletedFalse()` — all active entries for export |
| `BackupExportRepository.java` | `repository` | Records export audit log rows |

---

## Feature 1 — Export Vault (JSON or CSV)

### What it produces

```
JSON format:
[
  {
    "title": "Netflix",
    "username": "john@gmail.com",
    "password": "MyP@ssw0rd!",      ← PLAINTEXT in export file
    "websiteUrl": "netflix.com",
    "notes": null,
    "isFavorite": false,
    "category": { "id": 2, "name": "Entertainment" },
    "folder": { "id": 1, "name": "Streaming" },
    "createdAt": "2026-01-15T10:00:00"
  },
  ...
]

CSV format:
title,username,password,website_url,notes,is_favorite
Netflix,john@gmail.com,MyP@ssw0rd!,netflix.com,,false
```

### ASCII Flow Diagram

```
FRONTEND
    |
    | GET /api/backup/export?format=JSON&password=optionalEncryptionPw
    | Authorization: Bearer <token>
    |
    v
[API GATEWAY (:8080)]
    |-- RateLimitGlobalFilter: rate check
    |-- JwtAuthFilter: /api/backup/export → PROTECTED → validates JWT
    |       Injects: X-User-Name: john, X-Is-Duress: false
    v
VAULT-SERVICE (:8082)
    |-- CorsConfig: valid Origin
    |-- GatewayAuthFilter: reads X-User-Name header ? sets SecurityContext
    |-- SecurityConfig: /api/backup/** → authenticated() ✓
    v
[BackupController.exportVault(username="john", format="JSON", password=null)]
    | username read from X-User-Name header (set by gateway)
    v
[ExportService.exportVault("john", "JSON", null)]
    |
    ├── STEP 1: GET USER + ENTRIES
    │   userClient.getUserDetailsByUsername("john")
    │       → Feign → user-service → { id:1, masterPasswordHash, salt }
    │
    │   vaultEntryRepository.findByUserIdAndIsDeletedFalse(1)
    │       → MYSQL: SELECT * FROM vault_entries
    │                WHERE user_id = 1 AND is_deleted = false
    │       → Returns: [entry5 (Netflix), entry7 (Gmail), ...]
    │
    ├── STEP 2: DERIVE VAULT AES KEY
    │   encryptionService.deriveKey(masterPasswordHash, salt)
    │       → PBKDF2WithHmacSHA256, 100k iterations → 256-bit SecretKey
    │
    ├── STEP 3: SERIALIZE + DECRYPT TO OUTPUT FORMAT
    │
    │   FORMAT = JSON → exportAsJson(entries, key)
    │       ObjectMapper with INDENT_OUTPUT (pretty-print)
    │       For each entry → entryToMap(entry, key):
    │           title      → plaintext (never encrypted)
    │           username   → decrypt(entry.username, key)   → "john@gmail.com"
    │           password   → decrypt(entry.password, key)   → "MyP@ssw0rd!"
    │           websiteUrl → plaintext
    │           notes      → decrypt(entry.notes, key) if not null
    │           isFavorite → boolean
    │           category   → { id, name } if present
    │           folder     → { id, name } if present
    │       mapper.writeValueAsString(exportData) → JSON string
    │
    │   FORMAT = CSV → exportAsCsv(entries, key)
    │       Header: "title,username,password,website_url,notes,is_favorite\n"
    │       For each entry → decrypt each field → escapeCsv() → append
    │       (escapeCsv wraps values containing commas/quotes/newlines in "double quotes")
    │
    ├── STEP 4: OPTIONAL FILE ENCRYPTION (if password provided)
    │   IF encryptionPassword != null:
    │       salt = UUID.randomUUID().toString()   ← random per-export salt
    │       ciphertext = encryptionService.encryptForExport(jsonData, password, salt)
    │       data = salt + ":" + ciphertext        ← "uuid-salt:Base64ciphertext"
    │       encrypted = true
    │
    ├── STEP 5: RECORD AUDIT LOG
    │   BackupExport { userId, format=JSON, fileName="vault_export_1711213500000.json",
    │                  entryCount=15, isEncrypted=false, createdAt=now }
    │   backupExportRepository.save() → MYSQL INSERT INTO backup_exports
    │
    │   securityClient.logAction("VAULT_EXPORTED", "Exported 15 entries in JSON format")
    │
    v
[RESPONSE 200 OK]
    | Body: ExportResponse {
    |     fileName:    "vault_export_1711213500000.json"
    |     format:      "JSON"
    |     entryCount:  15
    |     encrypted:   false
    |     data:        "[...]"     ← full JSON/CSV string
    |     exportedAt:  "2026-03-24T00:44:10"
    | }
```

---

## Feature 2 — Preview Export (No Download, No Audit)

```
FRONTEND (shows user what the export will look like before downloading)
    |
    | GET /api/backup/export/preview?format=CSV
    | Authorization: Bearer <token>
    |
    v
[API GATEWAY] → validates JWT → X-User-Name: john → vault-service
    |
    v
[GatewayAuthFilter] → reads X-User-Name ? sets SecurityContext
    |
    v
[BackupController.previewExport(format)] → username from X-User-Name header
    v
[ExportService.previewExport("john", "CSV")]
    |
    |-- Same as exportVault BUT:
    |       previewEntries = entries.stream().limit(3)   ← ONLY FIRST 3 ENTRIES
    |       No BackupExport row saved (no audit record created)
    |       No securityClient.logAction()
    |       encrypted = always false
    |
    v
[RESPONSE 200 OK]
    | Body: ExportResponse {
    |     fileName:   "preview.csv"
    |     entryCount: 15       ← total count (not 3!)
    |     data:       "title,username,...\nNetflix,john,...\n..."  ← only 3 rows
    | }
```

---

## Feature 3 — Import Vault (JSON or CSV)

### Two Paths Through Import (Encrypted vs Plain)

```
Request format detection:
    If data starts with '[' or '{' → valid JSON → NOT encrypted
    If data starts with "[uuid-salt]:[base64]" → ENCRYPTED export
        → looksEncrypted(): colonIdx > 0 AND colonIdx < 50 AND not JSON start char

ENCRYPTED PATH:
    colonIdx = data.indexOf(':')
    salt = data.substring(0, colonIdx)            → "550e8400-e29b-41d4..."
    ciphertext = data.substring(colonIdx + 1)     → Base64 ciphertext
    data = encryptionService.decryptForImport(ciphertext, request.password, salt)
        → AES-256 with password-derived key → original JSON or CSV string
    If decryption fails → return "Decryption failed. Password may be incorrect."
```

### ASCII Flow Diagram

```
FRONTEND
    |
    | POST /api/backup/import
    | Authorization: Bearer <token>
    | Body: {
    |     format: "JSON",
    |     data: "[{\"title\":\"Netflix\",\"password\":\"MyP@ss\",...}]",
    |     password: null   ← or decryption password if encrypted file
    | }
    |
    v
[API GATEWAY] → validates JWT → X-User-Name: john → vault-service
    |
    v
[GatewayAuthFilter] → reads X-User-Name ? sets SecurityContext
    |
    v
[BackupController.importVault(request)] → username from X-User-Name header
    v
[ImportService.importVault("john", request)]
    |
    ├── STEP 1: GET USER + DERIVE KEY
    │   userClient.getUserDetailsByUsername("john")
    │       → masterPasswordHash + salt
    │   encryptionService.deriveKey(hash, salt) → 256-bit AES key
    │
    ├── STEP 2: DETECT & DECRYPT ENCRYPTED FILES
    │   looksEncrypted(data)?
    │       IF YES and no password → return error: "Please provide the decryption password"
    │       IF YES and password provided:
    │           split on first ':' → salt + ciphertext
    │           decryptForImport(ciphertext, password, salt) → raw JSON/CSV
    │           (fails cleanly → "Decryption failed")
    │
    ├── STEP 3: PARSE BASED ON FORMAT
    │
    │   FORMAT = JSON → importJson(user, data, key)
    │       ObjectMapper.readValue → List<Map<String,Object>>
    │       categoryCache: Map<String, Category>
    │       folderCache:   Map<String, Folder>
    │
    │       For EACH entry map:
    │           password = entryMap.get("password") → "MyP@ssw0rd!"
    │
    │           CATEGORY HANDLING (with cache):
    │               categoryName = entryMap.category.name → "Entertainment"
    │               categoryCache.computeIfAbsent("Entertainment", name ->
    │                   categoryRepository.findByUserIdAndName(userId, name)
    │                       → exists? return it
    │                       → not found? create new Category → save → return
    │               ) → single DB lookup per unique category name
    │
    │           FOLDER HANDLING (with cache):
    │               folderName = entryMap.folder.name → "Streaming"
    │               folderCache.computeIfAbsent("Streaming", name ->
    │                   folderRepository.findByUserId(userId)
    │                       .stream().filter(f -> name.equals(f.getName()))
    │                       .findFirst()
    │                   → not found? create new Folder → save → return
    │               )
    │
    │           Build VaultEntry:
    │               .username(encrypt(rawUsername, key))    ← AES re-encrypt
    │               .password(encrypt(rawPassword, key))    ← AES re-encrypt
    │               .notes(encrypt(rawNotes, key))          ← AES re-encrypt
    │               .title(plaintext), .websiteUrl(plaintext)
    │           Add to entriesToSave list; success++
    │           Any entry failure → fail++, continue (never aborts the whole import)
    │
    │       vaultService.bulkInsert(username, entriesToSave)
    │           → vaultEntryRepository.saveAll() → MYSQL batch INSERT
    │
    │   FORMAT = CSV → importCsv(user, data, key)
    │       BufferedReader → skip header row
    │       For each line → split(",", -1) → 6 columns (title, user, pass, url, notes, fav)
    │       Encrypt each field → VaultEntry → add to list
    │       vaultService.bulkInsert() → batch INSERT
    │
    v
[RESPONSE 200 OK]
    | Body: ImportResult {
    |     totalProcessed: 15,
    |     successCount:   14,
    |     failCount:       1,   ← row with empty title AND url was skipped
    |     message: "Import completed"
    | }
```

---

## Feature 4 — Validate Import (Dry Run)

```
FRONTEND
    | (user selects a file → preview which rows will be imported before committing)
    |
    | POST /api/backup/import/validate
    | Authorization: Bearer <token>
    | Body: { format: "JSON", data: "..." }
    |
    v
[API GATEWAY] → validates JWT → X-User-Name: john → vault-service
    |
    v
[GatewayAuthFilter] → reads X-User-Name ? sets SecurityContext
    |
    v
[BackupController.validateImport(request)] → username from X-User-Name header
    v
[ImportService.validateImport("john", request)]
    |
    |-- @Transactional(readOnly = true)  ← NO WRITES to DB even if DB is called
    |
    |-- FORMAT = JSON → validateJson():
    |       ObjectMapper.readValue → parse JSON
    |       For each entry:
    │           IF title.isEmpty() AND websiteUrl.isEmpty() → fail++ (nothing to identify it by)
    │           ELSE → success++
    │       (categories and folders NOT created — pure validation only)
    |
    |-- FORMAT = CSV → validateCsv():
    |       Skip header row
    |       Count non-empty lines as success++
    |
    v
[RESPONSE 200 OK]
    | Body: ImportResult {
    |     totalProcessed: 15,
    |     successCount:   14,
    |     failCount:       1,
    |     message: "Validation passed"
    | }
    |
    | NO DB CHANGES — this is purely a parse + count operation
```

---

## Feature 5 — Import from Third-Party Password Managers

### The Factory Pattern — How it Works

```
Spring auto-wires all @Component classes implementing Importer
into ImporterFactory via constructor injection:

    public ImporterFactory(List<Importer> importers) {
        for (Importer importer : importers) {
            importerMap.put(
                importer.getSupportedSource().toUpperCase(),
                importer
            );
        }
    }

Result at startup:
    importerMap = {
        "CHROME"      → ChromeImporter instance
        "FIREFOX"     → FirefoxImporter instance
        "LASTPASS"    → LastPassImporter instance
        "1PASSWORD"   → OnePasswordImporter instance
    }

To add a new importer:
    1. Create class that implements Importer
    2. Add @Component → Spring auto-discovers it → auto-registered
    No changes to ImporterFactory, ThirdPartyImportService, or controller
```

### Four Importer Formats

| Source | CSV Columns | Notes |
| :--- | :--- | :--- |
| `CHROME` | name, url, username, password | Chrome's Password Manager export |
| `FIREFOX` | title, url, username, password, notes | Firefox Lockwise export |
| `LASTPASS` | url, username, password, totp, extra, name, grouping | LastPass CSV format |
| `1PASSWORD` | Title, Username, Password, Notes, URL | 1Password 1PIF/CSV export |

### Chrome CSV Column Mapping

```
Chrome export header row (skipped):
    name, url, username, password

Chrome CSV row:
    "Netflix","https://netflix.com","john@gmail.com","MyP@ss"

ChromeImporter.parse():
    parts = line.split(",", -1)
    VaultEntry {
        title       = parts[0]  → "Netflix"
        websiteUrl  = parts[1]  → "https://netflix.com"
        username    = parts[2]  → "john@gmail.com"  (stored PLAINTEXT here — not encrypted)
        password    = encrypt(parts[3], key)  → AES ciphertext
    }

NOTE: Chrome importer encrypts the password BUT stores username as plaintext.
      The vault schema expects username to also be encrypted.
      This is a known limitation — username shows as plaintext in vault.
```

### ASCII Flow Diagram

```
FRONTEND
    |
    | POST /api/backup/import-external
    | Authorization: Bearer <token>
    | Body: {
    |     source: "CHROME",
    |     data: "name,url,username,password\nNetflix,https://netflix.com,john,MyP@ss\n..."
    | }
    |
    v
[API GATEWAY] → validates JWT → X-User-Name: john → vault-service
    |
    v
[GatewayAuthFilter] → reads X-User-Name ? sets SecurityContext
    |
    v
[BackupController.importExternal(request)] → username from X-User-Name header
    v
[ThirdPartyImportService.importFromThirdParty("john", request)]
    |
    ├── GET USER + DERIVE KEY
    │   userClient.getUserDetailsByUsername("john") → masterHash + salt
    │   encryptionService.deriveKey(hash, salt) → AES key
    │
    ├── FACTORY DISPATCH
    │   importerFactory.getImporter("CHROME")
    │       → importerMap.get("CHROME") → ChromeImporter instance
    │   If null → throw IllegalArgumentException("Unsupported import source: CHROME")
    │
    ├── PARSE
    │   ChromeImporter.parse(csvData, user, key, encryptionService)
    │       → Reads CSV line by line
    │       → Encrypts password per entry
    │       → Returns List<VaultEntry>
    │
    ├── BULK INSERT (if any entries parsed)
    │   vaultService.bulkInsert(username, entries)
    │       → Saves all to vault_entries via batch INSERT
    │       → Each entry triggers securityClient.logAction("ENTRY_CREATED")
    │
    v
[RESPONSE 200 OK]
    | Body: ImportResult {
    |     totalProcessed: 12,
    |     successCount:   12,
    |     failCount:       0,
    |     message: "CHROME import completed"
    | }
```

---

## Feature 6 — List Supported Third-Party Formats

```
FRONTEND
    |
    | GET /api/backup/import-external/formats
    | Authorization: Bearer <token>
    |
    v
[API GATEWAY] → validates JWT → X-User-Name: john → vault-service
    |
    v
[BackupController.getSupportedFormats()]
    v
[ThirdPartyImportService.getSupportedFormats()]
    |
    |-- importerFactory.getSupportedFormats()
    |       → return new ArrayList<>(importerMap.keySet())
    |       → ["CHROME", "FIREFOX", "LASTPASS", "1PASSWORD"]
    |
    v
[RESPONSE 200 OK] → ["CHROME", "FIREFOX", "LASTPASS", "1PASSWORD"]
```

---

## Feature 7 — Get All Vault Snapshots

```
FRONTEND
    |
    | GET /api/backup/snapshots
    | Authorization: Bearer <token>
    | X-User-Name: john  ← read by BackupController from gateway header
    |
    v
[API GATEWAY] → validates JWT → X-User-Name: john → vault-service
    |
    v
[GatewayAuthFilter] → reads X-User-Name ? sets SecurityContext
    |
    v
[BackupController.getAllSnapshots()] → username from X-User-Name header
    v
[VaultSnapshotService.getAllSnapshots("john")]
    |
    |-- userClient.getUserDetailsByUsername("john") → user.id = 1
    |
    |-- vaultSnapshotRepository.findByVaultEntryUserIdOrderByChangedAtDesc(1)
    |       → MYSQL: SELECT vs.* FROM vault_snapshots vs
    |                JOIN vault_entries ve ON vs.vault_entry_id = ve.id
    |                WHERE ve.user_id = 1
    |                ORDER BY vs.changed_at DESC
    |
    |-- mapToResponse(snapshot):
    |       { id, entryTitle, changedAt, password: "******" }
    |       (password ALWAYS masked in response)
    |
    v
[RESPONSE 200 OK]
    | Body: [
    |   { id: 5, entryTitle: "Netflix", changedAt: "2026-03-20T10:00:00", password: "******" },
    |   { id: 4, entryTitle: "Gmail",   changedAt: "2026-03-15T08:00:00", password: "******" },
    |   ...
    | ]
```

---

## Feature 8 — Restore from a Snapshot

```
FRONTEND
    | (user wants to roll back a vault entry to a previous password)
    |
    | POST /api/backup/snapshots/{id}/restore
    | Authorization: Bearer <token>
    | e.g. POST /api/backup/snapshots/5/restore
    |
    v
[API GATEWAY] → validates JWT → X-User-Name: john → vault-service
    |
    v
[GatewayAuthFilter] → reads X-User-Name ? sets SecurityContext
    |
    v
[BackupController.restoreSnapshot(id)] → username from X-User-Name header
    v
[VaultSnapshotService.restoreSnapshot("john", 5)]
    |
    ├── STEP 1: GET USER
    │   userClient.getUserDetailsByUsername("john") → user.id = 1
    │
    ├── STEP 2: LOAD SNAPSHOT
    │   vaultSnapshotRepository.findById(5)
    │       → MYSQL SELECT FROM vault_snapshots WHERE id = 5
    │   If not found → 404 ResourceNotFoundException
    │
    ├── STEP 3: OWNERSHIP CHECK
    │   snapshot.vaultEntry.userId == user.id?
    │   If not → AuthenticationException "Not authorized to restore this snapshot"
    │
    ├── STEP 4: TAKE A NEW SNAPSHOT OF CURRENT STATE
    │   createSnapshot(entry)
    │       → Saves CURRENT password to vault_snapshots
    │       → So this restore can itself be undone later
    │       (prevents losing history when rolling back)
    │
    ├── STEP 5: RESTORE
    │   entry.setPassword(snapshot.password)   ← OLD encrypted ciphertext
    │   entry.setUpdatedAt(LocalDateTime.now())
    │   vaultEntryRepository.save(entry) → MYSQL UPDATE vault_entries
    │
    │   Note: No key derivation needed — snapshot stores the CIPHERTEXT
    │   (encrypted with the same vault key), not plaintext
    │
    v
[RESPONSE 200 OK (empty body)]

TIMELINE AFTER RESTORE:
    Snapshot 1: "password123"  (oldest)
    Snapshot 2: "P@ssw0rd!"
    ─── restore to Snapshot 1 triggered ───
    Snapshot 3: "P@ssw0rd!"   ← NEW snapshot of current state (the one being replaced)
    CURRENT:    "password123"  ← the restored value
```

---

## End-to-End Data Flow Summary

```
EXPORT:
    vault_entries (encrypted) → decrypt with vault key → JSON/CSV (plaintext)
    Optional: re-encrypt JSON/CSV with user-typed export password

IMPORT (Own Format):
    JSON/CSV file (plaintext) → optional: decrypt file with export password
                              → encrypt each field with vault key
                              → vault_entries (encrypted)

IMPORT (Third-Party):
    Chrome/Firefox/LastPass/1Password CSV → parse with format-specific importer
                                         → encrypt passwords with vault key
                                         → vault_entries (encrypted)

SNAPSHOT:
    vault_entries.password (encrypted ciphertext) → vault_snapshots.password
    (stored as-is — same vault key, same AES-GCM ciphertext, no re-encryption needed)
```

---

## Dependencies Summary

| Dependency | Used For |
| :--- | :--- |
| `jackson-databind` (`ObjectMapper`) | JSON serialization in export + deserialization in import |
| `javax.crypto` (AES-GCM) | Vault key derivation + field-level encrypt/decrypt |
| `java.io.BufferedReader` | CSV line-by-line parsing in import |
| `spring-data-jpa` | `VaultEntryRepository`, `BackupExportRepository`, `VaultSnapshotRepository` |
| `spring-cloud-openfeign` | `UserClient` (vault key materials), `SecurityClient` (audit log) |
| Design Pattern: **Factory** | `ImporterFactory` — open/closed principle for adding new importers |
| Design Pattern: **Strategy** | `Importer` interface — each importer is a swappable parse strategy |

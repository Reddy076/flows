# Vault Service — Complete Feature List

A top-level map of every feature in the `vault-service`, organised by controller and service.

---

## 1. Vault Entries (`/api/vault`)

Core CRUD operations on stored passwords, notes, and secure data.

| Feature | Method | Endpoint | Service Class |
| :--- | :--- | :--- | :--- |
| Create vault entry | `POST` | `/api/vault` | `VaultService` |
| Get all entries | `GET` | `/api/vault` | `VaultService` |
| Get single entry | `GET` | `/api/vault/{id}` | `VaultService` |
| Update entry | `PUT` | `/api/vault/{id}` | `VaultService` |
| Delete entry (to trash) | `DELETE` | `/api/vault/{id}` | `VaultService` |
| Search & filter entries | `GET` | `/api/vault/search` | `VaultService` |
| Filter by category/folder | `GET` | `/api/vault/filter` | `VaultService` |
| Get recent entries | `GET` | `/api/vault/recent` | `VaultService` |
| Get recently used entries | `GET` | `/api/vault/recently-used` | `VaultService` |
| Get all favorites | `GET` | `/api/vault/favorites` | `VaultService` |
| Toggle favorite | `PUT` | `/api/vault/{id}/favorite` | `VaultService` |
| Bulk delete (to trash) | `POST` | `/api/vault/entries/bulk-delete` | `VaultService` |
| View decrypted password | `POST` | `/api/vault/entries/{id}/view-password` | `VaultService` |
| Toggle sensitive flag | `PUT` | `/api/vault/entries/{id}/sensitive` | `VaultService` |
| Access sensitive entry (master password required) | `POST` | `/api/vault/{id}/sensitive-view` | `VaultService` |
| Get entry password history | `GET` | `/api/vault/entries/{id}/history` | `VaultSnapshotService` |

### Trash Management

| Feature | Method | Endpoint | Service Class |
| :--- | :--- | :--- | :--- |
| List trash entries | `GET` | `/api/vault/trash` | `VaultTrashService` |
| Get trash count | `GET` | `/api/vault/trash/count` | `VaultTrashService` |
| Restore single entry | `POST` | `/api/vault/trash/{id}/restore` | `VaultTrashService` |
| Restore all from trash | `POST` | `/api/vault/trash/restore-all` | `VaultTrashService` |
| Permanently delete one | `DELETE` | `/api/vault/trash/{id}` | `VaultTrashService` |
| Empty trash (delete all) | `DELETE` | `/api/vault/trash/empty` | `VaultTrashService` |

**Key Design Notes:**
- All passwords are stored **AES-256 encrypted** — the raw text is never written to DB
- Deleting an entry moves it to a soft-deleted trash state (`is_deleted = true`) — not immediately wiped
- Sensitive entries (`is_highly_sensitive = true`) require master password re-verification before decryption can be returned
- Every create/update/delete records a snapshot for password history tracking

---

## 2. Folders (`/api/folders`)

Organisational folders to group vault entries — supports nesting (parent/child hierarchy).

| Feature | Method | Endpoint | Service Class |
| :--- | :--- | :--- | :--- |
| Get all folders | `GET` | `/api/folders` | `FolderService` |
| Get folder by ID | `GET` | `/api/folders/{id}` | `FolderService` |
| Create folder (optional parent) | `POST` | `/api/folders` | `FolderService` |
| Rename folder | `PUT` | `/api/folders/{id}` | `FolderService` |
| Move folder to new parent | `PUT` | `/api/folders/{id}/move` | `FolderService` |
| Delete folder | `DELETE` | `/api/folders/{id}` | `FolderService` |
| List entries inside folder | `GET` | `/api/folders/{id}/entries` | `VaultService` |

**Key Design Notes:**
- Folders support **nested hierarchy** via an optional `parentFolderId`
- Deleting a folder does not cascade-delete its entries — entries become uncategorised

---

## 3. Categories (`/api/categories`)

User-defined labels/tags for grouping vault entries by type (e.g. Social, Banking, Work).

| Feature | Method | Endpoint | Service Class |
| :--- | :--- | :--- | :--- |
| Get all categories | `GET` | `/api/categories` | `CategoryService` |
| Get category by ID | `GET` | `/api/categories/{id}` | `CategoryService` |
| Create category | `POST` | `/api/categories` | `CategoryService` |
| Update category | `PUT` | `/api/categories/{id}` | `CategoryService` |
| Delete category | `DELETE` | `/api/categories/{id}` | `CategoryService` |
| List entries in category | `GET` | `/api/categories/{id}/entries` | `VaultService` |

---

## 4. Secure Sharing (`/api/shares`)

Time-limited, encrypted one-time or expiring links to share a password with another person without revealing the vault.

| Feature | Method | Endpoint | Auth Required | Service Class |
| :--- | :--- | :--- | :--- | :--- |
| Create secure share link | `POST` | `/api/shares` | ✅ Yes | `SecureShareService` |
| Access shared password via token | `GET` | `/api/shares/{token}` | ❌ No (public) | `SecureShareService` |
| List my active shares | `GET` | `/api/shares` | ✅ Yes | `SecureShareService` |
| Revoke a share | `DELETE` | `/api/shares/{id}` | ✅ Yes | `SecureShareService` |
| List shares received (by email) | `GET` | `/api/shares/received` | ✅ Yes | `SecureShareService` |

**Key Design Notes:**
- Share tokens are one-time-use or time-expiring (configured per share)
- The shared password is **re-encrypted** with a share-specific key (`ShareEncryptionService`) — the vault AES key is NOT exposed
- The recipient accesses the share via a public URL — no login needed
- `ShareExpirationService` is a scheduler that cleans up expired shares
- `ShareTokenGenerator` creates cryptographically random URL-safe token strings

---

## 5. Backup & Import/Export (`/api/backup`)

Export the entire vault to JSON or CSV and import from native format or third-party password managers.

| Feature | Method | Endpoint | Service Class |
| :--- | :--- | :--- | :--- |
| Export vault (JSON or CSV) | `GET` | `/api/backup/export` | `ExportService` |
| Preview export (no download) | `GET` | `/api/backup/export/preview` | `ExportService` |
| Import vault from JSON/CSV | `POST` | `/api/backup/import` | `ImportService` |
| Validate import (dry run) | `POST` | `/api/backup/import/validate` | `ImportService` |
| Import from 3rd-party managers | `POST` | `/api/backup/import-external` | `ThirdPartyImportService` |
| List supported 3rd-party formats | `GET` | `/api/backup/import-external/formats` | `ThirdPartyImportService` |
| Get all vault snapshots | `GET` | `/api/backup/snapshots` | `VaultSnapshotService` |
| Restore from a snapshot | `POST` | `/api/backup/snapshots/{id}/restore` | `VaultSnapshotService` |

### Supported Third-Party Import Formats

| Format | Importer Class |
| :--- | :--- |
| Chrome CSV | `ChromeImporter` |
| Firefox JSON | `FirefoxImporter` |
| LastPass CSV | `LastPassImporter` |
| 1Password CSV | `OnePasswordImporter` |

**Key Design Notes:**
- Import uses the `ImporterFactory` pattern — `ImporterFactory.getImporter(format)` returns the right `Importer` implementation
- Validate import is a **dry run** — parses and validates the data but does NOT write to DB
- Snapshots are point-in-time full copies of vault state used for rollback

---

## 6. Vault Timeline & Analytics (`/api/timeline`)

A chronological activity feed and usage analytics dashboard for the vault.

| Feature | Method | Endpoint | Service Class |
| :--- | :--- | :--- | :--- |
| Get full activity timeline | `GET` | `/api/timeline` | `VaultTimelineService` |
| Get timeline summary stats | `GET` | `/api/timeline/summary` | `VaultTimelineService` |
| Get timeline for a single entry | `GET` | `/api/timeline/entry/{entryId}` | `VaultTimelineService` |
| Get chart-ready stats | `GET` | `/api/timeline/stats` | `VaultTimelineService` |
| Get 30-day access heatmap | `GET` | `/api/timeline/heatmap` | `AccessHeatmapService` |

**Key Design Notes:**
- Filters: `?days=N` (last N days), `?category=VAULT|AUTH|BREACH|SHARING|BACKUP|SECURITY`
- `ActivityAggregator` groups raw events into daily/weekly/monthly summaries
- `TimelineEventEnricher` adds human-readable descriptions to raw events
- Heatmap returns hourly + daily counts for a 30-day grid chart

---

## 7. Internal / Service-to-Service (`/internal/vault`)

Endpoints called by other microservices via Feign clients — never exposed to the browser.

| Feature | Method | Endpoint | Called By |
| :--- | :--- | :--- | :--- |
| Get decrypted vault entries for security analysis | `GET` | `/internal/vault/{userId}/decrypted-entries` | `security-service` (AI analysis) |
| Get vault stats for user dashboard | `GET` | `/internal/vault/stats/{userId}` | `user-service` (dashboard tile counts) |

**Key Design Notes:**
- `decrypted-entries`: derives the AES key from `masterPasswordHash + salt` (fetched from `user-service` via Feign), decrypts each password, and returns plaintext — only used so `security-service` can score password strength
- `stats/{userId}`: returns `vaultCount`, `favoriteCount`, `trashCount` — used by `user-service` to display the dashboard summary tiles
- These endpoints are `permitAll()` in `SecurityConfig` — protected only by Docker network isolation

---

## 8. Background / Automated Features

Tasks that run automatically on a schedule — no HTTP trigger required.

| Feature | Class | Schedule | What it does |
| :--- | :--- | :--- | :--- |
| Trash auto-cleanup | `TrashCleanupScheduler` | Configurable (cron) | Permanently deletes trash entries older than N days |
| Expired share cleanup | `SecureShareCleanupScheduler` | Configurable (cron) | Deletes secure share records that have passed their expiry time |

---

## 9. Cross-Cutting / Security Infrastructure

Features that apply to every request in the vault-service — not tied to a single endpoint.

| Feature | Class | Role |
| :--- | :--- | :--- |
| JWT trust filter | `GatewayAuthFilter` | Reads X-User-Name set by gateway ? populates SecurityContext |
| AES-256 encryption/decryption | `EncryptionUtil` | Encrypts all passwords before DB write, decrypts on read |
| AES key derivation from master password | `EncryptionUtil.deriveKey()` | PBKDF2 → 256-bit AES key from BCrypt hash + salt |
| CORS policy | `CorsConfig` | Allows Angular frontend to call vault-service APIs |
| AOP method-level logging | `LoggingAspect` | Auto-logs all service/repository/controller method calls |
| Swagger / OpenAPI docs | `OpenApiConfig` | Provides JWT bearer auth input in Swagger UI |
| Feign client to user-service | `UserClient` | Fetches masterPasswordHash, salt, 2FA secret per request |

---

## Architecture Summary

```
FRONTEND (Angular)
    │
    ▼
API GATEWAY (:8080)       ← JWT validation, rate limiting, CORS
    │
    ▼
vault-service (:8082)
    ├── VaultController           → /api/vault/**
    ├── FolderController          → /api/folders/**
    ├── CategoryController        → /api/categories/**
    ├── SecureShareController     → /api/shares/**
    ├── BackupController          → /api/backup/**
    ├── VaultTimelineController   → /api/timeline/**
    └── VaultInternalController   → /internal/vault/**  (service-to-service only)

vault-service calls:
    ├── user-service  (UserClient Feign) → get masterPasswordHash + salt for encryption
    └── MySQL          → all vault_entries, folders, categories, shares, snapshots

Other services call vault-service:
    ├── user-service  (VaultClient Feign) → dashboard stats tiles
    └── security-service (VaultAnalysisClient Feign) → decrypted entries for AI analysis
```

---

## Service Classes — Quick Reference

| Service | Responsibility |
| :--- | :--- |
| `VaultService` | Core CRUD for vault entries, search, favorites, sensitive access |
| `VaultTrashService` | Trash list, restore, permanent delete, empty trash |
| `VaultSnapshotService` | Password history snapshots, vault-level restore |
| `FolderService` | Folder CRUD, nesting, move operations |
| `CategoryService` | Category CRUD |
| `EncryptionService` | Wrapper around `EncryptionUtil` for service-level encrypt/decrypt |
| `DuressService` | Returns decoy vault data when a duress JWT token is detected |
| `SecureShareService` | Create, list, revoke, access shares |
| `ShareEncryptionService` | Re-encrypts shared passwords with share-specific keys |
| `ShareExpirationService` | Checks if a share token has expired |
| `ShareTokenGenerator` | Generates cryptographically random share tokens |
| `ExportService` | Export vault to JSON/CSV |
| `ImportService` | Import from native JSON/CSV format |
| `ThirdPartyImportService` | Import from Chrome, Firefox, LastPass, 1Password |
| `VaultTimelineService` | Full timeline, summary, per-entry timeline, stats |
| `AccessHeatmapService` | 30-day hourly/daily heatmap data |
| `ActivityAggregator` | Aggregates raw events into chart data |
| `TimelineEventEnricher` | Adds human-readable labels to raw timeline events |

# Vault Trash Management — Complete Flow Document
> Part of the `vault-service` deep-dive series.

This document explains the full end-to-end flow of every Vault Trash feature,
from the frontend HTTP request all the way through to the database — including every
file involved, every dependency used, and ASCII diagrams for each flow.

---

## How The Trash Works — The Lifecycle of a Vault Entry

```
LIFECYCLE:
┌──────────────┐   DELETE    ┌───────────────┐   30 DAYS   ┌──────────────────┐
│  ACTIVE VAULT │ ─────────► │  TRASH (soft  │ ──────────► │  AUTO-DELETED BY │
│  is_deleted=  │            │  is_deleted=  │ scheduler   │  TrashCleanup-   │
│  false        │            │  true)        │             │  Scheduler       │
└──────────────┘            └───────────────┘             └──────────────────┘
                                     │
                              RESTORE │           PERMANENT DELETE
                                     ▼                   │
                            ┌──────────────┐             ▼
                            │  ACTIVE VAULT │      ALL DATA WIPED
                            │  is_deleted=  │  (snapshots + shares too)
                            │  false        │
                            └──────────────┘

KEY POINT: "Deleting" an entry does NOT erase the row — it sets is_deleted=true.
           The row stays in vault_entries with all encrypted data intact,
           ready to be restored at any time within the 30-day window.
```

---

## Files Involved

| File | Package | Role |
| :--- | :--- | :--- |
| `VaultController.java` | `controller` | HTTP entry point for all `/api/vault/trash/**` endpoints |
| `VaultTrashService.java` | `service/vault` | All trash business logic — list, restore, permanent delete |
| `VaultTrashRepository.java` | `repository` | Queries `vault_entries` filtered by `is_deleted = true` |
| `VaultSnapshotRepository.java` | `repository` | Deletes snapshots when entry is permanently deleted |
| `SecureShareRepository.java` | `repository` | Deletes shares when entry is permanently deleted |
| `UserClient.java` | `client` | Feign — gets `user.id` from username (for DB queries) |
| `TrashCleanupScheduler.java` | `scheduler` | Auto-runs `cleanupExpired()` nightly |

---

## The TrashEntryResponse — What is Returned

Unlike vault list responses, trash responses do NOT include decrypted passwords.
They only return metadata needed for the trash UI:

```java
TrashEntryResponse {
    id          → entry ID
    title       → plaintext entry title
    websiteUrl  → plaintext URL
    categoryName→ category label (if any)
    folderName  → folder label (if any)
    deletedAt   → when it was deleted  (e.g. "2026-03-01T10:00:00")
    expiresAt   → deletedAt + 30 days  (e.g. "2026-03-31T10:00:00")
    daysRemaining → Math.max(0, days between NOW and expiresAt)
                  → e.g. 15  (shown as "15 days left to restore")
}
```

---

## How Every Trash Request Passes Through the Gateway

```
FRONTEND (Angular)
    |
    | GET /api/vault/trash   (or POST /trash/{id}/restore, etc.)
    | Authorization: Bearer eyJ...
    |
    v
[API GATEWAY]
    |-- JwtAuthFilter: validates JWT → adds X-User-Name header, X-Is-Duress header
    |-- RateLimitGlobalFilter: 100 req/min
    v
[vault-service Spring Security]
    |-- CorsConfig: validates Origin
    |-- GatewayAuthFilter: reads X-User-Name header ? sets SecurityContext
    |-- SecurityConfig: /api/vault/trash/** → anyRequest().authenticated()
    v
[VaultController]
    | getCurrentUsername() → from SecurityContextHolder
```

---

## Feature 1 — List Trash Entries

### What it does
Returns all vault entries currently sitting in the trash (soft-deleted), including
how many days remain before each is auto-deleted.

### ASCII Flow Diagram

```
FRONTEND
    |
    | GET /api/vault/trash
    | Authorization: Bearer <token>
    |
    v
[VaultController.getTrashEntries()]
    | getCurrentUsername() → "john"
    v
[VaultTrashService.getTrashEntries("john")]
    |
    |-- STEP 1: GET USER ID
    |   userClient.getUserDetailsByUsername("john")
    |       → Feign: GET http://user-service/api/users/internal/by-username/john
    |       → returns { id: 1, ... }
    |
    |-- STEP 2: FETCH ALL TRASHED ENTRIES
    |   vaultTrashRepository.findByUserIdAndIsDeletedTrue(userId=1)
    |       → MYSQL: SELECT * FROM vault_entries
    |                WHERE user_id = 1 AND is_deleted = true
    |
    |-- STEP 3: MAP TO RESPONSE (for each entry)
    |   mapToTrashResponse(entry):
    |       expiresAt = entry.deletedAt + 30 days
    |           e.g. deletedAt="2026-03-01" → expiresAt="2026-03-31"
    |       daysRemaining = Math.max(0, ChronoUnit.DAYS.between(NOW, expiresAt))
    |           e.g. NOW="2026-03-16" → daysRemaining = 15
    |       Build TrashEntryResponse:
    |           { id, title, websiteUrl, categoryName, folderName,
    |             deletedAt, expiresAt, daysRemaining: 15 }
    |       NO passwords, NO decryption needed
    |
    v
[RESPONSE 200 OK]
    | Body: [
    |   {
    |     id: 5,
    |     title: "Netflix",
    |     websiteUrl: "netflix.com",
    |     categoryName: "Entertainment",
    |     deletedAt: "2026-03-01T10:00:00",
    |     expiresAt: "2026-03-31T10:00:00",
    |     daysRemaining: 15
    |   },
    |   ...
    | ]
```

---

## Feature 2 — Get Trash Count

### What it does
Returns the total number of entries in the trash — used for the trash badge/counter
in the UI (e.g. the 🗑️ icon shows "3").

### ASCII Flow Diagram

```
FRONTEND
    |
    | GET /api/vault/trash/count
    | Authorization: Bearer <token>
    |
    v
[API GATEWAY] → validates JWT → X-User-Name: john → vault-service
    |
    v
[GatewayAuthFilter] → reads X-User-Name ? sets SecurityContext
    |
    v
[VaultController.getTrashCount()] → getCurrentUsername() → "john"
    v
[VaultTrashService.getTrashCount("john")]
    |
    |-- userClient.getUserDetailsByUsername("john") → user.id = 1
    |
    |-- vaultTrashRepository.countByUserIdAndIsDeletedTrue(1)
    |       → MYSQL: SELECT COUNT(*) FROM vault_entries
    |                WHERE user_id = 1 AND is_deleted = true
    |       → Returns: 3
    |
    v
[RESPONSE 200 OK] → { count: 3 }
```

---

## Feature 3 — Restore Single Entry

### What it does
Takes a trashed entry out of the trash and returns it to the active vault.
Clears the `is_deleted` flag and removes the `deleted_at` timestamp.

### ASCII Flow Diagram

```
FRONTEND
    | (user clicks "Restore" on a specific trash item)
    |
    | POST /api/vault/trash/{id}/restore
    | Authorization: Bearer <token>
    | e.g. POST /api/vault/trash/5/restore
    |
    v
[API GATEWAY] → validates JWT → X-User-Name: john → vault-service
    |
    v
[GatewayAuthFilter] → reads X-User-Name ? sets SecurityContext
    |
    v
[VaultController.restoreEntry(id)] → getCurrentUsername() → "john"
    v
[VaultTrashService.restoreEntry("john", 5)]
    |
    |-- STEP 1: GET USER ID
    |   userClient.getUserDetailsByUsername("john") → user.id = 1
    |
    |-- STEP 2: FIND THE TRASHED ENTRY (with ownership check)
    |   vaultTrashRepository.findByIdAndUserIdAndIsDeletedTrue(5, 1)
    |       → MYSQL: SELECT * FROM vault_entries
    |                WHERE id = 5 AND user_id = 1 AND is_deleted = true
    |   If not found → ResourceNotFoundException
    |       (entry either doesn't exist, belongs to another user,
    |        OR was never in the trash — all return same error for safety)
    |
    |-- STEP 3: RESTORE
    |   entry.setIsDeleted(false)       ← back in active vault
    |   entry.setDeletedAt(null)        ← clear the deletion timestamp
    |   vaultTrashRepository.save(entry)
    |       → MYSQL UPDATE vault_entries
    |            SET is_deleted = false, deleted_at = null
    |            WHERE id = 5
    |
    |-- LOG: "Restored vault entry 5 for user john"
    |
    v
[RESPONSE 200 OK]
    | Body: TrashEntryResponse
    |   { id: 5, title: "Netflix", ..., daysRemaining: 0 }
    |   (daysRemaining = 0 because expiresAt is nil now that it's restored)

EFFECT IN UI:
    Entry disappears from trash list
    Entry reappears in main vault list with all original data intact
    (encrypted password, username, notes — all preserved unchanged)
```

---

## Feature 4 — Restore All from Trash

### What it does
Restores every single entry in the trash back to the active vault in one operation.
Useful for accidentally selecting "delete all" or clearing the trash by mistake.

### ASCII Flow Diagram

```
FRONTEND
    | (user clicks "Restore All" button)
    |
    | POST /api/vault/trash/restore-all
    | Authorization: Bearer <token>
    |
    v
[API GATEWAY] → validates JWT → X-User-Name: john → vault-service
    |
    v
[GatewayAuthFilter] → reads X-User-Name ? sets SecurityContext
    |
    v
[VaultController.restoreAll()] → getCurrentUsername() → "john"
    v
[VaultTrashService.restoreAll("john")]
    |
    |-- userClient.getUserDetailsByUsername("john") → user.id = 1
    |
    |-- STEP 1: FETCH ALL TRASHED ENTRIES
    |   vaultTrashRepository.findByUserIdAndIsDeletedTrue(1)
    |       → MYSQL: SELECT * FROM vault_entries
    |                WHERE user_id = 1 AND is_deleted = true
    |       → Returns: [entry5, entry7, entry12]
    |
    |-- STEP 2: RESTORE ALL IN MEMORY
    |   trashedEntries.forEach(entry -> {
    |       entry.setIsDeleted(false)
    |       entry.setDeletedAt(null)
    |   })
    |
    |-- STEP 3: BATCH SAVE
    |   vaultTrashRepository.saveAll(trashedEntries)
    |       → MYSQL batch UPDATE vault_entries
    |            SET is_deleted = false, deleted_at = null
    |            WHERE id IN (5, 7, 12)
    |   (single batch query instead of 3 individual UPDATEs)
    |
    |-- LOG: "Restored 3 trashed entries for user john"
    |
    v
[RESPONSE 200 OK] → { message: "All trash entries restored successfully" }
```

---

## Feature 5 — Permanently Delete One Entry

### What it does
Irreversibly deletes a single entry from the trash — cannot be recovered.
Also cascades to delete all related data: password history snapshots and active share links.

### ASCII Flow Diagram

```
FRONTEND
    | (user clicks "Delete Forever" on a specific trash item)
    |
    | DELETE /api/vault/trash/{id}
    | Authorization: Bearer <token>
    | e.g. DELETE /api/vault/trash/5
    |
    v
[API GATEWAY] → validates JWT → X-User-Name: john → vault-service
    |
    v
[GatewayAuthFilter] → reads X-User-Name ? sets SecurityContext
    |
    v
[VaultController.permanentDelete(id)] → getCurrentUsername() → "john"
    v
[VaultTrashService.permanentDelete("john", 5)]
    |
    |-- STEP 1: GET USER + OWNERSHIP CHECK
    |   userClient.getUserDetailsByUsername("john") → user.id = 1
    |   vaultTrashRepository.findByIdAndUserIdAndIsDeletedTrue(5, 1)
    |       → MYSQL SELECT (must be in trash AND owned by user)
    |   If not found → ResourceNotFoundException
    |
    |-- STEP 2: CASCADE DELETE RELATED DATA (order matters!)
    |   secureShareRepository.deleteByVaultEntryId(5)
    |       → MYSQL DELETE FROM secure_shares WHERE vault_entry_id = 5
    |       (any links shared by this entry become permanently dead)
    |
    |   vaultSnapshotRepository.deleteByVaultEntryId(5)
    |       → MYSQL DELETE FROM vault_snapshots WHERE vault_entry_id = 5
    |       (all password change history is wiped)
    |
    |-- STEP 3: DELETE THE ENTRY ITSELF
    |   vaultTrashRepository.delete(entry)
    |       → MYSQL DELETE FROM vault_entries WHERE id = 5
    |       (the row is physically removed from the DB)
    |
    |-- LOG: "Permanently deleted vault entry 5 for user john"
    |
    v
[RESPONSE 200 OK] → { message: "Entry permanently deleted" }

WHAT IS DESTROYED (in order):
    1. secure_shares rows for entry 5      → all share links broken
    2. vault_snapshots rows for entry 5    → all password history gone
    3. vault_entries row for entry 5       → the entry itself gone
```

### Why delete shares and snapshots first?

```
Foreign key constraints:
    secure_shares.vault_entry_id → FK → vault_entries.id
    vault_snapshots.vault_entry_id → FK → vault_entries.id

If we deleted vault_entries first → MySQL would throw FK constraint violation
So: delete child records first, then the parent row
```

---

## Feature 6 — Empty Trash (Delete All)

### What it does
Permanently deletes **every** entry in the trash — the nuclear option.
Each entry's shares and snapshots are cascade-deleted before the entry itself.

### ASCII Flow Diagram

```
FRONTEND
    | (user clicks "Empty Trash" / "Delete All" button)
    |
    | DELETE /api/vault/trash/empty
    | Authorization: Bearer <token>
    |
    v
[API GATEWAY] → validates JWT → X-User-Name: john → vault-service
    |
    v
[GatewayAuthFilter] → reads X-User-Name ? sets SecurityContext
    |
    v
[VaultController.emptyTrash()] → getCurrentUsername() → "john"
    v
[VaultTrashService.emptyTrash("john")]
    |
    |-- STEP 1: GET USER + FETCH ALL TRASHED ENTRIES
    |   userClient.getUserDetailsByUsername("john") → user.id = 1
    |   vaultTrashRepository.findByUserIdAndIsDeletedTrue(1)
    |       → MYSQL SELECT * WHERE user_id = 1 AND is_deleted = true
    |       → Returns: [entry5, entry7, entry12, ...]
    |
    |-- STEP 2: CASCADE DELETE FOR EACH ENTRY
    |   trashedEntries.forEach(entry -> {
    |       secureShareRepository.deleteByVaultEntryId(entry.id)
    |           → MYSQL DELETE FROM secure_shares WHERE vault_entry_id = ?
    |       vaultSnapshotRepository.deleteByVaultEntryId(entry.id)
    |           → MYSQL DELETE FROM vault_snapshots WHERE vault_entry_id = ?
    |   })
    |
    |-- STEP 3: DELETE ALL ENTRIES
    |   vaultTrashRepository.deleteAll(trashedEntries)
    |       → MYSQL DELETE FROM vault_entries WHERE id IN (5, 7, 12, ...)
    |
    |-- LOG: "Emptied trash (3 entries) for user john"
    |
    v
[RESPONSE 200 OK] → { message: "Trash emptied successfully" }
```

---

## Feature 7 — Auto-Cleanup Scheduler (Background)

### What it does
Runs automatically on a schedule (triggered by `TrashCleanupScheduler`).
Permanently deletes any trash entry that has been in the trash for more than
**30 days** — even if the user never manually emptied it.

### ASCII Flow Diagram

```
[SPRING TASK SCHEDULER] — fires on configured cron
    |
    v
[TrashCleanupScheduler.runCleanup()]  (scheduler class)
    |
    v
[VaultTrashService.cleanupExpired()]
    |
    |-- expiry = LocalDateTime.now().minusDays(30)
    |       e.g. now = 2026-03-24 → expiry = 2026-02-22
    |
    |-- vaultTrashRepository.findExpiredTrashEntries(expiry)
    |       → MYSQL: SELECT * FROM vault_entries
    |                WHERE is_deleted = true
    |                AND deleted_at < '2026-02-22'
    |       → Returns: all entries deleted >30 days ago
    |
    |-- IF expired list is not empty:
    |   For each expired entry:
    |       secureShareRepository.deleteByVaultEntryId(entry.id)
    |       vaultSnapshotRepository.deleteByVaultEntryId(entry.id)
    |   vaultTrashRepository.deleteAll(expired)
    |
    |-- LOG: "Cleaned up 2 expired trash entries"
    |
    v
[DONE]

USER PERSPECTIVE:
    Day 0:  Entry moved to trash → daysRemaining = 30
    Day 15: Entry still visible → daysRemaining = 15
    Day 30: Scheduler fires at midnight → entry permanently deleted
    Day 30+: Entry is gone, cannot be recovered
```

---

## Complete Lifecycle Summary

```
STEP 1: User creates vault entry
    vault_entries row: id=5, is_deleted=false, deleted_at=null

STEP 2: User deletes entry (soft delete)
    is_deleted=true, deleted_at="2026-03-01T10:00:00"
    → Entry appears in trash with daysRemaining=30

STEP 3a: User restores entry (within 30 days)
    is_deleted=false, deleted_at=null
    → Back in active vault, nothing lost

STEP 3b: User permanently deletes (or 30 days pass)
    DELETE secure_shares WHERE vault_entry_id=5
    DELETE vault_snapshots WHERE vault_entry_id=5
    DELETE vault_entries WHERE id=5
    → Gone forever, all data wiped
```

---

## Database Tables Used

| Table | Operation | When |
| :--- | :--- | :--- |
| `vault_entries` | SELECT WHERE is_deleted=true | List trash, get count |
| `vault_entries` | UPDATE SET is_deleted=false | Restore single, restore all |
| `vault_entries` | DELETE | Permanent delete, empty trash, auto-cleanup |
| `secure_shares` | DELETE WHERE vault_entry_id=? | Before every permanent delete |
| `vault_snapshots` | DELETE WHERE vault_entry_id=? | Before every permanent delete |

---

## Dependencies Summary

| Dependency | Used For |
| :--- | :--- |
| `spring-data-jpa` | `VaultTrashRepository`, `VaultSnapshotRepository`, `SecureShareRepository` |
| `spring-cloud-openfeign` | `UserClient` — resolves username → userId for DB queries |
| `java.time.ChronoUnit` | `daysRemaining` calculation in `mapToTrashResponse()` |
| `spring-scheduling` | `TrashCleanupScheduler` automatic monthly cleanup |
| `slf4j` | All restore and delete operations logged at INFO level |

---

## Security Design Notes

| Decision | Why |
| :--- | :--- |
| All trash queries include `is_deleted = true` filter | Prevents restore/delete calls from working on ACTIVE (non-trashed) entries |
| Ownership check uses `findByIdAndUserIdAndIsDeletedTrue` | User must own the entry AND it must be in trash — prevents cross-user manipulation |
| Cascade delete: shares and snapshots first | Respects MySQL FK constraints; prevents constraint violation errors |
| `daysRemaining` computed fresh on every list call | Always accurate — no stale "days left" cached in the DB |
| 30-day auto-cleanup is server-side (not client-triggered) | Even if a user never opens the app, expired entries are cleaned up |

# Background & Automated Features — Complete Flow Document
> Part of the `vault-service` deep-dive series.

This document explains the tasks that run automatically on a schedule within the
`vault-service`. These features do not require HTTP requests from the frontend;
they trigger themselves via Spring's `@Scheduled` annotation.

---

## Technical Architecture for Scheduling

The application uses standard Spring scheduling driven by cron expressions.
All automated tasks scale simply in a single-instance environment. In a distributed
deployment (multiple `vault-service` pods running simultaneously), tasks like these
would either require ShedLock or a distributed job scheduler to avoid duplicate
execution — currently, it relies on idempotency/transaction isolation.

### Required Configuration
In the main application class, scheduling must be enabled:
```java
@EnableScheduling
@SpringBootApplication
public class VaultServiceApplication { ... }
```

---

## Files Involved

| File | Package | Role |
| :--- | :--- | :--- |
| `TrashCleanupScheduler.java` | `scheduler` | The cron-triggered beans for emptying old trash |
| `SecureShareCleanupScheduler.java` | `scheduler` | The cron-triggered beans for removing expired links |
| `VaultTrashService.java` | `service/vault` | Core logic for deleting trash + cascading deletes |
| `VaultTrashRepository.java` | `repository` | Custom JPQL to find entries past their retention date |
| `SecureShareRepository.java` | `repository` | Finds and deletes shares whose `expiresAt` is in the past |

---

## Feature 1 — Trash Auto-Cleanup

### What it does
When a user deletes a vault entry, it is moved to "Trash" (`is_deleted = true`).
To prevent the database from growing indefinitely, this scheduled job runs every night
at midnight to **permanently delete** any trash entry that has been sitting in the bin
for more than `TRASH_RETENTION_DAYS` (configured in service, default is usually 30 days).

Permanent deletion requires **cascading deletes** since vault entries have dependent records.

### ASCII Flow Diagram

```
[SYSTEM CLOCK]
    |
    | Reaches 00:00:00 (Midnight every day)
    | Cron: "0 0 0 * * ?"
    v
[TrashCleanupScheduler.java]
    |
    v
[VaultTrashService.cleanupExpired()]
    |
    |-- STEP 1: CALCULATE CUTOFF DATE
    |   LocalDateTime expiry = LocalDateTime.now().minusDays(30)
    |   (Any entry deleted before this date is considered "expired trash")
    |
    |-- STEP 2: FIND EXPIRED ENTRIES
    |   vaultTrashRepository.findExpiredTrashEntries(expiry)
    |       → MYSQL: SELECT * FROM vault_entries
    |                WHERE is_deleted = true AND deleted_at < ?
    |
    |-- IF expired list is empty → return (nothing to do)
    |
    |-- STEP 3: CASCADING DELETE (For each expired entry)
    |   For entry in expired:
    |       1. secureShareRepository.deleteByVaultEntryId(entry.getId())
    |          → MYSQL: DELETE FROM secure_shares WHERE vault_entry_id = ?
    |
    |       2. vaultSnapshotRepository.deleteByVaultEntryId(entry.getId())
    |          → MYSQL: DELETE FROM vault_snapshots WHERE vault_entry_id = ?
    |
    |-- STEP 4: DELETE THE ENTRIES THEMSELVES
    |   vaultTrashRepository.deleteAll(expired)
    |       → MYSQL: DELETE FROM vault_entries WHERE id IN (1, 5, 12, ...)
    |
    |-- STEP 5: LOGGING
    |   logger.info("Cleaned up {} expired trash entries", expired.size())
    v
[END OF TRANSACTION]
```

### Why Manual Cascading Deletes?
Notice that steps 3 and 4 perform manual deletes via the repository instead of
relying entirely on JPA `@OneToMany(cascade = CascadeType.REMOVE)`.
This is often done for **performance** in scheduled bulk operations, ensuring that
Hibernate doesn't have to load every single dependent row into memory just to
delete it. The repositories use direct `DELETE FROM` statements.

---

## Feature 2 — Expired Secure Share Cleanup

### What it does
When a user creates a secure share link, they set an expiry time (e.g., 1 hour, 1 day).
Once that time passes, the link naturally stops working because the controller checks
the `expiresAt` timestamp on read.

However, to keep the database table clean and remove stale cryptographic keys, this
scheduled job runs **every hour** to physically delete any `SecureShare` row that has
been expired for more than 24 hours.

### ASCII Flow Diagram

```
[SYSTEM CLOCK]
    |
    | Reaches XX:00:00 (Top of every hour)
    | Cron: "0 0 * * * ?"
    v
[SecureShareCleanupScheduler.java]
    |
    |-- @Transactional applied at method level
    |
    |-- STEP 1: CALCULATE CUTOFF DATETIME
    |   LocalDateTime cutoff = LocalDateTime.now().minusHours(24)
    |   (We give shares a 24-hour grace period after expiry before hard delete)
    |
    |-- STEP 2: FIND EXPIRED SHARES
    |   shareRepository.findByExpiresAtBeforeAndIsRevokedFalse(cutoff)
    |       → MYSQL: SELECT * FROM secure_shares
    |                WHERE expires_at < ? AND is_revoked = false
    |       (Revoked shares are optionally kept for audit, or deleted differently)
    |
    |-- IF expired list is empty → return (nothing to do)
    |
    |-- STEP 3: BULK DELETE
    |   shareRepository.deleteAll(expired)
    |       → MYSQL: DELETE FROM secure_shares WHERE id IN (14, 82, 91, ...)
    |       (The share_key and encrypted_password fields are permanently wiped)
    |
    |-- STEP 4: LOGGING
    |   logger.info("Cleaned up {} expired secure share(s) older than {}.", size, cutoff)
    v
[END OF TRANSACTION]
```

## Security Design Notes

| Security Benefit | Why it matters |
| :--- | :--- |
| **Hard delete of share rows** | Secures orphaned encryption data. An expired share contains an `encryptedPassword` and key material. Deleting the row removes all trace of the shared data from the `secure_shares` table. |
| **Transaction isolation** | Both schedulers use `@Transactional`. If a deletion fails midway (e.g. database connection drops), the entire batch rolls back, preventing orphaned dependent rows. |
| **Automated data minimization** | "Data you don't have can't be breached." Automatically purging soft-deleted trash and expired shares ensures the database surface area remains as small as possible. |

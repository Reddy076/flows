# Vault Timeline & Analytics — Complete Flow Document
> Part of the `vault-service` deep-dive series.

This document explains the full end-to-end flow of every Vault Timeline & Analytics feature,
from the frontend HTTP request all the way through to the database — including every
file involved, every dependency used, and ASCII diagrams for each flow.

---

## How Every Request Travels — FRONTEND → API Gateway → vault-service

```
FRONTEND (Angular :4200)
    │
    │  GET /api/timeline  (or /summary, /stats, /heatmap, /entry/{id})
    │  Authorization: Bearer eyJ...
    │
    ▼
API GATEWAY (:8080)
    ├── 1. RateLimitGlobalFilter → 100 req/min; 429 if over limit
    │
    ├── 2. JwtAuthFilter
    │       /api/timeline/** → PROTECTED route → JWT required
    │       Validates JWT signature + expiry
    │       Injects into forwarded request:
    │           X-User-Name: john    ← VaultTimelineController reads this directly
    │           X-Is-Duress: false
    │       If invalid → 401 Unauthorized
    │
    └── 3. Routes to vault-service (:8082)
    │
    ▼
VAULT-SERVICE (:8082)  ← arrives with X-User-Name, X-Is-Duress headers
    ├── CorsConfig              → validates Origin
    ├── GatewayAuthFilter       → reads X-User-Name → sets SecurityContext
    │       No JWT re-validation — trusts the gateway
    ├── SecurityConfig          → /api/timeline/** → authenticated()
    └── VaultTimelineController reads X-User-Name header directly
               (same pattern as BackupController)
```

> **Note on VaultTimelineController:** Like `BackupController`, this controller reads
> the username from the `X-User-Name` request header injected by the API Gateway,
> **not** from `SecurityContextHolder`. Both approaches are equivalent in security
> since the gateway only sets this header after validating the JWT.

---

## The Core Architecture — How Timeline Data is Sourced

```
IMPORTANT: Timeline data does NOT come from the vault DB directly.

All audit/event data lives in security-service.
VaultTimelineService calls security-service via Feign to get raw audit logs,
then enriches + aggregates them into timeline events locally.

┌─────────────────┐     Feign HTTP      ┌──────────────────┐
│  vault-service  │ ─────────────────►  │ security-service │
│ VaultTimeline   │   GET /audit/{user} │  audit_logs table│
│ Service.java    │ ◄────────────────── │  (all actions)   │
└─────────────────┘  List<AuditLogResponse>  └──────────────────┘
         │
         │  vault-service local DB (for entry ownership checks only)
         ▼
  VaultEntryRepository  → checks entry belongs to user
  VaultSnapshotRepository → counts password changes
  SecureShareRepository → counts share links per entry

Raw AuditLogResponse
    ──────────────────────────────────
    │ id        │ Long              │
    │ username  │ "john"            │
    │ action    │ "ENTRY_CREATED"   │
    │ details   │ "Created entry: Netflix" │
    │ ipAddress │ "192.168.1.100"   │
    │ timestamp │ 2026-03-20T10:00  │
    ──────────────────────────────────
         │
         ▼
    ActivityAggregator.aggregate()
         │  1. builds titleCache: Map<String,VaultEntry> (lowercase title → entry)
         │  2. TimelineEventEnricher.enrich() each raw log
         │     → resolves ActivityType from action string
         │     → extracts vaultEntryTitle from details string
         │     → looks up vaultEntryId from titleCache
         │     → assigns severity (CRITICAL/HIGH/MEDIUM/LOW)
         │     → builds human-readable description
         ▼
    List<TimelineEventDTO>  (final enriched events for API response)
```

---

## Files Involved

| File | Package | Role |
| :--- | :--- | :--- |
| `VaultTimelineController.java` | `controller` | HTTP entry point for all `/api/timeline/**` endpoints |
| `VaultTimelineService.java` | `service/analytics` | Core business logic — timeline, summary, entry timeline, stats |
| `AccessHeatmapService.java` | `service/analytics` | Heatmap: 30-day hourly + daily access grid |
| `ActivityAggregator.java` | `service/analytics` | Groups raw audit logs → `TimelineEventDTO` list |
| `TimelineEventEnricher.java` | `service/analytics` | Enriches raw logs with type, category, severity, description |
| `SecurityClient.java` | `client` | Feign — fetches all audit logs from `security-service` |
| `UserClient.java` | `client` | Feign — resolves username → user.id |
| `VaultEntryRepository.java` | `repository` | Ownership checks; builds titleCache |
| `VaultSnapshotRepository.java` | `repository` | Counts password changes per entry |
| `SecureShareRepository.java` | `repository` | Counts share links per entry |
| `GatewayAuthFilter.java` | `security` | Reads `X-User-Name` header → sets SecurityContext |
| `SecurityConfig.java` | `config` | `/api/timeline/**` → `authenticated()` |

### DTOs & Models

| Class | Role |
| :--- | :--- |
| `VaultTimelineResponse` | Full timeline — list of events + period + category breakdown |
| `TimelineSummaryResponse` | Stats: totals, most active day/hour, top 5 entries, 12-week histogram |
| `EntryTimelineResponse` | Single-entry timeline: events + password view count + share count |
| `TimelineStatsResponse` | Chart-ready: events by type/category, daily + monthly buckets, peak date |
| `HeatmapResponse` | 24-element hourly array + 7-element daily array + peak hour/day |
| `TimelineEventDTO` | Individual event: type, category, description, severity, timestamp |
| `AuditLogResponse` | Raw log from security-service before enrichment |
| `TimelineCategory` | Enum: `VAULT, AUTH, BREACH, SHARING, BACKUP, SECURITY` |
| `ActivityType` | Enum: maps audit action strings to typed enum values |

---

## Event Category Reference

| Category | Actions Included |
| :--- | :--- |
| `VAULT` | ENTRY_CREATED, ENTRY_UPDATED, ENTRY_DELETED, ENTRY_RESTORED, PASSWORD_VIEWED |
| `AUTH` | LOGIN, LOGOUT, LOGIN_FAILED, TOKEN_REFRESHED |
| `SHARING` | SHARE_CREATED, SHARE_ACCESSED, SHARE_REVOKED |
| `BREACH` | BREACH_DETECTED, BREACH_SCAN_RUN, BREACH_RESOLVED |
| `BACKUP` | VAULT_EXPORTED, VAULT_IMPORTED, SNAPSHOT_RESTORED |
| `SECURITY` | TIMELINE_VIEWED, DASHBOARD_VIEWED, SECURITY_ALERT_CREATED |

## Severity Reference

| Severity | Activity Types |
| :--- | :--- |
| `CRITICAL` | BREACH_DETECTED, LOGIN_FAILED |
| `HIGH` | ENTRY_DELETED, SHARE_CREATED, BREACH_SCAN_RUN |
| `MEDIUM` | ENTRY_UPDATED, ENTRY_RESTORED, SHARE_ACCESSED, SHARE_REVOKED, VAULT_EXPORTED, BREACH_RESOLVED |
| `LOW` | Everything else (creates, logins, views) |

---

## Feature 1 — Get Full Activity Timeline

### What it does
Returns the user's complete chronological activity feed — every vault action, login,
share, breach scan, backup, etc. Supports optional filtering by time range (`?days=N`)
and event category (`?category=VAULT`).

### ASCII Flow Diagram

```
FRONTEND
    |
    | GET /api/timeline?days=30&category=VAULT
    | Authorization: Bearer <token>
    |
    v
[API GATEWAY (:8080)]
    |-- RateLimitGlobalFilter: rate check
    |-- JwtAuthFilter: /api/timeline → PROTECTED → validates JWT
    |       Injects: X-User-Name: john, X-Is-Duress: false
    v
VAULT-SERVICE (:8082)
    |-- CorsConfig: valid Origin
    |-- GatewayAuthFilter: reads X-User-Name → sets SecurityContext
    |-- SecurityConfig: /api/timeline/** → authenticated() ✓
    v
[VaultTimelineController.getTimeline(days=30, category="VAULT", username="john")]
    | username read from X-User-Name header (set by gateway)
    v
[VaultTimelineService.getTimeline("john", 30, "VAULT")]
    |
    |-- VALIDATE CATEGORY FILTER
    |   VALID_CATEGORIES = {VAULT, AUTH, BREACH, SHARING, BACKUP, SECURITY}
    |   "VAULT" ∈ valid set → OK
    |   Unknown category → throw IllegalArgumentException
    |
    |-- STEP 1: GET USER ID
    |   userClient.getUserDetailsByUsername("john")
    |       → Feign: GET http://user-service/api/users/internal/by-username/john
    |       → returns { id: 1, ... }
    |
    |-- STEP 2: FETCH AUDIT LOGS FROM SECURITY-SERVICE
    |   securityClient.getAuditLogs("john")
    |       → Feign: GET http://security-service/api/logs/john
    |       → returns List<AuditLogResponse> (ALL time, unfiltered)
    |   fetchLogs() applies days filter in memory:
    |       since = now().minusDays(30) → 2026-02-22T01:19
    |       filter: log.timestamp.isAfter(since)
    |       → Returns only last 30 days of logs
    |
    |-- STEP 3: LOG THIS ACCESS
    |   securityClient.logAction("john", "TIMELINE_VIEWED", "Viewed vault timeline")
    |       → Feign: POST http://security-service/api/logs
    |       (the act of viewing the timeline is itself logged)
    |
    |-- STEP 4: AGGREGATE LOGS INTO EVENTS
    |   activityAggregator.aggregate(logs, userId=1)
    |       → builds titleCache: Map<lowercase title → VaultEntry>
    |          MYSQL SELECT * FROM vault_entries WHERE user_id = 1
    |       → TimelineEventEnricher.enrich() each raw log:
    |               resolveActivityType("ENTRY_CREATED")   → ActivityType.ENTRY_CREATED
    |               extractEntryTitle("Created entry: Netflix") → "Netflix"
    |               titleCache.get("netflix")              → VaultEntry { id:5, url:... }
    |               buildDescription(log, action)          → "Created entry: Netflix"
    |               resolveSeverity(ENTRY_CREATED)         → "LOW"
    |       → Returns List<TimelineEventDTO>
    |
    |-- STEP 5: APPLY CATEGORY FILTER (in memory)
    |   events.stream().filter(e -> "VAULT".equalsIgnoreCase(e.getCategory()))
    |   → Keeps only VAULT events, drops AUTH/BREACH/etc.
    |
    |-- STEP 6: COMPUTE CATEGORY BREAKDOWN
    |   computeCategoryBreakdown(events):
    |       counts events per category
    |       → { vaultEvents:12, authEvents:0, sharingEvents:0, ... }
    |
    |-- STEP 7: COMPUTE PERIOD
    |   TimelinePeriod.resolve(days=30, earliest=...)
    |       → { label:"LAST_30_DAYS", start:"2026-02-22", end:"2026-03-24" }
    |
    v
[RESPONSE 200 OK]
    | Body: VaultTimelineResponse {
    |   totalEvents: 12,
    |   period: "LAST_30_DAYS",
    |   startDate: "2026-02-22",
    |   endDate: "2026-03-24",
    |   categoryBreakdown: {
    |     vaultEvents: 12, authEvents: 0, sharingEvents: 0,
    |     securityEvents: 0, breachEvents: 0, backupEvents: 0
    |   },
    |   events: [
    |     {
    |       id: 101,
    |       eventType: "ENTRY_CREATED",
    |       category: "VAULT",
    |       description: "Created entry: Netflix",
    |       vaultEntryId: 5,
    |       vaultEntryTitle: "Netflix",
    |       websiteUrl: "netflix.com",
    |       ipAddress: "192.168.1.100",
    |       timestamp: "2026-03-20T10:00:00",
    |       severity: "LOW"
    |     },
    |     ...
    |   ]
    | }
```

### Query Parameter Options

| Parameter | Example | Behaviour |
| :--- | :--- | :--- |
| *(none)* | `GET /api/timeline` | Returns ALL-TIME events, all categories |
| `?days=7` | Last 7 days | Client-side time filter applied in memory after Feign fetch |
| `?category=VAULT` | Only vault CRUD events | Category filter applied after aggregation |
| `?days=30&category=BREACH` | Last 30 days, breach only | Both filters applied together |

---

## Feature 2 — Get Timeline Summary Stats

### What it does
Returns high-level usage statistics for the user's entire vault history:
total creates/deletes/password changes/shares/breach detections, the most
active day of week, most active hour, top 5 most-accessed entries,
and a 12-week rolling activity histogram.

### ASCII Flow Diagram

```
FRONTEND
    |
    | GET /api/timeline/summary
    | Authorization: Bearer <token>
    |
    v
[API GATEWAY] → validates JWT → X-User-Name: john → vault-service
    |
    v
[GatewayAuthFilter] → reads X-User-Name → sets SecurityContext
    |
    v
[VaultTimelineController.getSummary(username="john")]
    v
[VaultTimelineService.getSummary("john")]
    |
    |-- STEP 1: GET USER ID
    |   userClient.getUserDetailsByUsername("john") → userId = 1
    |
    |-- STEP 2: LOG ACCESS
    |   securityClient.logAction("john", "TIMELINE_VIEWED", "Viewed timeline summary")
    |
    |-- STEP 3: FETCH ALL AUDIT LOGS (ALL TIME, no day filter)
    |   securityClient.getAuditLogs("john") → List<AuditLogResponse> (all time)
    |
    |-- STEP 4: FETCH ALL VAULT SNAPSHOTS
    |   vaultSnapshotRepository.findByVaultEntryUserIdOrderByChangedAtDesc(1)
    |       → MYSQL SELECT * FROM vault_snapshots
    |                WHERE vault_entry_user_id = 1 ORDER BY changed_at DESC
    |
    |-- STEP 5: COMPUTE TOTALS (by scanning all logs)
    |   countByAction(logs, "ENTRY_CREATED")   → totalCreated = 15
    |   countByAction(logs, "ENTRY_DELETED")   → totalDeleted = 3
    |   snapshots.size()                        → totalPasswordChanges = 7
    |   countByAction(logs, "SHARE_CREATED")   → totalSharesCreated = 4
    |   countByAction(logs, "BREACH_DETECTED") → totalBreachDetections = 1
    |   allLogs.size()                         → totalAuditEvents = 87
    |
    |-- STEP 6: MOST ACTIVE DAY OF WEEK
    |   computeMostActiveDayOfWeek(allLogs)
    |       → int[7] counts — increment for each log's DayOfWeek
    |       → find max index → DayOfWeek.of(maxIdx).getDisplayName()
    |       → e.g. "Wednesday"
    |
    |-- STEP 7: MOST ACTIVE HOUR
    |   computeMostActiveHour(allLogs)
    |       → int[24] counts — increment for each log's getHour()
    |       → find max index
    |       → e.g. 14  (2pm)
    |
    |-- STEP 8: TOP 5 MOST ACCESSED ENTRIES
    |   computeMostAccessedEntries(allLogs, userId=1)
    |       → filter logs for PASSWORD_VIEWED and ENTRY_UPDATED actions
    |       → extract entry title from details string
    |       → Map<title, accessCount> → sort descending → top 5
    |       → enrich with vaultEntryId and websiteUrl from titleCache
    |       → e.g. [
    |               { entryId:5, title:"Netflix", url:"netflix.com", accessCount:8 },
    |               { entryId:3, title:"Gmail",   url:"gmail.com",   accessCount:5 },
    |               ...
    |            ]
    |
    |-- STEP 9: 12-WEEK ACTIVITY HISTOGRAM
    |   computeWeeklyActivity(allLogs)
    |       → Creates 12 week-start buckets (Mon) from today-11weeks to today
    |       → For each log in last 12 weeks: increment its week bucket
    |       → e.g. [
    |               { weekStart:"2026-01-05", eventCount:12 },
    |               { weekStart:"2026-01-12", eventCount:8 },
    |               ...
    |            ]
    |
    v
[RESPONSE 200 OK]
    | Body: TimelineSummaryResponse {
    |   totalEntriesCreated: 15,
    |   totalPasswordChanges: 7,
    |   totalEntriesDeleted: 3,
    |   totalSharesCreated: 4,
    |   totalBreachDetections: 1,
    |   totalAuditEvents: 87,
    |   mostActiveDayOfWeek: "Wednesday",
    |   mostActiveHour: 14,
    |   mostAccessedEntries: [ { entryId, title, url, accessCount } x5 ],
    |   weeklyActivity: [ { weekStart, eventCount } x12 ]
    | }
```

---

## Feature 3 — Get Timeline for a Single Entry

### What it does
Returns the full lifecycle of one specific vault entry — every action ever taken
on it: creation, updates, password views, shares, breach detections, and restores.
Also returns counts for password changes, password views, and share links created.
The entry must belong to the requesting user (ownership enforced).

### ASCII Flow Diagram

```
FRONTEND
    | (user clicks on a specific vault entry to see its full history)
    |
    | GET /api/timeline/entry/{entryId}
    | Authorization: Bearer <token>
    | e.g. GET /api/timeline/entry/5
    |
    v
[API GATEWAY] → validates JWT → X-User-Name: john → vault-service
    |
    v
[GatewayAuthFilter] → reads X-User-Name → sets SecurityContext
    |
    v
[VaultTimelineController.getEntryTimeline(entryId=5, username="john")]
    v
[VaultTimelineService.getEntryTimeline("john", 5)]
    |
    |-- STEP 1: GET USER ID
    |   userClient.getUserDetailsByUsername("john") → userId = 1
    |
    |-- STEP 2: LOG ACCESS
    |   securityClient.logAction("john", "TIMELINE_VIEWED", "Viewed timeline for entry id=5")
    |
    |-- STEP 3: OWNERSHIP CHECK
    |   vaultEntryRepository.findByIdAndUserId(entryId=5, userId=1)
    |       → MYSQL SELECT * FROM vault_entries WHERE id = 5 AND user_id = 1
    |   If not found → throw ResourceNotFoundException("Vault entry not found: 5")
    |       (covers both "doesn't exist" and "belongs to another user")
    |
    |-- STEP 4: FETCH ALL USER'S AUDIT LOGS
    |   securityClient.getAuditLogs("john") → all audit logs
    |
    |-- STEP 5: FILTER LOGS FOR THIS ENTRY
    |   isLogForEntry(log, entryId=5, entryTitle="Netflix")
    |       → only keeps actions: ENTRY_CREATED/UPDATED/DELETED/RESTORED,
    |                             PASSWORD_VIEWED, SHARE_CREATED/REVOKED,
    |                             BREACH_DETECTED/RESOLVED
    |       → extractEntryTitle(log.details, action) == "Netflix" (case-insensitive)
    |   Result: List<AuditLogResponse> (only those for "Netflix" entry)
    |
    |-- STEP 6: AGGREGATE INTO EVENTS
    |   activityAggregator.aggregate(entryLogs, userId)
    |       → enriches each log → List<TimelineEventDTO>
    |
    |-- STEP 7: COUNT PASSWORD CHANGES (from vault_snapshots)
    |   vaultSnapshotRepository.findByVaultEntryIdOrderByChangedAtDesc(5)
    |       → MYSQL SELECT * FROM vault_snapshots WHERE vault_entry_id = 5
    |       → passwordChanges = snapshots.size() = 3
    |
    |-- STEP 8: COUNT PASSWORD VIEWS (from audit logs)
    |   countByAction(entryLogs, "PASSWORD_VIEWED") → passwordViews = 8
    |
    |-- STEP 9: COUNT SHARES FOR THIS ENTRY
    |   secureShareRepository.findByOwnerIdOrderByCreatedAtDesc(userId=1)
    |       → MYSQL SELECT * FROM secure_shares WHERE owner_id = 1
    |       → filter: s.getVaultEntry().getId() == 5
    |       → shareCount = 2
    |
    v
[RESPONSE 200 OK]
    | Body: EntryTimelineResponse {
    |   entryId: 5,
    |   entryTitle: "Netflix",
    |   websiteUrl: "netflix.com",
    |   deleted: false,
    |   passwordChangeCount: 3,
    |   passwordViewCount: 8,
    |   shareCount: 2,
    |   events: [
    |     { eventType:"ENTRY_CREATED",    category:"VAULT",   severity:"LOW",      timestamp:"2024-01-10T09:00" },
    |     { eventType:"PASSWORD_VIEWED",  category:"VAULT",   severity:"LOW",      timestamp:"2026-03-01T14:22" },
    |     { eventType:"SHARE_CREATED",    category:"SHARING", severity:"HIGH",     timestamp:"2026-03-10T16:05" },
    |     { eventType:"BREACH_DETECTED",  category:"BREACH",  severity:"CRITICAL", timestamp:"2026-03-15T08:00" },
    |     ...
    |   ]
    | }
```

---

## Feature 4 — Get Chart-Ready Stats

### What it does
Returns aggregated statistics ready for charting libraries (Chart.js, Recharts, etc.):
- Events grouped by type (e.g. `ENTRY_CREATED: 15, SHARE_CREATED: 4`)
- Events grouped by category (e.g. `VAULT: 40, AUTH: 20`)
- Daily activity buckets (per-category counts per day)
- Monthly activity buckets
- Peak activity date and count
- Average events per day

Supports `?days=N` for time range filtering.

### ASCII Flow Diagram

```
FRONTEND
    | (dashboard chart rendering)
    |
    | GET /api/timeline/stats?days=30
    | Authorization: Bearer <token>
    |
    v
[API GATEWAY] → validates JWT → X-User-Name: john → vault-service
    |
    v
[GatewayAuthFilter] → reads X-User-Name → sets SecurityContext
    |
    v
[VaultTimelineController.getStats(days=30, username="john")]
    v
[VaultTimelineService.getStats("john", 30)]
    |
    |-- STEP 1: GET USER ID + LOG ACCESS
    |   userClient.getUserDetailsByUsername("john") → userId = 1
    |   securityClient.logAction("john", "TIMELINE_VIEWED", "Viewed timeline stats")
    |
    |-- STEP 2: FETCH + FILTER LOGS
    |   fetchLogs("john", days=30)
    |       → securityClient.getAuditLogs("john")
    |       → filter: timestamp.isAfter(now - 30 days)
    |
    |-- STEP 3: AGGREGATE INTO EVENTS
    |   activityAggregator.aggregate(logs, userId)
    |       → List<TimelineEventDTO>
    |
    |-- STEP 4: GROUP BY TYPE AND CATEGORY
    |   Map<eventType, count>:
    |       { "ENTRY_CREATED":12, "PASSWORD_VIEWED":8, "SHARE_CREATED":3, ... }
    |   Map<category, count>:
    |       { "VAULT":25, "AUTH":10, "SHARING":5, "BREACH":2, "BACKUP":1, "SECURITY":4 }
    |
    |-- STEP 5: BUILD DAILY BUCKETS (per-day, per-category breakdown)
    |   buildDailyBuckets(events):
    |       TreeMap<LocalDate, DailyActivityBucket> (sorted by date)
    |       For each event: increment total count + category-specific count
    |       e.g. {
    |           date: "2026-03-20",
    |           count: 5,
    |           vaultCount: 3, sharingCount: 1, authCount: 1,
    |           securityCount: 0, breachCount: 0, backupCount: 0
    |       }
    |
    |-- STEP 6: BUILD MONTHLY BUCKETS
    |   buildMonthlyBuckets(events):
    |       TreeMap<"yyyy-MM", count>
    |       e.g. { "2026-02": 18, "2026-03": 29 }
    |
    |-- STEP 7: COMPUTE AVERAGES AND PEAK
    |   computeAvgPerDay(events, days=30) → total / 30 = e.g. 1.57
    |   findPeakDate(dailyBuckets) → date with highest total count → "2026-03-20"
    |   peakCount = 5
    |
    v
[RESPONSE 200 OK]
    | Body: TimelineStatsResponse {
    |   eventsByType:      { "ENTRY_CREATED":12, "PASSWORD_VIEWED":8, ... },
    |   eventsByCategory:  { "VAULT":25, "AUTH":10, "SHARING":5, ... },
    |   totalEvents: 47,
    |   averageEventsPerDay: 1.57,
    |   peakActivityDate: "2026-03-20",
    |   peakActivityCount: 5,
    |   dailyActivity: [
    |     { date:"2026-02-22", count:2, vaultCount:2, ... },
    |     { date:"2026-02-23", count:0, ... },
    |     ...
    |     { date:"2026-03-24", count:3, vaultCount:2, sharingCount:1, ... }
    |   ],
    |   monthlyActivity: [
    |     { month:"2026-02", count:18 },
    |     { month:"2026-03", count:29 }
    |   ]
    | }
```

---

## Feature 5 — Get 30-Day Access Heatmap

### What it does
Returns two arrays suitable for rendering a heatmap chart:
- **24-element hourly array** — how many vault events happened at each hour (0–23)
- **7-element daily array** — how many vault events happened on each day of week (Sun–Sat)

Plus the peak hour (0-23) and peak day name (e.g. "Wednesday").
Always covers exactly the **last 30 days** — no filter parameter.

### ASCII Flow Diagram

```
FRONTEND
    | (renders "When am I most active?" heatmap chart)
    |
    | GET /api/timeline/heatmap
    | Authorization: Bearer <token>
    |
    v
[API GATEWAY] → validates JWT → X-User-Name: john → vault-service
    |
    v
[GatewayAuthFilter] → reads X-User-Name → sets SecurityContext
    |
    v
[VaultTimelineController.getHeatmap(username="john")]
    v
[AccessHeatmapService.getAccessHeatmap("john")]
    |
    |-- STEP 1: FETCH ALL AUDIT LOGS
    |   securityClient.getAuditLogs("john")
    |       → Feign: GET http://security-service/api/logs/john
    |       → All-time logs returned
    |
    |-- STEP 2: FILTER TO LAST 30 DAYS (in memory)
    |   thirtyDaysAgo = LocalDateTime.now().minusDays(30)
    |   filter: log.timestamp.isAfter(thirtyDaysAgo)
    |   → e.g. 87 total logs → 42 logs in last 30 days
    |
    |-- STEP 3: BUILD HOURLY COUNTS
    |   hourlyCounts = int[24]  (index 0 = midnight, 23 = 11pm)
    |   For each log: hourlyCounts[log.timestamp.getHour()]++
    |   e.g. hourlyCounts = [0,0,0,0,0,0,1,3,5,6,4,3,2,8,5,4,3,2,1,0,0,0,0,0]
    |                        ↑midnight                 ↑2pm peak               ↑11pm
    |
    |-- STEP 4: BUILD DAILY COUNTS
    |   dailyCounts = int[7]  (index 0 = Sunday, 1 = Monday, ..., 6 = Saturday)
    |   dayIndex = (log.timestamp.getDayOfWeek().getValue() % 7)
    |       → Java DayOfWeek: MON=1..SUN=7; % 7 maps SUN→0, MON→1..SAT→6
    |   e.g. dailyCounts = [3, 8, 12, 9, 7, 5, 2]
    |                        ↑Sun  ↑Wed peak          ↑Sat
    |
    |-- STEP 5: FIND PEAK HOUR
    |   Scan hourlyCounts → max value at index 13 → peakHour = 13 (1pm)
    |   peakHourCount = 8
    |
    |-- STEP 6: FIND PEAK DAY
    |   Scan dailyCounts → max value at index 2 → javaDayValue = 2 (TUESDAY)
    |   DayOfWeek.of(2).getDisplayName(FULL, Locale) → "Tuesday"
    |
    v
[RESPONSE 200 OK]
    | Body: HeatmapResponse {
    |   accessByHour: [0,0,0,0,0,0,1,3,5,6,4,3,2,8,5,4,3,2,1,0,0,0,0,0],
    |                  ← 24 integers, one per hour (0=midnight to 23=11pm)
    |   accessByDay:  [3, 8, 12, 9, 7, 5, 2],
    |                  ← 7 integers: [Sun, Mon, Tue, Wed, Thu, Fri, Sat]
    |   peakHour: 13,
    |   peakDay: "Tuesday",
    |   totalAccesses: 42,
    |   period: "LAST_30_DAYS"
    | }

FRONTEND (chart rendering):
    accessByHour → renders as a 24-column bar chart or heat row
    accessByDay  → renders as a 7-column bar chart or heat row
    Both together → 24×7 grid heatmap (GitHub-contributions style)
```

---

## How TimelineEventEnricher Works

The enricher is the key component that transforms raw, opaque audit log strings
into structured, human-readable timeline events:

```
RAW INPUT (AuditLogResponse from security-service):
    action:  "ENTRY_CREATED"
    details: "Created entry: Netflix"
    ip:      "192.168.1.100"
    ts:      2026-03-20T10:00

STEP 1 — resolveActivityType("ENTRY_CREATED")
    ActivityType.fromAuditAction("ENTRY_CREATED") → ActivityType.ENTRY_CREATED
    category: ActrivityType.ENTRY_CREATED → TimelineCategory.VAULT

STEP 2 — extractEntryTitle("Created entry: Netflix", "ENTRY_CREATED")
    switch("ENTRY_CREATED") → extractAfterPrefix(details, "Created entry: ")
    "Created entry: Netflix".substring("Created entry: ".length()) → "Netflix"

STEP 3 — titleCache lookup
    titleCache.get("netflix") → VaultEntry { id:5, websiteUrl:"netflix.com" }
    vaultEntryId = 5, websiteUrl = "netflix.com"

STEP 4 — buildDescription(log, "ENTRY_CREATED")
    No special case → return details = "Created entry: Netflix"

STEP 5 — resolveSeverity(ActivityType.ENTRY_CREATED)
    Not in CRITICAL/HIGH/MEDIUM sets → "LOW"

OUTPUT (TimelineEventDTO):
    eventType:      "ENTRY_CREATED"
    category:       "VAULT"
    description:    "Created entry: Netflix"
    vaultEntryId:   5
    vaultEntryTitle:"Netflix"
    websiteUrl:     "netflix.com"
    ipAddress:      "192.168.1.100"
    timestamp:      2026-03-20T10:00
    severity:       "LOW"
```

---

## Database Tables Used

| Table | Operation | When |
| :--- | :--- | :--- |
| `vault_entries` | SELECT WHERE user_id = ? | Build titleCache for all timeline features |
| `vault_entries` | SELECT WHERE id = ? AND user_id = ? | Ownership check in entry timeline |
| `vault_snapshots` | SELECT WHERE vault_entry_id = ? | Count password changes in entry timeline |
| `vault_snapshots` | SELECT WHERE vault_entry_user_id = ? | Count total password changes in summary |
| `secure_shares` | SELECT WHERE owner_id = ? | Count share links in entry timeline |

> **Note:** The main event/audit data does NOT come from vault DB tables.
> All audit logs are fetched from `security-service` via the `SecurityClient` Feign client.

---

## External Service Calls (Feign)

| Feign Client | Endpoint Called | When |
| :--- | :--- | :--- |
| `SecurityClient.getAuditLogs(username)` | `GET /api/logs/{username}` | Every timeline feature — fetches raw event data |
| `SecurityClient.logAction(username, action, details)` | `POST /api/logs` | Every feature — logs "TIMELINE_VIEWED" event |
| `UserClient.getUserDetailsByUsername(username)` | `GET /api/users/internal/by-username/{username}` | Every feature — resolves username → userId |

---

## Dependencies Summary

| Dependency | Used For |
| :--- | :--- |
| `spring-data-jpa` | `VaultEntryRepository`, `VaultSnapshotRepository`, `SecureShareRepository` |
| `spring-cloud-openfeign` | `SecurityClient` (audit logs), `UserClient` (user ID lookup) |
| `java.time` | `LocalDateTime`, `LocalDate`, `DayOfWeek`, `ChronoUnit` — all time calculations |
| `java.util.stream` | filtering, grouping, sorting events in memory |
| `Lombok` | `@Builder`, `@RequiredArgsConstructor`, `@Slf4j` |

---

## Security Design Notes

| Decision | Why |
| :--- | :--- |
| Username read from `X-User-Name` header | Same as BackupController — trusts the gateway |
| Ownership check via `findByIdAndUserId` | Entry timeline: user can only see their own entries |
| Category filter validated against whitelist | Prevents injection of arbitrary filter strings |
| `@Transactional(readOnly = true)` on all methods | No writes happen — hint to DB to use read replicas |
| Viewing timeline itself is logged | Full audit trail — even analytics access is recorded |
| Audit data sourced from security-service | Single source of truth for all audit events across services |

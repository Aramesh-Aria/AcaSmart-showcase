# AcaSmart — Technical Decisions (ADR Summaries)

> Architecture Decision Records 0001–0013 (14 records — two share number 0001), condensed for quick review. Full ADRs live in `docs/adr/` of the private repository.

---

## ADR-0001: Application-Level License System with Offline Activation

**Decision:** Ed25519-signed license keys (`AMG-XXXX-XXXX-XXXX`), device fingerprint binding, 7-day trial, embedded public key, vendor-side key generation.

| Why | Trade-off Accepted |
|-----|-------------------|
| Installer-based keys were Windows-only, visible in script, required rebuild per key | Added `cryptography` dependency (C extensions); device fingerprinting uses `subprocess` (wmic on Win7) |

**Key details:** Cross-platform (Win 7+, macOS ARM64/x86), offline validation after activation, key renewal without app rebuild, survives backup/restore via fingerprint match.

---

## ADR-0001 (domain): One-to-One Lessons First

**Decision:** Every enrollment schedule, session, attendance record, tuition cycle, payment and renewal reminder belongs to **one Student in one Class**. Group classes are out of scope.

| Why | Trade-off Accepted |
|-----|-------------------|
| The academy teaches one-to-one; group support changes session identity, attendance, conflicts, billing, reporting and notifications at once | A future group-class feature is a new model, not a flag |

---

## ADR-0002: Enrollment Schedule Is Source of Truth (Model-B)

**Decision:** No `sessions` table. Lessons computed weekly from `student_terms.start_date` + `class.day`. Attendance is the only record of what happened.

| Why | Trade-off Accepted |
|-----|-------------------|
| Pre-creating sessions made cancellation, rescheduling, completion, history harder to reason about | Every "list lessons" query is now a computation over `student_terms` + `attendance`; migration v6 dropped legacy table |

**Impact:** Conflict detection now uses term schedules (interval overlap on `start_time`/`lesson_duration`) — strictly better, reflects real weekly commitment.

---

## ADR-0003: Term Pricing Is Snapshotted at Enrollment

**Decision:** `tuition_fee`, `sessions_limit`, `currency_unit` copied to `student_terms` at creation. Price list changes affect only future terms.

| Why | Trade-off Accepted |
|-----|-------------------|
| Changing academy price list must not silently rewrite what an already-enrolled student owes | Tuition and session limit duplicated onto every `student_terms` row instead of live lookup |

---

## ADR-0004: Attendance Stays Date-Keyed (Not Session-Keyed)

**Decision:** `attendance` PK = `(term_id, date)` — not a session FK. One row per term per date.

| Why | Trade-off Accepted |
|-----|-------------------|
| A lesson is one per day per term; reschedule moves date, not session identity | Same-day attendance on merged terms collapses to one record (handled in migration v5) |

---

## ADR-0005: Term Completion Is Derived, Not a Frozen Lock

**Decision:** `end_date` set when `present + absent >= sessions_limit`; cleared when attendance deleted. Completed terms stay editable.

| Why | Trade-off Accepted |
|-----|-------------------|
| Attendance gets corrected after the fact; a completed term must re-open when a record is deleted | Completion recomputed after every attendance change rather than one-way state transition |

**Critical fix:** Prior design hid completed terms and wrote one-shot "SMS sent" flag even on failure — transient SMS failure froze term forever. Derived completion removes the locked state entirely.

---

## ADR-0006: Three-State Attendance; Reschedule = Cancellation + New Attendance

**Decision:** Status ∈ `{present, absent, canceled}`. Present+Absent consume a session. Canceled consumes zero. Reschedule = Canceled (reason="rescheduled to...") + Attendance on new date.

| Why | Trade-off Accepted |
|-----|-------------------|
| Canceled sessions must not burn a paid session; moved lesson is cancel old + attend new | Every counting query must filter `AND status != 'canceled'` |

**Fairness bug fixed:** Boolean `is_present` model charged students for canceled lessons.

---

## ADR-0007: Lightweight Guardrails, Not an Audit Ledger

**Decision:** Two rules: (1) Term deletion blocked if any payment/attendance exists. (2) Editing historical term requires confirmation warning. No "Adjustment" entity / audit trail.

| Why | Trade-off Accepted |
|-----|-------------------|
| Single non-technical operator, single machine — deletion blocked, corrections are deliberate edits | No full change history / undo; trades auditability for simpler schema and UI |

**Future:** If multi-operator, introduce real adjustment records.

---

## ADR-0008: Versioned, Backed-Up Migrations Before Big Refactor

**Decision:** `PRAGMA user_version` tracking; WAL-safe backup before each migration; transactional steps; restore + abort on failure.

| Why | Trade-off Accepted |
|-----|-------------------|
| DB lives on user's machine with real data; bad migration on launch would lose it | Startup takes backup + can abort; migration functions must be idempotent |

**Strategy:** Separate cheap hardening (v2–v5) from risky structural change (v6 Model-B).

---

## ADR-0009: Duplicate Active Terms (1-hr Lesson Splits) Merged by Summing

**Decision:** Pre-Model-B, 1-hour lessons were split into two 30-min terms. Migration v5 merges: keep survivor's session limit (6), **sum fees & payments** (4.2M + 4.2M → 8.4M), repoint attendance, delete duplicate, add unique index.

| Why | Trade-off Accepted |
|-----|-------------------|
| Academy pricing: 8.4M per 12 meetings — 1-hour = two paid 30-min halves | One-time irreversible data migration; runs after full backup (ADR-0008) |

**Why summing not dropping:** Custom pricing profiles would lose correct total if one half dropped.

---

## ADR-0010: Sidebar Shell Replaces Multi-Window

**Decision:** One `ShellWindow` — sidebar (`QListWidget`) + `QStackedWidget`. Every module is a Page built once and never torn down; the shell calls `load_data` / `refresh_data` on activation. One `navigate_to()` hub serves the sidebar, the Ctrl+K command palette, notification cards and in-page links.

| Why | Trade-off Accepted |
|-----|-------------------|
| A launcher grid of separate windows with a `return_target` back button fragmented navigation and made "go to the right page for this card" impossible | One-shot refactor of every manager window into a page; Reports collapsed into one page with an internal sub-nav |

**Guardrail:** `test_retired_patterns.py` fails if `BaseSecondaryWindow` / `return_target` reappear. RTL is applied app-wide, so the sidebar sits on the right.

---

## ADR-0011: SMS Sending Off the UI Thread

**Decision:** Both provider calls — the automatic renewal reminder and the manual bulk invitation — run on `SmsSendWorker` (`QThread` + signals). Everything else stays synchronous (local SQLite is cheap).

| Why | Trade-off Accepted |
|-----|-------------------|
| The SMS provider is the only network dependency; a slow or hanging call froze the whole UI, and a frozen UI cannot observe the exact provider status the domain rule depends on | Thread lifetime must be managed (workers are parented to their page; tests join them before teardown) |

**Rule enforced:** a Renewal Reminder counts as sent only on an explicit success reported back by the worker.

---

## ADR-0012: Append-Only SMS Send Log Alongside the Latest-Outcome Ledger

**Decision:** Keep `sms_notifications` as a **latest-outcome ledger** (one row per student-term, overwritten — answers *"may we resend?"*) and add an **append-only `sms_send_log`** (migration v8 — answers *"what happened, and when?"*). Both reminder and manual sends write one log row per attempt outcome.

| Why | Trade-off Accepted |
|-----|-------------------|
| The ledger cannot hold attempt history (`UNIQUE(student_id, term_id)`, overwrite semantics) and manual SMS persisted nothing | Two SMS tables coexist on purpose; v8 backfills one log row per ledger row so the report has history from day one |

**Not stored:** the rendered SMS body (the provider renders it from template parameters) and `DISABLED` attempts.

---

## ADR-0013: Academy Today Uses a Fixed UTC+3:30 Offset

**Decision:** The default "today" (Academy Today) is the computer's UTC clock plus a hard-coded +3:30 — not the computer's timezone setting and not `zoneinfo`/tz data. The date picker became Jalali-native at the same time: its own month grid, Shamsi strings in and out.

| Why | Trade-off Accepted |
|-----|-------------------|
| Iran abolished DST in 2022, so a fixed offset *is* `Asia/Tehran`; un-updated Windows 7 machines (the production target) still apply the old DST rule and shifted "today" by an hour around midnight; `zoneinfo` doesn't exist on Python 3.8 | The constant must change if Iran changes its offset; a machine whose wall clock was hand-corrected for stale DST has a wrong UTC and inherits that error near midnight |

**Unchanged on purpose:** Record Timestamps (`created_at`, `sent_at`, …) stay on the computer's clock via SQLite `datetime('now','localtime')` — moving them would mean table rebuilds for the column defaults. License and trial dates stay Gregorian.

---

## Decision Summary Table

| ADR | Decision | Reversible? | Blast Radius |
|-----|----------|-------------|--------------|
| 0001 | Ed25519 license system | Yes (key rotation) | Medium — new dep, new tables |
| 0001 (domain) | One-to-one lessons first | Yes (scope) | High — shapes every table |
| 0002 | Schedule-as-truth (drop `sessions`) | **No** (data migration) | **High** — core domain rewrite |
| 0003 | Snapshotted pricing | Yes (add column) | Low — additive |
| 0004 | Date-keyed attendance | No (schema) | Medium — migration needed |
| 0005 | Derived completion | Yes (logic only) | Medium — behavior change |
| 0006 | Three-state attendance | No (schema) | Medium — migration v2 |
| 0007 | Guardrails over audit | Yes (add later) | Low — policy only |
| 0008 | Backed-up migrations | Yes (process) | Low — infrastructure |
| 0009 | Merge 1-hr splits | **No** (data) | High — one-time migration |
| 0010 | Sidebar shell | Yes (UI only) | High — every window became a page |
| 0011 | SMS off the UI thread | Yes | Medium — threading in two pages |
| 0012 | Append-only send log | Yes (additive) | Low — new table + report |
| 0013 | Academy Today = UTC+3:30 | Yes (one constant) | Low — every "today" goes through one function |

---

## Decisions NOT Made (Explicitly Deferred)

| Topic | Status | Trigger to Revisit |
|-------|--------|-------------------|
| Multi-user / multi-branch | Deferred | Second academy location |
| Audit ledger / adjustments | Deferred | Multiple operators |
| Online license revocation | Placeholder | License abuse detected |
| Floating/concurrent licenses | Deferred | Enterprise demand |
| Plugin/extension system | Deferred | Third-party integrations |
| Web/mobile companion | Deferred | Market demand |

---

## How We Make Decisions

1. **Grill the design** — stress-test assumptions (see `docs/adr/0001-license-system.md` for 13-question grilling)
2. **Write ADR** — context, decision, consequences, trade-offs
3. **Implement in slices** — each tested on live DB copy before next
4. **Click-test GUI** — untestable logic verified by human
5. **Ship migration** — with backup, behind feature flag if needed
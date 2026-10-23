# AcaSmart — Architecture Deep Dive

> This document expands on the architecture section of README.md for engineers who want to understand the system's structure, data flows, and module responsibilities.

---

## High-Level Layer Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                      UI LAYER (PySide6)                      │
│  ui/shell.py        ShellWindow — sidebar + QStackedWidget  │
│  ui/pages/*.py      one Page per Module (built once)        │
│  ui/reports/*.py    report sub-views hosted by ReportsPage  │
│  ui/widgets/*.py    palette, notification center, inputs,   │
│                     Shamsi date picker, theme manager       │
│  ui/windows/*.py    modal dialogs: login, license, password │
│  Presentation only — zero business logic                     │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                     SERVICES LAYER                           │
│  attendance_recording.py  renewal_reminder.py               │
│  sms_notifier.py          sms_worker.py (QThread)           │
│  notification_feed.py     backup_service.py                 │
│  Use cases — orchestrate repos, enforce invariants           │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                      CORE DOMAIN                             │
│  core/schedule.py      Weekly occurrence math (Shamsi)      │
│  core/license.py       Validation, trial, device binding    │
│  core/device_fingerprint.py  Cross-platform HW ID           │
│  core/config.py        .env loaded once, typed accessors    │
│  core/app_init.py      Startup: template DB, migrations     │
│  core/fa_collation.py  Persian collation for SQLite         │
│  Zero UI imports — pure Python domain logic                  │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                      DATA LAYER                              │
│  data/db.py              get_connection() context manager   │
│  data/schema.py          CREATE TABLE statements            │
│  data/migrator.py        Versioned migrations + backup      │
│  data/repos/*.py         One module per domain (plain funcs)│
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                      STORAGE                                 │
│  SQLite DB (single file, per-user OS app-data dir)          │
│  PRAGMA foreign_keys=ON, journal_mode=WAL, synchronous=NORMAL│
└─────────────────────────────────────────────────────────────┘
```

---

## UI Shell (ADR-0010)

The app used to be a launcher dashboard that opened a separate window per module, each with a back button. It is now **one window**:

```
┌──────────────────────── ShellWindow ────────────────────────┐
│ toolbar: theme · search box (Ctrl+K) · license · 🔔 badge    │
├───────────────────────────────────────────────┬─────────────┤
│                                               │ 🧭 داشبورد   │
│        QStackedWidget                         │ روزمره       │
│        (the active Page)                      │  حضور و غیاب │
│                                               │  پرداخت‌ها    │
│                                               │  پیامک       │
│                                               │ ثبت‌نام...    │
└───────────────────────────────────────────────┴─────────────┘
```

- **Pages are built once** by `ui/pages::default_pages()` and never torn down. On activation the shell duck-types `load_data()` / `refresh_data()`, so data edited on another page is never stale.
- **One navigation hub:** sidebar clicks, the Ctrl+K command palette, notification-card clicks and in-page `navigate_requested` signals all call `ShellWindow.navigate_to(key)`.
- **Modal survivors:** login, license activation, change password, license details, notification center and command palette are `QDialog`s.
- **RTL is applied app-wide** at theme bootstrap, so the sidebar sits on the right.
- `tests/unit/test_retired_patterns.py` fails the build if the old `BaseSecondaryWindow` / `return_target` pattern reappears.

---

## Module Responsibilities

### `core/` — Domain Rules (Zero UI Imports)

| Module | Responsibility |
|--------|----------------|
| `schedule.py` | Pure Shamsi date math: `days_between`, `is_weekly_occurrence`, `occurrence_dates`, `add_weeks`, `week_start_shamsi`, `weekday_fa`, `first_on_or_after` |
| `license.py` | Ed25519 verification, trial logic, device fingerprint matching, license storage |
| `device_fingerprint.py` | Cross-platform stable HW ID (Win: wmic with hidden console, macOS: IOPlatformUUID, Linux: machine-id); computed once per process |
| `config.py` | Single seam over `.env`: loaded once (path-aware, works inside the PyInstaller bundle), typed accessors for admin and IPPanel settings |
| `app_init.py` | `initialize_database()`: copy template DB on first run, run migrations, seed settings and the admin user |
| `fa_collation.py` | SQLite collation for Persian sorting |

### `data/db.py` — Connection Seam

```python
@contextmanager
def get_connection():
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    conn.execute("PRAGMA foreign_keys = ON")
    conn.execute("PRAGMA journal_mode = WAL")
    conn.execute("PRAGMA synchronous = NORMAL")
    try:
        yield conn
        conn.commit()
    except Exception:
        conn.rollback()
        raise
    finally:
        conn.close()
```

**Every repo call is `with get_connection() as conn:`.** The close is guaranteed because a connection left open blocks file replacement on Windows (`WinError 32`), which used to break backup/restore. `tests/unit/test_connection_lifecycle.py` asserts that failing repo calls leave no handle open.

### `data/repos/` — One Module Per Domain

| Repo | Key Functions |
|------|---------------|
| `enrollment_repo.py` | `enroll()` / `reschedule()` → `EnrollmentResult(status, term_id)`; interval-aware student & teacher conflict checks; `fetch_scheduled_students_for_class_on_date()` |
| `terms_repo.py` | `term_progress()` → `TermProgress`, `term_config()` → `TermConfig`, `consumed()`, `remaining()`, `refresh_completion()`, bulk `consumed_by_terms()` / `tuition_by_terms()` |
| `attendance_repo.py` | `insert_attendance_with_date()`, `delete_attendance()`, `fetch_attendance_status_by_date()` |
| `payments_repo.py` | `insert_payment()` (tuition capped at remaining debt), `get_remaining_tuition_debt()`, `delete_term_if_no_history()` |
| `profiles_repo.py` | Pricing profiles: create/update/delete (blocked when in use), default flip, `apply_profile_to_term()` |
| `dashboard_repo.py` | KPI queries: `active_student_count()`, `outstanding_debt()`, `near_completion_terms()`, `week_absence_rate()`, `payments_per_month()`, `recent_payments()` |
| `reports_repo.py` | Attendance grid, student-term financials, summaries, contacts — batched to avoid N+1 queries |
| `notifications_repo.py` | Renewal-reminder **latest-outcome ledger**: `mark_renew_sms_sent()`, `record_renew_sms_attempt()`, `fetch_resend_eligible_renew_rows()` |
| `send_log_repo.py` | **Append-only** `log_send_attempt()` / `fetch_send_log()` for the SMS Delivery Report |
| `students_repo.py`, `teachers_repo.py`, `teacher_instruments_repo.py`, `classes_repo.py`, `settings_repo.py` | CRUD + list helpers |

### `services/` — Use Cases

| Service | Responsibility |
|---------|----------------|
| `attendance_recording.py` | `record_attendance()` → `AttendanceOutcome`: write the row → `refresh_completion()` → plan the renewal reminder; the send is handed to a dispatcher, never run inline |
| `renewal_reminder.py` | `plan_reminder()` / `plan_resend()` → `ReminderPlan`; `execute_reminder_job()` — the single send path, which records the ledger as *sent* only on an explicit provider success |
| `sms_notifier.py` | `SmsNotifier` wraps IPPanel; `normalize_status()` maps any response to `SENT` / `FAILED` / `DISABLED` / unknown; `execute_invitation_job()` for Manual SMS |
| `sms_worker.py` | `SmsSendWorker(QThread)`: runs a batch of send jobs off the UI thread, emits `send_reported` per recipient and `batch_done` with a summary |
| `notification_feed.py` | `collect_actionable_notifications()` — pure data (no UI import) for the badge and Notification Center |
| `backup_service.py` | `create_verified_backup()`, `validate_backup_file()`, `apply_backup()` — online-backup API, never a raw file copy of a live WAL DB |

---

## Data Flow: Recording Attendance (End-to-End)

```
User clicks "Save" on AttendancePage
         │
         ▼
services.attendance_recording.record_attendance(term_id, student_id, class_id, date, status,
                                                dispatch=page._dispatch_reminder)
         │
         ├─▶ attendance_repo.insert_attendance_with_date()   [present / absent / canceled]
         │
         ├─▶ terms_repo.refresh_completion(term_id)
         │       ├─▶ consumed = present + absent (canceled never counts)
         │       ├─▶ limit   = Term snapshot → Pricing Profile → default setting
         │       ├─▶ reached → SET end_date (last counted date)
         │       └─▶ below   → CLEAR end_date (re-opens!)
         │
         └─▶ renewal_reminder.plan_reminder()  — due only when exactly 1 session remains
                 │
                 ▼  job handed to dispatch(); record_attendance returns AttendanceOutcome at once
         SmsSendWorker (QThread)
                 │
                 ├─▶ execute_reminder_job(job) → SmsNotifier (IPPanel)
                 ├─▶ explicit SENT  → ledger status='sent'
                 ├─▶ failed/unknown → ledger status='failed'|'unknown' → stays resendable
                 ├─▶ every outcome  → sms_send_log row (append-only)
                 └─▶ signals → page shows the exact per-recipient status
```

---

## Database Schema (Key Tables)

```sql
-- Core domain
CREATE TABLE students (id PK, name, birth_date, gender, national_code UNIQUE, phone, father_name, ...);
CREATE TABLE teachers (id PK, name, national_code UNIQUE, teaching_card_number, gender, phone,
                       birth_date, card_number, iban, ...);
CREATE TABLE teacher_instruments (teacher_id FK, instrument);
CREATE TABLE classes (id PK, name, teacher_id FK, instrument, day, start_time, end_time, room);

-- Enrollment (Model-B: schedule-as-truth)
CREATE TABLE student_terms (
    id PK, student_id FK, class_id FK,
    start_date TEXT,       -- Shamsi YYYY-MM-DD, snapped to the class weekday
    start_time TEXT,       -- HH:mm
    lesson_duration INT,   -- 30 or 60 minutes (conflicts use the full interval)
    end_date TEXT,         -- NULL = active, SET = completed (derived)
    sessions_limit INT,    -- SNAPSHOTTED at enrollment
    tuition_fee INT,       -- SNAPSHOTTED at enrollment
    currency_unit TEXT, profile_id FK, ...
);
CREATE UNIQUE INDEX idx_one_active_term
    ON student_terms(student_id, class_id) WHERE end_date IS NULL;

CREATE TABLE pricing_profiles (id PK, name, sessions, fee, currency_unit, is_default);

-- Attendance (three-state, date-keyed)
CREATE TABLE attendance (
    id PK, student_id FK, class_id FK, term_id FK, date TEXT,
    status TEXT NOT NULL DEFAULT 'present',   -- 'present' | 'absent' | 'canceled' (v3)
    cancel_reason TEXT,                       -- e.g. "rescheduled to ..." (v3)
    is_present INT,                           -- legacy boolean, kept for backward compatibility
    UNIQUE(student_id, class_id, term_id, date)
);

-- Payments
CREATE TABLE payments (
    id PK, student_id FK, class_id FK, term_id FK, amount INT, payment_date TEXT,
    payment_type TEXT, description TEXT,
    CHECK (amount > 0 AND payment_type IN ('tuition','extra'))   -- v2 table rebuild
);

-- SMS: latest outcome per term (resend logic) + append-only history (report)
CREATE TABLE sms_notifications (student_id, term_id, status, message, sent_at,
                                UNIQUE(student_id, term_id));
CREATE TABLE sms_send_log (id PK, student_id, term_id, phone, kind, status, message, sent_at);

-- License system
CREATE TABLE licenses (id PK, key_hash, key_payload, signature, device_fingerprint,
                       activated_at, expires_at, is_active);
CREATE UNIQUE INDEX idx_licenses_active_device
    ON licenses(device_fingerprint) WHERE is_active=1;

-- Settings (key-value) and users
CREATE TABLE settings (key PK, value);
CREATE TABLE users (id PK, mobile, password);   -- password stored hashed
```

---

## Migration System

- **Tracking:** `PRAGMA user_version`; `BASELINE_VERSION = 1`
- **Safety:** before applying anything pending, a consistent snapshot via SQLite's **online-backup API** (`sqlite3.Connection.backup`) to `backups/acasmart.v<from>.<timestamp>.db` — WAL-safe
- **Execution:** each migration manages its own transaction (`transactional(conn)` helper); on failure → restore the backup → abort startup with a clear message
- **Idempotency:** every step guards with `PRAGMA table_info` / `sqlite_master`, so it is safe to re-run

**Migration history (v2–v8 shipped):**
| Version | Change |
|---------|--------|
| v1 | Baseline schema |
| v2 | Payments integrity — table rebuilt with `CHECK (amount > 0 AND payment_type IN ('tuition','extra'))` |
| v3 | Three-state attendance — additive `status` + `cancel_reason`, backfilled from `is_present` |
| v4 | `lesson_duration` on terms |
| v5 | Merge duplicate active terms (1-hour lesson splits) + `idx_one_active_term` |
| v6 | **Drop `sessions` table** (Model-B complete) |
| v7 | SMS ledger becomes latest-outcome: `status` + `message` columns |
| v8 | Append-only `sms_send_log`, backfilled from the ledger |

---

## Cross-Platform Build Pipeline

```
git tag v1.x.y  ──▶  GitHub Actions: .github/workflows/build.yml
                          │
     ┌──────────────┬─────┴────────┬────────────────┬────────────────┐
     ▼              ▼              ▼                ▼                ▼
 Windows 7      Windows 10     macOS arm64      macOS x86_64      Linux x86_64
 Py 3.8         Py 3.11        Py 3.11          Py 3.11           Py 3.11
 PyInstaller    PyInstaller    PyInstaller      PyInstaller       PyInstaller
 5.7 main.spec  6.10 main.spec mac.spec         mac.spec          linux.spec
 Inno Setup     Inno Setup     ditto → .zip     ditto → .zip      tar.gz
     └──────────────┴──────────────┴────────┬───────┴────────────────┘
                                            ▼
                               GitHub Release (assets attached)
```

**Dual Python strategy:** `requirements.txt` uses environment markers to pin compatible versions for 3.8 (Windows 7: PySide6 6.0.4, `cryptography` 42.0.8) and 3.10+ (PySide6 6.7.3). `main.spec` refuses to build a Windows 7 bundle with a `cryptography` that imports Windows 8+ APIs.

---

## License System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      VENDOR (OFFLINE)                        │
│  tools/generate_keypair.py  →  Ed25519 key pair             │
│  tools/generate_license.py  →  Customer keys                │
│  Private key: NEVER distributed, chmod 600                  │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                      APP BINARY                              │
│  acasmart/core/license_public_key.py  ← Embedded public key │
│  acasmart/core/license.py             ← Validation logic    │
│  acasmart/core/device_fingerprint.py  ← HW ID               │
└────────────────────────────┬────────────────────────────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
         First Run      Valid License    Trial Active
              │              │              │
              ▼              ▼              ▼
         LicenseDialog   LoginWindow    LoginWindow
         (enter/start)       │          (+ trial banner)
                             ▼
                        ShellWindow
```

**Key format:** `AMG-<base32(payload + signature)>` — payload includes `cid`, `exp`, `tier`, optional `fp` (pre-bound device)

---

## Persian/RTL Considerations

- **Calendar:** All business dates stored/displayed as Shamsi `YYYY-MM-DD` text
- **Collation:** Custom SQLite collation (`fa_collation`) for correct Persian sorting
- **UI:** App-wide `RightToLeft` layout, Vazirmatn font bundled, Jalali date picker (`ShamsiDatePicker` / `ShamsiDateField`)
- **Input:** Phone, money and national-code fields accept Persian digits and normalize them; national codes are checksum-validated
- **Reports:** Excel exports with Persian headers

---

## Security Considerations

| Area | Approach |
|------|----------|
| License validation | Ed25519 — cannot forge without private key |
| Device binding | SHA256 fingerprint from stable HW attributes |
| Trial persistence | `keyring` (OS credential store) + DB settings table |
| Clock rollback | Not yet detected (planned) |
| Binary protection | None beyond PyInstaller packaging (obfuscation considered, not adopted) |
| Revocation | Placeholder — fail-open, log for future sync |
| DB integrity | WAL, FK constraints, CHECK constraints, partial unique indexes, pre-migration backups |
| Admin credentials | Password stored hashed in `users`; DB file `chmod 600` |

---

## Extensibility Points

| Extension | Current State | Effort |
|-----------|---------------|--------|
| Multi-branch (concurrent users) | Single-file SQLite, single-user | Major — needs PostgreSQL + sync |
| Group classes | Out of scope (one-to-one only, ADR-0001) | Medium — new attendance model |
| Cloud backup | Planned (local verified backup exists) | Medium — encrypt + upload DB |
| New module | Add a Page + one `PageEntry` in `default_pages()`; palette & sidebar pick it up | Small |
| Mobile companion | Not started | Major — separate codebase |

---

## Related Documents

- [DOMAIN_MODEL.md](DOMAIN_MODEL.md) — Canonical vocabulary
- [TECHNICAL_DECISIONS.md](TECHNICAL_DECISIONS.md) — ADR trade-offs
- [DEPLOYMENT.md](DEPLOYMENT.md) — Build, release, license operations
- [TEST_STRATEGY.md](TEST_STRATEGY.md) — Test coverage & philosophy

# AcaSmart — Architecture Deep Dive

> This document expands on the architecture section of README.md for engineers who want to understand the system's structure, data flows, and module responsibilities.

---

## High-Level Layer Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                      UI LAYER (PySide6)                      │
│  ui/windows/*.py  •  ui/widgets/*.py  •  ui/reports/*.py    │
│  Presentation only — zero business logic                      │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                     SERVICES LAYER                           │
│  services/attendance_recording.py   services/renewal_reminder.py
│  services/sms_notifier.py                                          │
│  Cross-repo workflows — orchestrate repos, enforce invariants   │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                      CORE DOMAIN                             │
│  core/schedule.py      Weekly occurrence math (Shamsi)      │
│  core/license.py       Validation, trial, device binding    │
│  core/device_fingerprint.py  Cross-platform HW ID           │
│  core/app_init.py      Startup, migrations, DB init         │
│  core/config.py        Settings, version, constants         │
│  core/fa_collation.py  Persian collation for SQLite         │
│  Zero external deps — pure Python domain logic               │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                      DATA LAYER                              │
│  data/db.py              get_connection() / tx()            │
│  data/schema.py          CREATE TABLE statements            │
│  data/migrations.py      Idempotent migration steps         │
│  data/migrator.py        Runner with WAL-safe backup        │
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

## Module Responsibilities

### `core/` — Domain Rules (Zero UI Imports)

| Module | Responsibility |
|--------|----------------|
| `schedule.py` | Pure Shamsi date math: `days_between`, `is_weekly_occurrence`, `occurrence_dates`, `add_weeks`, `weekday_fa`, `first_on_or_after` |
| `license.py` | Ed25519 verification, trial logic, device fingerprint matching, license storage |
| `device_fingerprint.py` | Cross-platform stable HW ID (Win: wmic, macOS: IOPlatformUUID, Linux: machine-id) |
| `app_init.py` | `initialize_database()`: run migrations, create licenses table, seed settings |
| `config.py` | `APP_VERSION`, `DB_VERSION`, default settings, currency units |
| `fa_collation.py` | SQLite collation for Persian sorting |

### `data/db.py` — Connection Factory

```python
def get_connection() -> sqlite3.Connection:
    con = sqlite3.connect(DB_PATH)
    con.row_factory = sqlite3.Row
    con.execute("PRAGMA foreign_keys=ON")
    con.execute("PRAGMA journal_mode=WAL")
    con.execute("PRAGMA synchronous=NORMAL")
    con.execute("PRAGMA busy_timeout=5000")
    return con

@contextmanager
def tx():
    con = get_connection()
    try:
        yield con
        con.commit()
    except Exception:
        con.rollback()
        raise
    finally:
        con.close()
```

**Every repo opens, works, closes** — no long-lived connections.

### `data/repos/` — One Module Per Domain

| Repo | Key Functions |
|------|---------------|
| `enrollment_repo.py` | `enroll()`, `reschedule()`, `has_student_schedule_conflict()`, `has_teacher_schedule_conflict()` |
| `terms_repo.py` | `refresh_term_completion()`, `get_active_terms()`, `get_term_with_financials()` |
| `attendance_repo.py` | `save_attendance()`, `get_attendance_for_term()`, `count_held_sessions()` |
| `payments_repo.py` | `insert_payment()`, `get_total_paid_for_term()`, `delete_term_if_no_payments()` |
| `students_repo.py`, `teachers_repo.py`, `classes_repo.py` | CRUD + list helpers for UI |
| `profiles_repo.py` | Pricing profiles (tuition, session limit, currency) |
| `settings_repo.py` | `get_setting()`, `set_setting()`, boolean helpers |
| `reports_repo.py` | Aggregated queries for financial/attendance/teacher reports |
| `notifications_repo.py` | Renewal reminder sent/unsent tracking |

### `services/` — Cross-Repo Workflows

| Service | Responsibility |
|---------|----------------|
| `attendance_recording.py` | `AttendanceManager.save_attendance()`: writes attendance rows → calls `refresh_term_completion()` → triggers renewal SMS if 1 session left |
| `renewal_reminder.py` | `RenewalReminderService.check_and_send()`: finds terms with 1 session left, sends via `SmsNotifier`, logs to `sms_notifications` **only on provider success** |
| `sms_notifier.py` | `SmsNotifier.send()`: wraps IPPanel, returns structured result (success/error/disabled) |

---

## Data Flow: Recording Attendance (End-to-End)

```
User clicks "Save" in AttendanceWindow
         │
         ▼
AttendanceManager.save_attendance(term_id, date, records[])
         │
         ├─▶ attendance_repo.upsert_attendance()  [Present/Absent/Canceled]
         │
         ├─▶ terms_repo.refresh_term_completion(term_id)
         │       │
         │       ├─▶ Count present + absent for term
         │       ├─▶ Compare to term.sessions_limit (snapshotted)
         │       ├─▶ If reached: SET end_date = today
         │       └─▶ If not: CLEAR end_date (re-opens!)
         │
         └─▶ If held_count == sessions_limit - 1:
                 renewal_reminder.check_and_send(term_id)
                      │
                      ├─▶ SmsNotifier.send(pattern_code_1, params)
                      │
                      └─▶ On SUCCESS: notifications_repo.mark_renewal_sent(term_id)
                          On FAILURE:  NO ledger entry → resendable later
```

---

## Database Schema (Key Tables)

```sql
-- Core domain
CREATE TABLE students (id PK, name, national_code, phone, ...);
CREATE TABLE teachers (id PK, name, national_code, phone, instruments[], ...);
CREATE TABLE classes (id PK, name, teacher_id FK, day, start_time, room, ...);

-- Enrollment (Model-B: schedule-as-truth)
CREATE TABLE student_terms (
    id PK,
    student_id FK, class_id FK,
    start_date TEXT,      -- Shamsi YYYY-MM-DD
    start_time TEXT,      -- HH:mm
    end_date TEXT,        -- NULL = active, SET = completed
    sessions_limit INT,   -- SNAPSHOTTED at enrollment
    tuition_fee INT,      -- SNAPSHOTTED at enrollment
    currency_unit TEXT,
    profile_id FK,
    lesson_duration INT,  -- 30 or 60 minutes
    updated_at TEXT
);
CREATE UNIQUE INDEX idx_one_active_term
    ON student_terms(student_id, class_id) WHERE end_date IS NULL;

-- Attendance (three-state)
CREATE TABLE attendance (
    term_id FK, date TEXT,  -- PK (term_id, date)
    status TEXT CHECK (status IN ('present','absent','canceled')),
    reason TEXT,            -- for canceled: "rescheduled to ..."
    PRIMARY KEY (term_id, date)
);

-- Payments
CREATE TABLE payments (
    id PK, term_id FK, amount INT, currency_unit TEXT,
    payment_type TEXT CHECK (type IN ('tuition','extra')),
    description TEXT, date TEXT, created_at TEXT
);

-- License system
CREATE TABLE licenses (
    id PK,
    key_hash TEXT,           -- SHA256 of key (for lookup)
    key_payload TEXT,        -- JSON payload
    signature TEXT,          -- Ed25519 signature
    device_fingerprint TEXT, -- 32-char hex
    activated_at TEXT,
    expires_at TEXT,
    is_active INT DEFAULT 1
);
CREATE UNIQUE INDEX idx_one_active_license_per_device
    ON licenses(device_fingerprint) WHERE is_active=1;

-- Settings (key-value)
CREATE TABLE settings (key PK, value);
-- Includes: trial_start_date, trial_device_fingerprint, term_session_count, term_fee, currency_unit
```

---

## Migration System

- **Tracking:** `PRAGMA user_version` (integer, increments per migration)
- **Safety:** Before *any* migration, WAL-safe file copy to `backups/acasmart_backup_<timestamp>.db`
- **Execution:** Each step in a transaction; on failure → restore backup → abort startup with message
- **Idempotency:** Migration functions check current state before applying (safe to re-run)

**Migration history (v1–v6 shipped):**
| Version | Change |
|---------|--------|
| v1 | Initial schema |
| v2 | Attendance `status` (three-state), payment `CHECK` constraints |
| v3 | Licenses table + indexes |
| v4 | Active-term unique index + duplicate cleanup |
| v5 | Two-way `end_date`, renewal-SMS success fix |
| v6 | **Drop `sessions` table** (Model-B complete) |

---

## Cross-Platform Build Pipeline

```
Source (Python + PySide6)
         │
         ├──────────────────────┬──────────────────────┐
         ▼                      ▼                      ▼
   Windows x86_64         Windows arm64           macOS arm64
   (Python 3.8)           (Python 3.11)           (Python 3.11)
         │                      │                      │
         ▼                      ▼                      ▼
   PyInstaller            PyInstaller             PyInstaller
   (main.spec)            (main.spec)             (mac.spec)
         │                      │                      │
         ▼                      ▼                      ▼
   Inno Setup            Inno Setup              ZIP bundle
   (setup.iss)           (setup.iss)             (.app)
         │                      │                      │
         └──────────────────────┴──────────────────────┘
                            │
                            ▼
                    GitHub Releases
                    (per-arch assets)
```

**Dual Python version strategy:** `requirements.txt` uses environment markers to pin compatible versions for 3.8 (legacy) vs 3.10+ (modern).

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
         (enter/start)   (unlocked)     (+ trial banner)
```

**Key format:** `AMG-<base32(payload + signature)>` — payload includes `cid`, `exp`, `tier`, optional `fp` (pre-bound device)

---

## Persian/RTL Considerations

- **Calendar:** All business dates stored/displayed as Shamsi `YYYY-MM-DD` text
- **Collation:** Custom SQLite collation (`fa_collation`) for correct Persian sorting
- **UI:** RTL layouts, Persian numerals, Jalali date picker widget (`ShamsiDatePicker`)
- **Reports:** Excel exports with Persian headers, RTL sheet direction

---

## Security Considerations

| Area | Approach |
|------|----------|
| License validation | Ed25519 — cannot forge without private key |
| Device binding | SHA256 fingerprint from stable HW attributes |
| Trial persistence | `keyring` (OS credential store) + DB settings table |
| Clock rollback | Planned: "last seen date" tracking in DB |
| Binary protection | PyArmor / pyconcrete obfuscation (separate step) |
| Revocation | Placeholder — fail-open, log for future sync |
| DB integrity | WAL, FK constraints, CHECK constraints, migration backups |

---

## Extensibility Points

| Extension | Current State | Effort |
|-----------|---------------|--------|
| Multi-branch (concurrent users) | Single-file SQLite, single-user | Major — needs PostgreSQL + sync |
| Group classes | Out of scope (one-to-one only) | Medium — new attendance model |
| Cloud backup | Planned | Medium — encrypt + upload DB |
| Plugin system | Not designed | Major — would need interface layer |
| Mobile companion | Not started | Major — separate codebase |

---

## Related Documents

- [DOMAIN_MODEL.md](DOMAIN_MODEL.md) — Canonical vocabulary
- [TECHNICAL_DECISIONS.md](TECHNICAL_DECISIONS.md) — ADR trade-offs
- [DEPLOYMENT.md](DEPLOYMENT.md) — Build, release, license operations
- [TEST_STRATEGY.md](TEST_STRATEGY.md) — Test coverage & philosophy
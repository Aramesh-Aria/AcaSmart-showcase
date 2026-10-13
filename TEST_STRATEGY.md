# AcaSmart — Test Strategy

> Philosophy, coverage areas, and CI roadmap for a desktop app with domain-heavy logic and GUI presentation layer.

---

## Testing Philosophy

| Principle | Practice |
|-----------|----------|
| **Domain logic is testable; GUI is not** | Repos + services = pure functions → 100% unit/integration coverage. GUI = click-tested by human. |
| **Test against real SQLite** | Tests use throwaway copy of real schema + data (`tempfile.mkstemp()`), not mocks. |
| **Test behavior, not implementation** | Assert `EnrollmentResult.status == CREATED`, not `execute() called with SQL`. |
| **Edge cases over happy paths** | Conflict detection, boundary conditions (exactly 1 session left), duplicate guards, clock rollback. |
| **Migrations are tested** | Each migration step verified on copy of production DB before ship. |

---

## Test Inventory (60+ tests)

### `test_enrollment.py` — Enrollment Schedule & Conflicts
| Test | Scenario |
|------|----------|
| `test_created` | New enrollment succeeds |
| `test_existing_is_idempotent` | Re-enroll same student/class/time → returns existing term |
| `test_duplicate_active_different_time` | Same student+class, different time → `DUPLICATE_ACTIVE` (partial unique index) |
| `test_teacher_conflict` | Two students, same teacher, overlapping interval → `TEACHER_CONFLICT` |
| `test_student_conflict` | One student, two classes, overlapping interval → `STUDENT_CONFLICT` |
| `test_reschedule` | Move term to new class/time → old term ends, new term created |
| `test_reschedule_teacher_conflict` | Reschedule into teacher conflict → blocked |
| `test_one_active_term_per_student_class` | Index enforces at DB level |

### `test_attendance_recording.py` — Three-State Attendance
| Test | Scenario |
|------|----------|
| `test_save_present_absent_canceled` | All three statuses written correctly |
| `test_canceled_does_not_count` | Canceled status excluded from held count |
| `test_reschedule_modeled_as_cancel_plus_attend` | Old date canceled, new date attended = net 1 session |
| `test_upsert_idempotent` | Re-saving same date/status is no-op |
| `test_reason_persisted_for_canceled` | Reschedule reason stored |

### `test_term_progress.py` & `test_finished_terms.py` — Derived Completion
| Test | Scenario |
|------|----------|
| `test_completion_sets_end_date` | Held count reaches limit → `end_date` set |
| `test_deleting_attendance_reopens_term` | Delete attendance → `end_date` cleared (derived!) |
| `test_completed_term_visible_in_history` | Completed terms queryable via `get_finished_terms()` |
| `test_session_limit_resolution` | Term snapshot → profile → default setting fallback |

### `test_term_config.py` — Session Limit Resolution
| Test | Scenario |
|------|----------|
| `test_term_snapshot_priority` | Term's own `sessions_limit` used first |
| `test_profile_fallback` | No term snapshot → pricing profile `session_count` |
| `test_default_setting_fallback` | No profile → `settings.term_session_count` |

### `test_renewal_reminder.py` — "Exactly One Left" Trigger
| Test | Scenario |
|------|----------|
| `test_trigger_on_exactly_one_remaining` | Save attendance leaving 1 → SMS queued |
| `test_no_trigger_on_zero_or_two` | 0 or 2 remaining → no SMS |
| `test_ledger_only_on_provider_success` | Provider error → no `sms_notifications` row → resendable |
| `test_manual_resend` | Explicit resend action works after failure |

### `test_config.py` — Settings Helpers
| Test | Scenario |
|------|----------|
| `test_get_set_setting` | Round-trip |
| `test_bool_helpers` | `get_bool_setting()` handles "1"/"0"/"true"/"false" |

### `tests/integration/test_license_e2e.py` — License Flow
| Test | Scenario |
|------|----------|
| `test_trial_start_and_expiry` | 7-day trial, expires correctly |
| `test_license_activation_valid_key` | Ed25519 verification, device bind, DB storage |
| `test_license_expired_blocks_app` | Expired key → LicenseDialog |
| `test_prebound_key_device_mismatch` | Pre-bound key fails on different device |
| `test_trial_persists_across_restart` | Trial start date in settings + keyring |
| `test_keyring_fallback` | keyring unavailable → DB-only trial storage |

---

## What Is NOT Automated (Click-Tested)

| Area | Verification Method |
|------|---------------------|
| Full GUI flows (dashboard → attendance → save → SMS) | Human click-through on each release |
| Dialog sequences (license → login → dashboard) | Human |
| Theme switching (light/dark) | Human |
| Date picker (Shamsi calendar widget) | Human |
| Installer (Inno Setup) → first run → activation | Human on clean VM |
| macOS Gatekeeper / notarization flow | Human (when budget allows) |

**Rationale:** PySide6 widget testing under `pytest-qt` is brittle for complex dialog chains. Human verification catches UX issues automation misses.

---

## CI Roadmap (Next Milestone)

```yaml
# .github/workflows/ci.yml (planned)
name: CI
on: [push, pull_request]
jobs:
  test:
    strategy:
      matrix:
        python: ["3.8", "3.11"]
        os: [ubuntu-latest, macos-latest, windows-latest]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: ${{ matrix.python }} }
      - name: Install deps
        run: pip install -r requirements.txt
      - name: Run tests
        run: pytest -v
      - name: Compile check
        run: python -m compileall -q acasmart
```

**Blockers for CI:**
- PySide6 on Windows CI needs `qt6-tools` or offscreen platform
- macOS ARM64 runners have queue times
- Python 3.8 wheels for some deps (numpy 1.21, pandas 1.3) — may need `pip install --only-binary=:all:`

---

## Test Data Strategy

- **No fixtures** — each test builds minimal schema + seed data in `setUp()`
- **Schema inline** — `_SCHEMA` constant per test file (matches production `data/schema.py`)
- **Isolation** — `tempfile.mkstemp()` per test class, `DB_PATH` monkey-patched
- **Shamsi dates** — Test data uses realistic Persian dates (e.g., `1403-01-04` = Farvardin 1, 1403)

---

## Coverage Goals

| Layer | Target | Current |
|-------|--------|---------|
| Repos (`data/repos/*.py`) | 95%+ | ~90% |
| Services (`services/*.py`) | 90%+ | ~85% |
| Core (`core/*.py`) | 100% | ~95% |
| Migrations | 100% (manual) | Verified on prod copy |
| GUI (`ui/`) | 0% (click-tested) | N/A |

---

## Running Tests Locally

```bash
# All tests
pytest -v

# Specific module
pytest tests/test_enrollment.py -v

# With coverage
pytest --cov=acasmart --cov-report=term-missing

# Integration only
pytest tests/integration/ -v
```

---

## Related

- [ARCHITECTURE.md](ARCHITECTURE.md) — Module map for what to test
- [DEPLOYMENT.md](DEPLOYMENT.md) — Release verification checklist
- [TECHNICAL_DECISIONS.md](TECHNICAL_DECISIONS.md) — Invariants that tests enforce
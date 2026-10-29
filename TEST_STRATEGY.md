# AcaSmart — Test Strategy

> Philosophy, coverage areas, and CI for a desktop app with domain-heavy logic and a PySide6 presentation layer.

---

## Testing Philosophy

| Principle | Practice |
|-----------|----------|
| **Test at the seam the user touches** | Domain rules at the repo/service seam; UI behaviour by driving the real page headlessly with pytest-qt — not by testing private helpers. |
| **Test against real SQLite** | Each test points `data.db.DB_PATH` at a throwaway file (`tempfile.mkstemp()`) built from the real schema. No database mocks. |
| **Fake only the network** | The SMS provider is the one injected dependency: fake transports return canned `SENT` / `FAILED` / unknown / raising responses. |
| **Test behaviour, not implementation** | Assert `EnrollmentResult.status == CREATED` or "the ledger says sent", never "`execute()` was called with this SQL". |
| **Edge cases over happy paths** | Exactly one session left, canceled-only terms, garbage provider responses, overlapping 60-minute lessons, junk backup files. |
| **Guard the architecture** | Tests fail if retired patterns return, if code stops being Python 3.8-compatible, or if a connection can leak. |

---

## Test Inventory (653 tests)

| Area | Files | What they prove (real test names) |
|------|-------|-----------------------------------|
| **Enrollment & schedule** | `test_enrollment.py`, `unit/test_schedule_week.py` | `test_teacher_conflict`, `test_student_conflict` (interval overlap, not exact time), `test_duplicate_active_different_time`, `test_existing_is_idempotent`, `test_before_previous_end` |
| **Term progress & config** | `test_term_progress.py`, `test_term_config.py` | `test_at_limit_is_complete_and_closes_on_last_date`, `test_under_limit_signals_reopen`, `test_consumed_excludes_canceled`, `test_falls_back_to_profile_when_term_limit_null` |
| **Attendance use case** | `test_attendance_recording.py` | `test_reminder_fires_at_one_remaining`, `test_canceled_does_not_consume_or_remind`, `test_final_session_completes_term_without_reminder`, `test_due_reminder_is_queued_not_sent_inline` |
| **Renewal reminder** | `test_renewal_reminder.py`, `test_notification_feed.py` | `test_failed_send_does_not_write_ledger`, `test_execute_unknown_response_is_not_sent_and_resendable`, `test_later_success_upgrades_earlier_failed_attempt`, `test_failed_attempt_suppresses_the_duplicate_reminder_card` |
| **SMS worker & send log** | `unit/test_sms_worker.py`, `unit/test_send_log.py` | `test_hanging_sender_does_not_freeze_the_caller`, `test_reminder_garbage_response_is_unknown`, `test_v7_db_backfills_one_log_row_per_ledger_row`, `test_disabled_is_never_logged` |
| **Payments & profiles** | `test_payments_repo.py`, `test_profiles_repo.py` | `test_insert_cap_rejects_overpay`, `test_extra_payment_is_uncapped_and_keeps_type_and_description`, `test_delete_used_profile_is_blocked_and_kept` |
| **Dashboard KPIs** | `unit/test_dashboard_repo.py`, `unit/test_dashboard_page.py` | `test_extra_payments_do_not_offset_tuition_debt`, `test_canceled_attendance_does_not_increase_consumption`, `test_counts_only_records_inside_the_shamsi_frame` |
| **Backup / restore** | `unit/test_backup_service.py`, `unit/test_settings_page.py` | `test_backup_survives_a_wal_mode_source`, `test_overwriting_a_sqlite_target_never_deletes_it`, `test_rejects_a_foreign_sqlite_without_core_tables` |
| **License** | `unit/test_license.py`, `unit/test_license_dialog.py`, `integration/test_license_e2e.py` | `test_prebound_key_only_works_on_correct_device`, `test_backup_restore_different_device_license_invalid`, `test_trial_expires_license_required` |
| **Device fingerprint** | `unit/test_device_fingerprint.py` | `test_wmic_spawns_with_create_no_window`, `test_fingerprint_computed_once_per_process`, `test_wmic_timeout_handled` |
| **Input widgets** | `unit/test_input_widgets.py`, `unit/test_shamsi_date_field.py` | `test_persian_digits_are_normalized_from_text`, `test_checksum_failure_marked_invalid_and_value_none`, `test_10_digit_starting_with_9_is_normalized` |
| **Jalali calendar & Academy Today** | `unit/test_jalali_calendar.py` | `test_iran_midnight_is_at_2030_utc`, `test_no_daylight_saving_in_summer`, `test_esfand_in_a_leap_year_has_30_days`, `test_day_clamps_to_esfand_of_a_common_year` |
| **Date picker** | `unit/test_shamsi_date_popup.py`, `unit/test_shamsi_date_picker.py` | `test_single_click_selects_without_closing`, `test_next_month_arrow_rolls_into_the_next_year`, `test_left_arrow_is_the_next_day_in_rtl`, `test_has_no_gregorian_api` |

### GUI tests (pytest-qt, headless)

~540 tests under `tests/unit/` run every page under `QT_QPA_PLATFORM=offscreen` and assert on what the user would see or what lands in the DB:

| Page / component | Example flows |
|------------------|---------------|
| Shell & navigation | `test_refresh_called_exactly_once_per_activation`, `test_data_edited_elsewhere_are_not_stale`, `test_page_navigate_requested_routes_through_the_hub` |
| Command palette | `test_ctrl_k_shortcut_is_registered`, `test_page_command_navigates_via_hub_exactly_once`, `test_backup_command_creates_verified_backup` |
| Attendance page | `test_save_persists_present_and_recomputes_progress`, `test_cancel_then_record_consumes_exactly_one_session`, `test_delete_removes_record_and_reopens_completed_term` |
| Manual SMS page | `test_send_hands_invitation_jobs_to_background_worker`, `test_retry_resends_only_failed_and_unknown_recipients`, `test_selection_send_outcome_and_retry_with_fake_transport` |
| Reports page | `test_each_report_type_opens_from_the_subnav`, `test_back_returns_to_the_previous_report` |
| Notification center | badge count from local data, card routing to the responsible page |
| Students / teachers / classes / sessions / payments / pricing profiles / settings | create, edit, delete-guard and refresh-contract flows per page |

### Architecture guard tests

| Test file | Guards against |
|-----------|----------------|
| `test_retired_patterns.py` | Reintroducing `BaseSecondaryWindow` / `return_target` or importing deleted window modules |
| `test_platform_conformance.py` | Python 3.9+ syntax in app code (AST scan: `match`, PEP 604 unions, builtin generics); `cryptography` ≥ 43 on the Windows 7 track; subprocesses spawning a visible console |
| `test_connection_lifecycle.py` | A repo call that raises leaving an SQLite handle open (`WinError 32` on Windows) |

---

## What Is Still Verified by Hand

| Area | Verification Method |
|------|---------------------|
| Installer → first run → activation | Human on a clean VM per release |
| Real IPPanel delivery | Human, with the academy's live account |
| macOS Gatekeeper on unsigned builds | Human |
| Visual polish (spacing, RTL glyph shaping) | Human; screenshots regenerated from a seeded demo DB |

---

## CI (GitHub Actions)

**`ci.yml`** — started manually from the Actions tab (no automatic runs on push or PR):

| OS | Python | Track |
|----|--------|-------|
| `windows-latest` | 3.8 | Windows 7 (PySide6 6.0.4) |
| `windows-latest` | 3.11 | Windows 10/11 (PySide6 6.7.3) |
| `macos-latest` | 3.11 | Apple Silicon |
| `ubuntu-latest` | 3.11 | Linux (fast feedback) |

Steps: `pip install -r requirements.txt` → `pytest -v` with `QT_QPA_PLATFORM=offscreen` → `python -m compileall -q acasmart`; a separate job runs `ruff check` (pinned version, curated rule set that stays compatible with the 3.8 track).

**`build.yml`** — release builds for all five targets, also started manually (see [DEPLOYMENT.md](DEPLOYMENT.md)).

### Lessons CI taught

- **Thread lifetime is a test concern.** A parentless `QThread` garbage-collected right after its last signal aborted the whole interpreter on the Windows runner only. Tests now join every worker before teardown, and a regression test widens the race on purpose.
- **Windows file locking.** Connections held open blocked backup replacement (`WinError 32`); fixed by making `get_connection()` a context manager that always closes, with tests to keep it that way.
- **No `exec()` / `qWait` on PySide6 6.0.4.** The Windows 7 track uses `exec_()` and `qtbot.waitUntil` on observable conditions.

---

## Test Data Strategy

- **Isolation** — `tempfile.mkstemp()` DB per test; `DB_PATH` patched in `data.db` (and `data.migrator` for migration tests)
- **Minimal seeds** — each test builds only the rows it needs through the real repo functions
- **Migration tests** — build an older-version DB, run `run_migrations()`, assert schema and data (e.g. v8 backfill is idempotent)
- **Shamsi dates** — realistic Persian dates (e.g. `1404-07-01`); week/month frames computed in Jalali

---

## Running Tests Locally

```bash
# All tests (headless)
QT_QPA_PLATFORM=offscreen pytest -v

# Domain only
pytest tests/test_enrollment.py tests/test_term_progress.py -v

# GUI pages
QT_QPA_PLATFORM=offscreen pytest tests/unit -v

# Lint + compile check (same as CI)
ruff check acasmart/ tests/
python -m compileall -q acasmart
```

---

## Related

- [ARCHITECTURE.md](ARCHITECTURE.md) — Module map for what to test
- [DEPLOYMENT.md](DEPLOYMENT.md) — Build & release pipeline
- [TECHNICAL_DECISIONS.md](TECHNICAL_DECISIONS.md) — Invariants that tests enforce

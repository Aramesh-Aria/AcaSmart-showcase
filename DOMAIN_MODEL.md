# AcaSmart — Domain Model (Canonical Vocabulary)

> Source: `CONTEXT.md` — this is the authoritative glossary. All code, docs, and conversations should use these terms.

---

## Core Entities

### Student
A learner enrolled in academy lessons, uniquely identified by **national code**. The Student owns the financial account for their Terms, even when a parent or guardian pays in practice. Phone is optional for enrollment but required for SMS delivery.

_Avoid:_ Client, Payer

### Teacher
An instructor who teaches one or more **Instruments**, uniquely identified by **national code**. A Class has exactly one primary Teacher. Teacher phone is optional contact information.

_Avoid:_ Staff member, Substitute

### Instrument
A named subject taught by Teachers and offered through Classes.

_Avoid:_ Skill, Course type

### Class
An academy offering taught by one Teacher for one Instrument, optionally associated with a room and default weekly time. A Class is **not** a Student's enrollment and does not by itself prove any Student attends.

_Avoid:_ Term, Enrollment

---

## Financial Domain

### Financial Account
The tuition, extra payments, debt, and settlement state for a Student's Term.

_Avoid:_ Parent account, Payer account

### Tuition Payment
A Payment applied to a Term's tuition debt. Tuition can be collected any time after the Term is created, including before the first Session is held, but a Tuition Payment cannot exceed the remaining tuition debt for that Term.

_Avoid:_ Extra payment

### Extra Payment
A non-tuition charge or payment associated with a Student's Term. Extra Payments are tracked for reporting but are not applied to tuition debt, and should have a description explaining why the money was collected.

_Avoid:_ Tuition payment

---

## Temporal Domain

### Term
A **paid enrollment cycle** for one Student in one Class. A Term is created when the academy confirms the Student's Enrollment Schedule for that Class, **snapshots the agreed tuition and session limit**, completes when total Attendance records (Present + Absence) reach that limit, and owns that cycle's tuition, extra payments, attendance records, and renewal reminders.

**Completion is a derived status, not a frozen lock:** a completed Term stays visible and editable, and editing its Attendance recomputes whether it is complete.

_Avoid:_ Course, Semester, Registration

### Renewal Reminder
An SMS reminder sent when **exactly one session remains** in a Term, after recording the Attendance that leaves one remaining. A Renewal Reminder counts as sent **only when the SMS provider returns an explicit successful or accepted response**; disabled sending, errors, provider rejection, and unknown responses remain eligible for resend.

_Avoid:_ End-of-term message, Generic SMS

### Manual SMS
A user-initiated message sent to selected Students outside the automatic Renewal Reminder workflow. Manual SMS attempts are recorded in the Send Log, but they never mark a Term's Renewal Reminder as sent.

_Avoid:_ Renewal reminder

### Send Log
An **append-only** history of every attempted outgoing SMS — Renewal Reminders and Manual SMS alike — one entry per attempt, carrying the recipient, message kind, outcome, and send time. Entries are never overwritten; it answers "what was sent to whom, when, and what happened." The rendered message body is never stored (the provider renders it from template parameters). Distinct from the Renewal Reminder ledger, which keeps only the **latest outcome** per Student Term for resend logic.

_Avoid:_ SMS ledger, inbox, pending list

### Send Outcome
The final result of one logged send attempt: **Sent**, **Failed**, or **Unknown**. There is no stored Pending state — an attempt is logged only once its outcome is known — and disabled sending logs nothing.

_Avoid:_ Pending

### SMS Delivery Report
A read-only report over the Send Log showing recipient, outcome, and send time, with status, date-range, student, and message-kind filters.

_Avoid:_ Call report

---

## Calendar & Time

### Business Date
A Shamsi/Jalali calendar date used for academy operations, stored and displayed as `YYYY-MM-DD`.

_Avoid:_ Gregorian business date

### Academy Today
The current Business Date as it is in Iran (UTC+3:30, no daylight saving), regardless of the computer's own timezone setting. It is the default date wherever the app means "today".

_Avoid:_ System date, local date

### Record Timestamp
The Gregorian date and time at which a row was created, changed, or a send was attempted. It is a system record, not a Business Date, and is shown to users only after conversion to Jalali.

_Avoid:_ Business Date, Shamsi timestamp

### Lesson Time
A local academy time for scheduling lessons, stored and displayed as `HH:mm`.

_Avoid:_ Timezone-aware lesson time

### Lesson Duration
The length of every lesson in a Term, fixed when the Term is created (typically 30 or 60 minutes). Duration affects scheduling conflict detection and tuition, but **not the session count**: a 60-minute lesson is one Session that counts once toward the Term's session limit.

_Avoid:_ Slot, Session weight

---

## Scheduling & Attendance

### Session
One actual dated meeting for one Student in one Class.

_Avoid:_ Weekly slot, Class schedule

### One-to-One Lesson
A lesson between one Teacher and one Student. AcaSmart's core scheduling and attendance model is centered on One-to-One Lessons.

_Avoid:_ Group class

### Group Class
A class format where multiple Students attend the same Session together. **Out of scope** for AcaSmart's current scheduling, attendance, tuition, and reminder model.

_Avoid:_ One-to-one lesson

### Enrollment Schedule
The recurring day and time commitment for one Student in one Class during a Term. The Enrollment Schedule is the **source of truth for normal future occurrences**, and neither a Student nor a Teacher can have overlapping Enrollment Schedules, where overlap is judged by Lesson Duration (a 60-minute lesson conflicts with anything in its full interval), not by an exact start-time match. Room is a descriptive label only and is not a conflict dimension.

_Avoid:_ Session, Class

### Attendance
The recorded outcome of a Session that actually happened. Attendance is either **Present** or **Absence**, and each consumes one session from the Term. A scheduled lesson that did not happen is recorded as a **Canceled Session** instead, which is not Attendance and consumes no session.

_Avoid:_ Cancellation, Rescheduling

### Present
An Attendance record showing that the Student attended a Session that the academy held.

_Avoid:_ Completed session

### Absence
An Attendance record showing that the academy held the scheduled Session and the Student missed it. An Absence consumes one session from the Term.

_Avoid:_ Cancellation, Skipped session

### Canceled Session
A scheduled Session that does not happen and therefore does not consume a session from the Student's Term. Canceled Sessions stay visible in history with a cancellation status or reason, but do not appear as attendable Sessions.

_Avoid:_ Absence, Deleted session

### Session Consumption
The number of sessions a Term has used, counted as its **Present + Absence** Attendance records; a Canceled Session never counts. A Term is complete when consumption reaches its session limit, where the limit is resolved as: Term snapshot → Pricing Profile → default `term_session_count` setting. Because completion is derived, deleting an Attendance record lowers consumption and can re-open a completed Term.

_Avoid:_ Held sessions, Attended count (present-only)

### Rescheduled Session
A scheduled Session moved to a different date or time while staying in the same Term. Rescheduling is **not a distinct record**: it is modeled as a Canceled Session (reason "rescheduled") on the original date plus normal Attendance on the new date, so the Term still consumes exactly one session.

_Avoid:_ Absence, Separate reschedule entity

---

## Data Integrity

### Deletion
Removal of a Term or Session that has no recorded history. Deletion is **blocked once a Term has any Tuition Payment, Extra Payment, or Attendance**. A Term that already has history is corrected by editing it directly — a deliberate, warned action — never by deletion.

_Avoid:_ Cancellation, Rescheduling, Adjustment

---

## Licensing Domain

### License Key
A cryptographically signed string (format `AMG-XXXX-XXXX-XXXX`) encoding customer ID, expiry date, and tier. Verified offline using embedded Ed25519 public key. Yearly expiration.

### Device Fingerprint
A stable, hashed identifier derived from hardware attributes (motherboard serial, disk serial, machine-id, MAC address). Used to bind a license to a specific device. Survives OS reinstall but changes on hardware replacement.

### Trial Period
A 7-day grace period for new devices without a license. Tied to device fingerprint. Stored locally with start date. Cannot be reset by reinstalling the app (fingerprint persists).

### License Activation
The process of validating a license key against its signature, expiry, and device fingerprint. On success, the license record is stored in the local SQLite database and the app unlocks.

### License Dialog
A modal window shown on first launch (or when license expires/invalid) before the Login Window. Allows key entry or trial start.

---

## Application Shell (UI vocabulary)

### Shell Window
The single main window (ADR-0010): a sidebar listing Modules plus a stacked Page area. Navigation of any kind — sidebar, in-page links, Command Palette, Notification Center cards — routes only through its `navigate_to()` hub.

_Avoid:_ Dashboard window, secondary windows, back button

### Module
A main app capability — Attendance, Payments, Manual SMS, Term Enrollment, Pricing Profiles, Students, Teachers, Classes, Reports, Settings — shown as a sidebar entry and one Page.

_Avoid:_ Window, screen tab

### Page
The widget hosting one Module inside the Shell. Built once, never torn down; refreshed from local repos each time the Shell activates it.

_Avoid:_ Screen, dialog

### Command Palette
A modal search dialog (Ctrl+K or the toolbar search box) with navigation Commands that route to Modules and app Commands (theme toggle, instant backup, exit).

_Avoid:_ Menu item, launcher button

### Notification Center
A read-only card dialog fed by local notification-eligibility data (Reminder-sendable Terms; failed/unknown Renewal Reminder sends). Clicking a card routes to the responsible Module; the toolbar badge always reflects local data, never the SMS provider.

_Avoid:_ SMS list, inbox

---

## Relationship Diagram

```
Student ──────┬──────► Financial Account ◄──────► Term ◄──────► Class ◄──────► Teacher
              │                   ▲                    │              │
              │                   │                    │              ▼
              │                   │                    │        Instrument
              │                   │                    │
              │                   └──────► Attendance (Present/Absence/Canceled)
              │
              └──────► Renewal Reminder (SMS) ──► Send Log (every attempt)
              │
              └──────► Enrollment Schedule (source of truth)
```

---

## Key Invariants (Enforced in Code)

1. **One active Term per Student per Class** — enforced by partial unique index: `UNIQUE(student_id, class_id) WHERE end_date IS NULL`
2. **No overlapping Enrollment Schedules** — student or teacher cannot have interval-overlapping weekly slots
3. **Term pricing snapshotted** — `tuition_fee`, `sessions_limit`, `currency_unit` copied to `student_terms` at creation
4. **Completion derived** — `end_date` set/cleared automatically by `terms_repo.refresh_completion()`
5. **Renewal Reminder counts as sent only on explicit provider success** — the latest-outcome ledger records `failed` / `unknown` attempts so they stay resendable; disabled sending records nothing
6. **Send Log is append-only** — one row per attempt, never updated
7. **Deletion blocked on history** — terms with payments/attendance cannot be deleted
8. **Tuition Payment capped** — a tuition payment cannot exceed the Term's remaining tuition debt
9. **License one-per-device** — partial unique index on `licenses(device_fingerprint) WHERE is_active=1`

---

## Persian-Localized Terms

| English | Persian (FA) | Usage |
|---------|--------------|-------|
| Student | دانش‌آموز | UI, reports |
| Teacher | استاد | UI, reports |
| Term | ترم | UI, DB, code |
| Session | جلسه | UI, code |
| Attendance | حضور و غیاب | UI, code |
| Present | حاضر | UI, DB |
| Absence | غایب | UI, DB |
| Canceled | لغو شده | UI, DB |
| Tuition | شهریه | UI, DB, reports |
| Renewal Reminder | یادآوری تمدید | UI, code |
| Enrollment Schedule | برنامه ثبت‌نام | UI, code |
| License Key | کلید لایسنس | UI, code |
| Trial | دوره آزمایشی | UI, code |
| SMS Delivery Report | گزارش پیامک‌ها | UI |
| Dashboard | داشبورد | UI |
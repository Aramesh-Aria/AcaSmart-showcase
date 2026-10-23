# AcaSmart — Video Demo Guide

> Reference for the demo video at https://aria-aramesh.ir/public/media/portfolio/project-1-demo.mp4

> **Note:** the current video was recorded on **v1.1**, before the sidebar-shell redesign (ADR-0010). The domain behaviour it shows is unchanged; the UI now looks like the [screenshots](#screenshots) below. A re-recording outline for v1.3 is at the end.

---

## Video Walkthrough Script (v1.1 recording)

### 0:00–0:30 — License Activation & Login
- First launch → License Dialog (trial vs key entry)
- Enter valid key → Login Window
- Trial banner shown (7 days remaining)

### 0:30–1:30 — Dashboard
- Launcher dashboard (replaced in v1.3 by the KPI dashboard inside the shell)

### 1:30–3:00 — Student Management
- Add student (national code, phone)
- Search/filter (Persian collation)
- Student detail: terms, payments, attendance history

### 3:00–4:30 — Teacher & Class Management
- Add teacher with instruments
- Create class (teacher, instrument, day, time, room)
- Teacher conflict detection

### 4:30–6:30 — Enrollment & Scheduling (Model-B)
- Enroll student in class → start date snapped to the class weekday
- **Schedule-as-truth:** no sessions table — lessons computed weekly
- Conflict detection: student and teacher interval overlap
- One-active-term rule enforced

### 6:30–8:30 — Attendance Recording
- Select class + date → list scheduled students (computed)
- Three-state: Present / Absent / Canceled
- Canceled with reason ("rescheduled to...")
- Save → term completion auto-updated
- **Renewal SMS triggers** when exactly 1 session left

### 8:30–10:00 — Payments & Financials
- Tuition payment (applies to debt, capped)
- Extra payment (description required)
- Financial report (Excel export)
- Student term summary (debt, paid, sessions left)

### 10:00–11:30 — Reports
- Attendance report (filter by class/date/teacher)
- Teacher summary
- Contact directory (students + teachers, export)

### 11:30–12:30 — Settings & License
- Pricing profiles (tuition, session limit)
- License details (expiry, device fingerprint)
- Theme toggle (light/dark)

---

## Key Moments to Highlight (for Employers)

| Timestamp | What It Demonstrates |
|-----------|---------------------|
| 0:15 | Ed25519 license validation, device fingerprint, trial logic |
| 4:45 | Schedule-as-truth: computed lessons, no `sessions` table |
| 5:30 | Interval-aware conflict detection (not just exact time match) |
| 7:15 | Three-state attendance + derived term completion |
| 8:00 | Auto SMS on "exactly 1 left" + ledger only on success |
| 9:30 | Snapshotted pricing — term immune to price list changes |

---

## Video Hosting

- **Primary:** https://aria-aramesh.ir/public/media/portfolio/project-1-demo.mp4
- **Showcase repo:** Link in README.md points to hosted version
- **Format:** MP4, 1080p, ~12 min, Persian narration + English subtitles (planned)

---

## Screenshots

All in [SCREENSHOTS/](SCREENSHOTS/), captured from **v1.3.21** on macOS (Retina, 2880×1800) over a seeded demo database — every name, phone number and national code is fictional.

| File | Shows |
|------|-------|
| `dashboard.png` | Shell + KPI dashboard: active students, debt, near-completion terms, revenue, weekly absence rate, recent payments |
| `dashboard_dark.png` | Same dashboard in the dark theme |
| `attendance.png` | Attendance for one class/date: computed roster, present/absent, cancel, delete, resend reminder |
| `attendance_dark.png` | Attendance in the dark theme |
| `term_enrollment.png` | Term enrollment: student, class, start date snapped to weekday, active enrollments of the class |
| `payments.png` | Payment form for a term with live debt / consumption status |
| `manual_sms.png` | Manual SMS: recipient list with preselection, message preview, retry for failed sends |
| `students.png` | Student form (checksum-validated national code, normalized phone) |
| `teachers.png` | Teacher form with taught instruments |
| `classes.png` | Classes, colour-coded by weekday, with day filter |
| `pricing_profiles.png` | Pricing profiles (sessions, fee) and default flag |
| `report_attendance.png` | Attendance report: per-date status grid, filters, Excel export |
| `report_attendance_dark.png` | Attendance report in the dark theme |
| `report_financial.png` | Financial report: tuition, extras, paid, debt per term |
| `report_summaries.png` | Student summaries: sessions held/absent, attendance rate |
| `report_sms_delivery.png` | SMS Delivery Report over the append-only send log |
| `report_contacts.png` | Contact directory (students + teachers) |
| `settings.png` | Academy settings, verified backup/restore, password and license details |
| `command_palette.png` | Ctrl+K command palette over the dashboard |
| `notification_center.png` | Notification Center: failed and due renewal reminders |
| `login.png` | Login window |
| `license_activation.png` | First-run license dialog: key entry or 7-day trial |

---

## Re-recording Outline (v1.3)

1. **0:00** License dialog → trial → login → shell opens on the KPI dashboard
2. **0:45** Sidebar sections; Ctrl+K palette — jump to a page, toggle dark theme, instant backup
3. **1:30** Enroll a student (weekday snapping, interval conflict warning)
4. **3:00** Attendance: save leaving one session → reminder sent in the background; UI stays responsive
5. **4:30** Notification Center: a failed reminder card → routes to SMS / attendance → resend
6. **5:30** Payments with debt cap; financial report; Excel export
7. **7:00** SMS Delivery Report — append-only history of every attempt
8. **8:00** Settings: verified backup → restore with guard

## Recording Notes

- Record at 1920x1080, 30fps
- Use OBS Studio or macOS Screen Recording
- Narrate in Persian; add English subtitles in post
- Use demo data only (the screenshot seed script generates a realistic fictional academy)
- Show Windows and macOS if possible

# AcaSmart — Video Demo Guide

> Reference for the demo video at https://aria-aramesh.ir/public/media/portfolio/project-1-demo.mp4

---

## Video Walkthrough Script

### 0:00–0:30 — License Activation & Login
- First launch → License Dialog (trial vs key entry)
- Enter valid key → Login Window
- Trial banner shown (7 days remaining)

### 0:30–1:30 — Dashboard
- Student/Teacher/Class counts
- Today's classes with attendance status
- Upcoming renewal reminders
- Quick actions toolbar

### 1:30–3:00 — Student Management
- Add student (national code, phone, instruments)
- Search/filter (Persian collation)
- Student detail: terms, payments, attendance history

### 3:00–4:30 — Teacher & Class Management
- Add teacher with instruments
- Create class (teacher, instrument, day, time, room)
- Teacher conflict detection (visual)

### 4:30–6:30 — Enrollment & Scheduling (Model-B)
- Enroll student in class → snap start date to class weekday
- **Schedule-as-truth:** No sessions table — lessons computed weekly
- Conflict detection: student interval overlap, teacher interval overlap
- One-active-term rule enforced

### 6:30–8:30 — Attendance Recording
- Select class + date → list scheduled students (computed)
- Three-state: Present / Absent / Canceled
- Canceled with reason ("rescheduled to...")
- Save → term completion auto-updated
- **Renewal SMS triggers** when exactly 1 session left

### 8:30–10:00 — Payments & Financials
- Tuition payment (applies to debt)
- Extra payment (description required)
- Financial report (Excel export)
- Student term summary (debt, paid, sessions left)

### 10:00–11:30 — Reports
- Attendance report (filter by class/date/teacher)
- Teacher summary (hours, students, revenue)
- Contact directory (students + teachers, export)

### 11:30–12:30 — Settings & License
- Pricing profiles (tuition, session limit per instrument)
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

## Screenshots (in SCREENSHOTS/)

| File | Shows |
|------|-------|
| `dashboard_preview.png` | Dashboard with today's classes, stats |
| `attendance_preview.png` | Attendance recording (three-state) |
| `student_manager_preview.png` | Student list + detail |
| `teacher_manager_preview.png` | Teacher + instrument management |
| `class_preview.png` | Class creation with schedule |
| `finance_preview.png` | Financial report + Excel export |
| `session_preview.png` | Session/enrollment manager |
| `sms_preview.jpg` | SMS notification window |

---

## Recording Notes (for future updates)

- Record at 1920x1080, 30fps
- Use OBS Studio or macOS Screen Recording
- Narrate in Persian; add English subtitles in post
- Blur any real student/teacher data (use demo data)
- Show both Windows and macOS if possible
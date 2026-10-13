# AcaSmart — Music Academy Management Desktop App

**Persian RTL · PySide6 · SQLite · In production at a music academy since 2023**

[![License: FSL-1.0-Apache-2.0](https://img.shields.io/badge/License-FSL--1.0--Apache--2.0-blue.svg)](LICENSE)
[![Python](https://img.shields.io/badge/Python-3.8%20%7C%203.11-blue)](requirements.txt)
[![UI](https://img.shields.io/badge/UI-PySide6%20(Qt)-green)](https://www.qt.io/)
[![Tests](https://img.shields.io/badge/Tests-pytest%20(60%2B)-orange)](tests/)

---

## 🎯 For Hiring Managers — 30-Second Summary

| Aspect | Detail |
|--------|--------|
| **Role** | Sole developer — requirements → domain modeling → architecture → implementation → tests → packaging → releases |
| **Scale** | Production desktop app, used daily by admin at a music academy; 12+ tagged releases |
| **Hardest problems solved** | Schedule-as-truth domain model (no `sessions` table), derived term completion (reversible), three-state attendance with reschedule-as-cancellation, Ed25519 offline license system with device binding & trial |
| **Tech depth** | Full stack: SQLite migrations (WAL, backup-on-migrate), Shamsi/Jalali calendar throughout, PySide6 custom theming, PyInstaller + Inno Setup cross-platform builds, cryptographic license keys |
| **Testing** | 60+ pytest tests covering enrollment conflicts, attendance logic, term completion, renewal reminders, license flows; integration tests for license E2E |

**Live demo:** [Video walkthrough](https://aria-aramesh.ir/public/media/portfolio/project-1-demo.mp4) · [Screenshots](SCREENSHOTS/)

---

## 📖 Documentation Index

| Document | Audience | Purpose |
|----------|----------|---------|
| [ARCHITECTURE.md](ARCHITECTURE.md) | Engineers | Layered architecture, data flow, module map |
| [DOMAIN_MODEL.md](DOMAIN_MODEL.md) | Engineers/PMs | Canonical domain vocabulary, bounded contexts |
| [TECHNICAL_DECISIONS.md](TECHNICAL_DECISIONS.md) | Architects | 9 ADRs with trade-off tables |
| [TEST_STRATEGY.md](TEST_STRATEGY.md) | QA/Leads | Test philosophy, coverage areas, CI strategy |
| [DEPLOYMENT.md](DEPLOYMENT.md) | DevOps/Engineers | Build pipeline, release process, license system |
| [COMMERCIAL.md](COMMERCIAL.md) | Buyers/Partners | Licensing terms, pricing model, support |

---

## 🏗️ Architecture at a Glance

```
SQLite DB (WAL, FK=ON)
    ↓
data/db.py          → Connection factory (Row factory, PRAGMAs)
data/repos/*.py     → Plain-function repos (no classes)
services/*.py       → Cross-repo workflows (attendance, SMS, reminders)
ui/windows/*.py     → PySide6 presentation only
core/*.py           → Domain rules (zero UI imports)
```

**Key principle:** Domain logic in `core/` and `services/` has **zero UI imports** — testable in isolation.

---

## ✨ Standout Features

| Feature | Technical Highlight |
|---------|---------------------|
| **Schedule-as-truth** | No `sessions` table — lessons computed weekly from enrollment; attendance is the only record of what happened |
| **Derived term completion** | Completion = `present + absent >= session_limit`; reversible — deleting attendance re-opens term |
| **Three-state attendance** | `Present` / `Absence` / `Canceled`; reschedule = cancel old + attend new (net 1 session) |
| **Snapshotted pricing** | Tuition & session limit frozen at enrollment; price changes affect only future terms |
| **Ed25519 license system** | Offline validation, device fingerprint (HW-bound), 7-day trial, yearly expiry, key rotation |
| **Auto renewal SMS** | Fires when exactly 1 session remains; ledger records only on provider success (resendable) |
| **Shamsi/Jalali throughout** | All business dates Persian calendar; custom date picker widget |

---

## 🛠️ Tech Stack

| Layer | Technology |
|-------|------------|
| Language | Python 3.8 (legacy Win7/8) + 3.11 (modern) |
| UI | PySide6 (Qt 6), custom QSS theming (token dicts + interpolation) |
| Database | SQLite (`foreign_keys=ON`, WAL, `PRAGMA user_version` migrations) |
| Dates | `jdatetime` / `jalali-core` — Shamsi `YYYY-MM-DD` stored as text |
| SMS | IPPanel (pattern-based templates) |
| Reporting | pandas / openpyxl (Excel export) |
| Packaging | PyInstaller + Inno Setup (Windows), PyInstaller (macOS) |
| Crypto | `cryptography` (Ed25519), `keyring` (trial persistence) |
| Testing | pytest, pytest-qt, 60+ tests |

---

## 📦 Releases & Distribution

- **12+ tagged releases** (current: `v1.1.25`)
- Per-architecture builds: `x86_64` (Intel/AMD), `arm64` (Apple Silicon / Windows ARM)
- Windows: `.exe` via Inno Setup installer (Win7, 8, 10, 11)
- macOS: `.app` bundle in ZIP (right-click → Open for Gatekeeper)
- License activation on first run (trial or key)

---

## 🧪 Testing Strategy

```
tests/
├── test_enrollment.py           # Conflict detection, one-active-term, idempotency
├── test_attendance_recording.py # Three-state writes, counting, cancellation
├── test_term_progress.py        # Derived completion, re-opening on delete
├── test_finished_terms.py       # History view, completed term queries
├── test_term_config.py          # Session limit resolution (snapshot → profile → default)
├── test_renewal_reminder.py     # "Exactly 1 left" trigger, ledger success-only rule
├── test_config.py               # Settings helpers
└── integration/
    └── test_license_e2e.py      # Full license flow: trial → activate → expire → renew
```

**Philosophy:** Test domain logic (repos + services) exhaustively; GUI flows click-tested. CI planned (GitHub Actions).

---

## 🚀 What's Next (Roadmap)

- [ ] GitHub Actions CI (pytest on push, matrix: Python 3.8/3.11, Ubuntu/macOS/Windows)
- [ ] Headless widget smoke tests (`QT_QPA_PLATFORM=offscreen`)
- [ ] Optional cloud backup for app-data DB
- [ ] Deeper modules: payments, reporting, notifications (see [IMPLEMENTATION-PLAN.md](docs/IMPLEMENTATION-PLAN.md))

---

## 📄 License

**Functional Source License 1.0 — Apache-2.0 Future Grant (2-year delay)**

This repository is a **professional showcase**. Source is viewable for evaluation. Commercial use requires a separate license.

→ See [LICENSE](LICENSE) for full terms  
→ See [COMMERCIAL.md](COMMERCIAL.md) for purchasing / licensing inquiries

---

## 📞 Contact

**Ahmad (Aria) Aramesh Moghaddam**  
Python/Django Developer — Desktop & Backend Systems  
📧 aramesh_aria@yahoo.com · 🔗 [github.com/Aramesh-Aria](https://github.com/Aramesh-Aria)  
📍 Open to opportunities — backend, desktop, domain-driven design, systems architecture
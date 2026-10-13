# AcaSmart — Commercial Licensing & Sales

> For music academy owners, distributors, and partners. This showcase repository demonstrates the product; commercial use requires a separate license agreement.

---

## Product Overview

**AcaSmart** is a Persian RTL desktop application for one-to-one music academies. It manages:
- Student & teacher records (instruments, contact, bank details)
- Weekly enrollment scheduling with conflict detection
- Per-session attendance (Present / Absent / Canceled)
- Tuition & extra payments, debt tracking
- Financial reporting (Excel export)
- Automatic renewal-reminder SMS (IPPanel) when 1 session remains
- Contact directory & manual bulk SMS

**In production** at a music academy in Iran since 2023. 12+ releases.

---

## Licensing Model

### Per-Academy Perpetual License (Recommended)
| Tier | Price (IRR) | Includes |
|------|-------------|----------|
| **Standard** | 150M / year | Unlimited students/teachers, all features, 1 device |
| **Multi-Device** | +50M / device | Additional device activations |
| **Support** | 30M / year | Email support, priority bug fixes, migration assistance |

### License Terms
- **License key format:** `AMG-XXXX-XXXX-XXXX` (Ed25519 signed, yearly expiry)
- **Device binding:** License tied to hardware fingerprint (survives OS reinstall)
- **Trial:** 7-day free trial on new devices (no key needed)
- **Renewal:** Generate new key with `tools/generate_license.py` — no app update needed
- **Source code:** Not included. Source escrow available for Enterprise tier.

### What You Get
- Installable builds (Windows `.exe`, macOS `.app`)
- License key(s) for your device(s)
- 1 year of updates (new releases)
- Email support (Persian/English)

---

## Pricing Rationale (for Iranian Market)

| Factor | Consideration |
|--------|---------------|
| **Target** | Small–medium academies (50–500 students) |
| **Current alternative** | Paper + spreadsheets + manual SMS |
| **Value** | Saves ~20 hrs/month admin time; prevents revenue leakage from missed renewals |
| **Price anchor** | ~$300–500/year comparable to SaaS (but on-premise, data stays local) |
| **Currency** | IRR pricing, adjusted annually for inflation |

---

## Purchase Process

1. **Demo:** Schedule a 30-min video call — I'll walk through the app on your screen
2. **Trial:** Install on your academy device — 7-day free trial (full features)
3. **Quote:** I'll send a proforma invoice (IRR, includes VAT if applicable)
4. **Payment:** Bank transfer (Iran: SHETAB / Card-to-card; International: Wise/crypto)
5. **Delivery:** License key emailed + download link for your architecture
6. **Activation:** Enter key on first launch — app unlocks immediately
7. **Support:** 30-day onboarding support included

---

## Data Ownership & Portability

- **Your data, always:** SQLite DB lives on YOUR machine (`~/Library/Application Support/AcaSmart/acasmart.db` or `%LOCALAPPDATA%\AcaSmart\acasmart.db`)
- **No cloud dependency:** App works 100% offline (except SMS sending)
- **Export anytime:** Excel reports for students, payments, attendance, teachers
- **Backup:** Single file copy — no proprietary format
- **No vendor lock-in:** Schema documented in [ARCHITECTURE.md](ARCHITECTURE.md)

---

## Support & SLA

| Tier | Response Time | Channel |
|------|---------------|---------|
| Standard | 48 hours | Email |
| Priority (add-on) | 4 hours business hours | Email + Telegram |
| Emergency | 2 hours | Phone/Telegram (critical bugs only) |

**Bug fixes:** Critical bugs patched within 1 week, delivered as point release.

**Migrations:** Schema migrations tested on your data copy before you update.

---

## Roadmap Commitments (Next 12 Months)

| Feature | Target |
|---------|--------|
| Multi-device license management UI | Q1 |
| Optional encrypted cloud backup (Iran-hosted) | Q2 |
| Teacher portal (mobile web view) | Q3 |
| Group class support (MVP) | Q4 |
| Financial dashboard (charts, trends) | Q4 |

---

## FAQ

### Can I try before buying?
**Yes.** Download the latest release → 7-day full-feature trial. No credit card, no registration.

### What if I change computers?
Enter your license key on the new device. If you hit the device limit, contact me for a reset (verified ownership required).

### What if the SMS provider (IPPanel) changes API?
I'll release a patched version. SMS is isolated in `services/sms_notifier.py` — single integration point.

### Can I get the source code?
**Source escrow** available for Enterprise tier (annual fee). Standard license = binary only.

### Is there a web version?
No. AcaSmart is deliberately **offline-first desktop** — Iranian academies often have unreliable internet, and student data stays on-premise.

### Do you offer customization?
Custom fields, report templates, or workflow tweaks: available as paid development (separate quote).

---

## Contact for Commercial Inquiries

**Ahmad (Aria) Aramesh Moghaddam**  
📧 aramesh_aria@yahoo.com  
💬 Telegram: @ArameshAria  
🌐 github.com/Aramesh-Aria  

**Response time:** Within 24 hours (Iran business days)

---

## Legal

This showcase repository is licensed under **FSL-1.0-Apache-2.0** (2-year delay).  
Commercial use of the **software product** requires a separate **commercial license agreement** — contact above.

The showcase source code is for **evaluation only**. It does not grant rights to use, modify, or distribute the software commercially.
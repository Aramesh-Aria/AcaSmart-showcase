# AcaSmart — Deployment, Releases & License Operations

> Build pipeline, release process, and vendor operations for the license system.

---

## Build Pipeline

### Prerequisites

| Platform | Python | Key Dependencies |
|----------|--------|------------------|
| Windows x86_64 | 3.8.10 | PySide6 6.0.4, numpy 1.21.6, pandas 1.3.5 |
| Windows arm64 | 3.11+ | PySide6 6.7.3, numpy 1.26.4, pandas 2.2.2 |
| macOS arm64 | 3.11+ | PySide6 6.7.3, pyobjc-core 11.1 |

**Virtual environments maintained per platform:**
- `.venv-win-x86` (Python 3.8)
- `.venv-win-arm64` (Python 3.11)
- `.venv-macos-arm64` (Python 3.11)

### Build Commands

```bash
# Windows (run on each architecture)
source .venv-win-x86/bin/activate
pyinstaller packaging/main.spec --clean --noconfirm
# Output: dist/AcaSmart/ → Inno Setup → AcaSmart_v1.x.x_x86_setup.exe

source .venv-win-arm64/bin/activate
pyinstaller packaging/main.spec --clean --noconfirm
# Output: dist/AcaSmart/ → Inno Setup → AcaSmart_v1.x.x_arm64_setup.exe

# macOS (Apple Silicon)
source .venv-macos-arm64/bin/activate
pyinstaller packaging/mac.spec --clean --noconfirm
# Output: dist/AcaSmart.app → ZIP → AcaSmart_v1.x.x_macos_arm64.zip
```

### Spec Files

- `packaging/main.spec` — Windows: hidden imports, datas (resources, license key), icon, uac_admin
- `packaging/mac.spec` — macOS: bundle identifier, icon, codesign prep (ad-hoc for now)
- `packaging/setup.iss` — Inno Setup: license page removed (app-level license), per-arch output

---

## Release Process

### Versioning
- **Scheme:** `v<major>.<minor>.<patch>` (e.g., `v1.1.25`)
- **Tags:** Git annotated tags on `main` branch
- **Current:** 12+ releases, latest `v1.1.25`

### Release Checklist

```markdown
- [ ] All tests pass locally (`pytest -v`)
- [ ] Compile check passes (`python -m compileall -q acasmart`)
- [ ] Version bumped in `acasmart/core/version.py`
- [ ] CHANGELOG.md updated (or release notes drafted)
- [ ] Build Windows x86_64 → test on Win7 VM
- [ ] Build Windows arm64 → test on Win11 ARM
- [ ] Build macOS arm64 → test on Apple Silicon
- [ ] Verify license activation on clean VM (trial + key)
- [ ] Create GitHub Release with all 3 assets
- [ ] Tag: `git tag -a v1.x.x -m "Release v1.x.x"`
- [ ] Push: `git push origin v1.x.x`
```

### GitHub Releases
- Assets named: `AcaSmart_v<version>_<arch>.zip` / `.exe`
- Release notes: Features, fixes, migration notes (if schema changed)
- Pre-release checkbox for beta builds

---

## License System Operations

### Vendor Key Generation (Offline)

```bash
# One-time: generate key pair (run once, store private key SECURELY)
python tools/generate_keypair.py
# Output: tools/license_private.key (chmod 600!)
#         acasmart/core/license_public_key.py (auto-generated, committed)

# Generate customer license (standard 1-year)
python tools/generate_license.py --customer CUST-001 --days 365
# Output: AMG-ABCD-EFGH-IJKL-MNOP

# Generate with specific expiry
python tools/generate_license.py --customer CUST-001 --expires 1404-12-29

# Pre-bound key (for non-technical customer)
python tools/generate_license.py --customer CUST-001 --days 365 --device-fp a1b2c3d4...

# Save to file for delivery
python tools/generate_license.py --customer CUST-001 --days 365 --output license_CUST-001.txt
```

### Key Rotation (If Private Key Compromised)

```bash
# 1. Generate new key pair
python tools/generate_keypair.py

# 2. Update embedded public key in app
#    → acasmart/core/license_public_key.py is auto-generated

# 3. Rebuild app (all architectures)

# 4. Re-issue ALL customer keys with new private key
#    (old keys become invalid on new app version)
```

### License Delivery to Customers

| Customer Type | Delivery Method |
|---------------|-----------------|
| Technical (can enter key) | Email key string: `AMG-ABCD-EFGH-IJKL-MNOP` |
| Non-technical | Pre-bound key + `.license` file (double-click to import — future) |
| Trial users | No key needed — app offers 7-day trial on first launch |

---

## Database Migrations in Production

### On User Machine (App Startup)
```python
# acasmart/core/app_init.py
def initialize_database():
    run_migrations()  # Checks PRAGMA user_version, applies pending
```

**Migration safety:**
1. Backup: `shutil.copy2(DB_PATH, backup_path)` (WAL-safe — copies `-wal`/`-shm` too)
2. Run migration steps in transaction
3. On any failure: `shutil.copy2(backup_path, DB_PATH)` → abort with message
4. On success: `PRAGMA user_version = new_version`

### Vendor: Testing Migrations Before Release
```bash
# 1. Copy production DB from test academy
cp ~/Library/Application\ Support/AcaSmart/acasmart.db test_migration.db

# 2. Point test DB at migration code
DB_PATH=test_migration.db python -c "
from acasmart.data.migrator import run_migrations
run_migrations()
print('Migration OK')
"

# 3. Verify schema + data integrity
sqlite3 test_migration.db ".schema"
sqlite3 test_migration.db "SELECT * FROM student_terms LIMIT 5;"
```

---

## Environment Variables (`.env`)

| Variable | Required | Purpose |
|----------|----------|---------|
| `ADMIN_MOBILE` | Yes | Admin login username |
| `ADMIN_PASSWORD` | Yes | Admin login password (hashed in DB) |
| `IPPANEL_API_KEY` | For SMS | IPPanel API key |
| `IPPANEL_FROM_NUMBER` | For SMS | Sender number |
| `IPPANEL_PATTERN_CODE_1` | For auto-SMS | Pattern: "1 session left" (params: student_name, class_name) |
| `IPPANEL_PATTERN_CODE_2` | For manual SMS | Pattern: "Register new term" (param: student_name) |

**Development:** Copy `.env.example` → `.env` and fill.

**Production:** Set in OS environment or bundled in installer (Inno Setup `[Registry]` section).

---

## Distribution Notes

### Windows
- **Installer:** Inno Setup (`.exe`), per-user install (no admin required)
- **App data:** `%LOCALAPPDATA%\AcaSmart\acasmart.db`
- **Uninstall:** Clean removal via Windows Settings
- **Updates:** Download new installer, run (preserves app-data DB)

### macOS
- **Bundle:** `.app` in ZIP (no DMG/notarization yet — $99/yr Apple Developer Program)
- **Gatekeeper:** User must right-click → Open on first run
- **App data:** `~/Library/Application Support/AcaSmart/acasmart.db`
- **Updates:** Download new ZIP, replace `.app` (preserves app-data DB)

### Architecture-Specific Assets
| Asset | Platform | Python |
|-------|----------|--------|
| `AcaSmart_v1.x.x_x86_setup.exe` | Windows 7+ (Intel/AMD) | 3.8 |
| `AcaSmart_v1.x.x_arm64_setup.exe` | Windows 10/11 ARM | 3.11 |
| `AcaSmart_v1.x.x_macos_arm64.zip` | macOS 11+ (Apple Silicon) | 3.11 |

---

## Support & Maintenance

| Task | Frequency |
|------|-----------|
| Dependency updates (security) | Monthly |
| Python version support review | Per release |
| License key renewals | Per customer (yearly) |
| Migration testing on prod copy | Pre-release |
| Backup verification | Per release |

---

## Future: CI/CD Automation

```yaml
# Planned .github/workflows/release.yml
on:
  workflow_dispatch:
    inputs:
      version:
        type: string
        required: true
jobs:
  build-windows-x86:
    runs-on: windows-2022
    steps: [...]
  build-windows-arm64:
    runs-on: windows-11-arm  # when available
    steps: [...]
  build-macos:
    runs-on: macos-14  # Apple Silicon
    steps: [...]
  create-release:
    needs: [build-windows-x86, build-windows-arm64, build-macos]
    steps:
      - uses: softprops/action-gh-release@v1
        with:
          files: |
            dist/*_x86_setup.exe
            dist/*_arm64_setup.exe
            dist/*_macos_arm64.zip
```

---

## Related

- [ARCHITECTURE.md](ARCHITECTURE.md) — Build outputs, module structure
- [TECHNICAL_DECISIONS.md](TECHNICAL_DECISIONS.md) — Migration strategy (ADR-0008)
- [COMMERCIAL.md](COMMERCIAL.md) — License delivery to paying customers
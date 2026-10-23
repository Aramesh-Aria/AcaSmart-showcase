# AcaSmart — Deployment, Releases & License Operations

> Build pipeline, release process, and vendor operations for the license system.

---

## Build Pipeline (GitHub Actions)

All shipping artifacts are built by **`.github/workflows/build.yml`** from a clean checkout — no hand-built binaries.

| Target | Runner | Python / Qt / PyInstaller | Spec | Output |
|--------|--------|---------------------------|------|--------|
| Windows 7 | `windows-latest` | 3.8 · PySide6 6.0.4 · 5.7 | `main.spec` | Inno Setup installer |
| Windows 10/11 | `windows-latest` | 3.11 · PySide6 6.7.3 · 6.10 | `main.spec` | Inno Setup installer |
| macOS Apple Silicon | `macos-15` | 3.11 · PySide6 6.7.3 · 6.10 | `mac.spec` | `.app` zipped with `ditto` |
| macOS Intel | `macos-15-intel` | 3.11 · PySide6 6.7.3 · 6.10 | `mac.spec` | `.app` zipped with `ditto` |
| Linux x86_64 | `ubuntu-22.04` (oldest glibc → widest compatibility) | 3.11 · PySide6 6.7.3 · 6.10 | `linux.spec` | one-dir bundle, `tar.gz` |

### What the workflow guarantees

- **Version consistency** — fails fast unless `acasmart/core/version.py`, `setup.iss` `AppVersion`, `mac.spec` `APP_VERSION` and the tag (`v1.3.21` style) all agree. Line-ending agnostic (`setup.iss` is often CRLF).
- **Pinned redistributable** — the Windows 7-tested `VC_redist.x64.exe` (14.44.35208) is downloaded from a dedicated `build-deps` release and **SHA-256 verified** — never "latest from Microsoft", which may drop Windows 7.
- **Hand-build layout reproduced** — `setup.iss` resolves sources relative to itself (`AcaSmart-repo\dist\…`, `Setup_files\VC_redist.x64.exe`), so the job lays out the workspace the same way instead of editing the script.
- **Architecture check** — macOS jobs assert the binary's arch with `lipo -archs`.
- **Bundled Qt dependencies on Linux** — `libxcb-cursor0` & friends are installed on the runner so PyInstaller bundles them (Qt 6.5+ needs them; many desktops lack them).
- **Configuration from secrets** — the `.env` the app reads at runtime is written from an Actions secret during the build; it never lives in the repository.
- **Windows 7 guard** — `main.spec` refuses to build if `cryptography` ≥ 43 is installed on Python 3.8 (its Rust module imports Windows 8+ APIs).

### Spec Files

- `packaging/main.spec` — Windows: hidden imports, data files (template DB, fonts, icons), runtime hook, `.ico`
- `packaging/mac.spec` — macOS: `BUNDLE` with bundle identifier, `Info.plist` (version, `LSMinimumSystemVersion` 11.0, dark-mode support)
- `packaging/linux.spec` — Linux: standard one-dir layout (`exclude_binaries=True`) — `main.spec` can't be reused because without an `.exe` suffix its EXE and COLLECT outputs collide
- `packaging/setup.iss` — Inno Setup: x64, `MinVersion=6.1` (Windows 7), optional VC++ redistributable task, desktop shortcut

### Local build (still supported)

```bash
pyinstaller --noconfirm --clean packaging/main.spec    # Windows → dist/AcaSmart/ → ISCC setup.iss
pyinstaller --noconfirm --clean packaging/mac.spec     # macOS   → dist/AcaSmart.app
pyinstaller --noconfirm --clean packaging/linux.spec   # Linux   → dist/AcaSmart/
```

---

## Release Process

### Versioning
- **Scheme:** `v<major>.<minor>.<patch>` (e.g., `v1.3.21`)
- **Current:** `v1.3.21`; 12+ releases

### Release Checklist

```markdown
- [ ] CI green on main (tests on 4 OS/Python configs + ruff)
- [ ] Version bumped in version.py, setup.iss, mac.spec
- [ ] Create the release / push tag `v1.x.y` → build.yml builds all 5 targets
- [ ] Assets attached to the release automatically (draft if the tag had no release)
- [ ] Smoke-test Windows 7 installer on a VM, Windows 10/11, macOS, Linux
- [ ] Verify license activation on a clean VM (trial + key)
- [ ] Publish release notes (features, fixes, migration notes if schema changed)
```

### Release Assets

| Asset | Platform |
|-------|----------|
| `AcaSmart-<ver>-win7-x64-setup.exe` | Windows 7 / 8 |
| `AcaSmart-<ver>-win10-x64-setup.exe` | Windows 10 / 11 |
| `AcaSmart-<ver>-macos-arm64.zip` | macOS 11+ Apple Silicon |
| `AcaSmart-<ver>-macos-x86_64.zip` | macOS 11+ Intel |
| `AcaSmart-<ver>-linux-x86_64.tar.gz` | Linux x86_64 |

---

## License System Operations

### Vendor Key Generation (Offline)

```bash
# One-time: generate key pair (store the private key SECURELY — never committed)
python tools/generate_keypair.py
# Output: tools/license_private.key (chmod 600, gitignored)
#         acasmart/core/license_public_key.py (embedded public key, committed)

# Standard 1-year customer license
python tools/generate_license.py --customer CUST-001 --days 365

# Specific expiry (Gregorian YYYY-MM-DD)
python tools/generate_license.py --customer CUST-001 --expires 2027-03-20

# Pre-bound key (for a non-technical customer — works only on that device)
python tools/generate_license.py --customer CUST-001 --days 365 --device-fp a1b2c3d4...

# Save to file for delivery
python tools/generate_license.py --customer CUST-001 --days 365 --output license_CUST-001.txt
```

### Key Rotation (If Private Key Compromised)

1. `python tools/generate_keypair.py` — new pair, new embedded public key
2. Tag a release — CI rebuilds every target with the new key
3. Re-issue all customer keys with the new private key (old keys stop validating on the new version)

### License Delivery to Customers

| Customer Type | Delivery Method |
|---------------|-----------------|
| Technical (can enter key) | Email the key string |
| Non-technical | Pre-bound key generated from their device fingerprint (shown in License Details) |
| Trial users | No key needed — app offers a 7-day trial on first launch |

---

## Database Migrations in Production

### On User Machine (App Startup)
```python
# acasmart/core/app_init.py
def initialize_database():
    # first run: copy the bundled template DB to the app-data dir (chmod 600)
    run_migrations()  # PRAGMA user_version → apply pending v2..v8
```

**Migration safety:**
1. Backup: SQLite **online-backup API** (`Connection.backup`) → `backups/acasmart.v<from>.<timestamp>.db` — a consistent snapshot even in WAL mode
2. Each migration runs in its own transaction and is idempotent
3. On any failure: restore the backup → abort startup with a clear message
4. On success: `PRAGMA user_version = <new>`

### Vendor: Testing Migrations Before Release
```bash
# Point the app's DB paths at a copy of a real database and run the migrator
cp ~/Library/Application\ Support/AcaSmart/acasmart.db /tmp/migration_test.db
python - <<'EOF'
import acasmart.data.db as db, acasmart.data.migrator as m
db.DB_PATH = m.DB_PATH = "/tmp/migration_test.db"
m.run_migrations(); print("Migration OK")
EOF
sqlite3 /tmp/migration_test.db "PRAGMA user_version; SELECT COUNT(*) FROM sms_send_log;"
```

---

## Environment Variables (`.env`)

| Variable | Required | Purpose |
|----------|----------|---------|
| `ADMIN_MOBILE` | Yes | Admin login username |
| `ADMIN_PASSWORD` | Yes | Initial admin password (stored hashed in the DB) |
| `IPPANEL_API_KEY` | For SMS | IPPanel API key |
| `IPPANEL_FROM_NUMBER` | For SMS | Sender number |
| `IPPANEL_PATTERN_CODE_1` | For auto-SMS | Pattern: "one session left" (params: `student_name`, `class_name`) |
| `IPPANEL_PATTERN_CODE_2` | For manual SMS | Pattern: "register for a new term" (param: `student_name`) |

Read once by `core/config.py` (path-aware, so it works inside the PyInstaller bundle). **Development:** `.env` at the repo root. **Release builds:** written from a GitHub Actions secret during the build and bundled with the app.

---

## Distribution Notes

### Windows
- **Installer:** Inno Setup `.exe`, installs to Program Files (administrator rights required), Start-menu + optional desktop shortcut, optional VC++ redistributable for Windows 7
- **App data:** `%LOCALAPPDATA%\AcaSmart\acasmart.db`
- **Updates:** run the new installer — the app-data DB is untouched and migrated on next launch

### macOS
- **Bundle:** `.app` in ZIP, unsigned (no notarization yet)
- **Gatekeeper:** right-click → Open on first launch (or `xattr -cr AcaSmart.app`)
- **App data:** `~/Library/Application Support/AcaSmart/acasmart.db`
- **Updates:** replace the `.app` (app-data DB preserved)

### Linux
- **Bundle:** extract the `tar.gz`, run `AcaSmart/AcaSmart`
- **App data:** `$XDG_DATA_HOME/AcaSmart/` (default `~/.local/share/AcaSmart/`)

### Backup & Restore (in-app)
- Settings page and command palette: **verified backup** — online-backup API, then `PRAGMA integrity_check` and a schema-version match on the copy, which is checkpointed into one self-contained file (no `-wal` sidecar)
- Restore validates the file is an AcaSmart DB before replacing the live one, behind a confirmation guard; the shell then restarts its pages in place

---

## Support & Maintenance

| Task | Frequency |
|------|-----------|
| Dependency updates (security) | Monthly |
| Python version support review | Per release |
| License key renewals | Per customer (yearly) |
| Migration testing on a real DB copy | Pre-release |
| Backup verification | Per release |

---

## Related

- [ARCHITECTURE.md](ARCHITECTURE.md) — Build outputs, module structure
- [TECHNICAL_DECISIONS.md](TECHNICAL_DECISIONS.md) — Migration strategy (ADR-0008)
- [COMMERCIAL.md](COMMERCIAL.md) — License delivery to paying customers

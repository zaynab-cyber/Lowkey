# Lowkey: Implementation Plan

## 1. Development loop (applies to every phase)

```
 branch -> write failing test -> implement -> run tests -> manual VM test
    ^                                                          |
    |                 fail: fix and repeat <-------------------+
    +-- pass: commit -> push -> CI green -> merge to main -> tag
```

Rules
1. One branch per phase: `phase/N-name`. Merge to `main` through a pull request even if solo.
2. Tests first for logic (hashing, vault, rate limit). Integration and manual tests for ACL and watchdog.
3. Never test ACL or watchdog on your office PC. Use a Windows 11 VM with snapshots.
4. Commit small and often, using Conventional Commits (`feat:`, `fix:`, `test:`, `docs:`, `chore:`, `ci:`).
5. Tag each finished phase: `v0.N.0`.

## 2. Repository setup

```
Lowkey/
  src/ Lowkey.Core  Lowkey.Service  Lowkey.UI  Lowkey.Installer
  tests/ Lowkey.Tests
  docs/ TECHNICAL_SPEC.md  SOFTWARE_SPEC.md  IMPLEMENTATION.md
  .github/workflows/ci.yml
  README.md  LICENSE  .gitignore  Lowkey.sln
```

```bash
git init -b main
git remote add origin https://github.com/<you>/Lowkey.git
git add . && git commit -m "chore: initial repository with specs"
git push -u origin main
```
CI (`ci.yml`): windows-latest, `dotnet build`, `dotnet test`, `dotnet publish -r win-x64 --self-contained`, upload artifact.

## 3. Phases

### Phase 0: Scaffolding
- Create solution and 5 projects, add xUnit, set .NET 8, enable nullable and analyzers, add CI.
- **Test:** `dotnet build` and CI both green.
- `chore: scaffold solution and CI`

### Phase 1: Core crypto and vault
- Argon2id hasher (Konscious), salt generation, constant-time compare.
- Vault model, AES-GCM encrypt/decrypt, ECDSA sign/verify, recovery key generator.
- **Tests:** hash verifies correct/incorrect pw; same pw gives different hashes (salt); tampered vault fails signature; wrong key fails decrypt; recovery key format.
- `feat(core): argon2id password hashing` / `feat(core): encrypted signed vault` / `test(core): vault tamper cases`

### Phase 2: Profile discovery
- Locate Chrome User Data dir (per-user and custom paths), parse `Local State` for profile names.
- **Tests:** fixture `Local State` JSON files (0, 1, many profiles, missing file). Manual: matches Chrome's own list.
- `feat(core): chrome profile scanner`

### Phase 3: Service skeleton and IPC
- Worker Service as Windows Service, named pipe server with caller SID check, ListProfiles command, UI client stub.
- **Tests:** IPC round-trip integration tests, malformed message rejection. Manual: install with `sc create`, start/stop.
- `feat(service): windows service with named pipe` / `feat(ui): minimal client`

### Phase 4: ACL lock engine (primary lock)
- Apply/remove Deny ACE for the user SID, verify state, persisted desired state, startup reconciliation.
- **Tests (VM):** locked profile cannot be read by the user; Chrome with that profile fails to load it; unlock restores; reboot/kill service leaves it locked; other profiles unaffected.
- Edge cases to test: open handles on re-lock, long paths, Chrome running during lock.
- `feat(service): ACL lock engine` / `test(service): ACL integration on VM`

### Phase 5: Password flows and rate limiting
- Enable, Unlock, Change, Disable, lockout back-off, master password setup, recovery key reset.
- **Tests:** every flow success and failure; lockout timing with injectable clock; no plaintext in logs.
- `feat(service): password lifecycle` / `feat(service): exponential lockout`

### Phase 6: UI
- WPF screens: first-run wizard, profile list, enable/change/disable dialogs, unlock prompt window (topmost, used by watchdog), recovery key screen.
- **Tests:** manual script covering UC-1 to UC-5; UI never receives hashes.
- `feat(ui): profile list and dialogs` / `feat(ui): first-run wizard`

### Phase 7: Session management and watchdog
- Chrome process start trace, suspend/resume, window monitor for in-process profile switch, re-lock on close, idle timeout, logoff.
- **Tests (VM):** launch via shortcut, `chrome.exe --profile-directory`, profile switcher, "Add person", guest mode. Measure bypass rate and flash time.
- `feat(service): process watchdog` / `feat(service): relock on close, idle, logoff`

### Phase 8: Installer and release
- WiX MSI, service registration with restart policy and DACL, Start Menu entry, upgrade-safe vault, master-password-protected uninstall.
- **Tests:** clean VM install, upgrade 0.1 to 0.2, wrong-password uninstall refused, repair, Windows Search finds app.
- Publish: GitHub Release with MSI and SHA-256 file. Optional: SignPath Foundation application.
- `feat(installer): msi with service` / `feat(installer): guarded uninstall` / `chore(release): v1.0.0`

### Phase 9 (optional): Encryption at rest
- AES-GCM encrypt profile folder on lock, decrypt on unlock, crash-safe journal.
- **Tests:** crash mid-unlock, large profile timing, corruption recovery.
- `feat(core): encrypted profile container`

## 4. Test matrix (run before each release tag)

| Scenario | Expected |
|---|---|
| Launch locked profile by shortcut | Blocked, prompt shown |
| `chrome.exe --profile-directory=...` | Blocked |
| Switch profile in running Chrome | Window closed or blocked |
| Wrong password x10 | Escalating lockout |
| Kill service, reboot | Still locked |
| Edit vault.dat | All locked (fail closed) |
| Chrome update | Profiles still detected and lockable |
| Uninstall, wrong password | Refused |
| Admin takes ownership | Bypass (documented limitation) |

## 5. Suggested first commits (copy and run)

```bash
git checkout -b phase/0-scaffold
dotnet new sln -n Lowkey
git commit -am "chore: scaffold solution and CI"
git checkout main && git merge --no-ff phase/0-scaffold && git tag v0.0.1
git push origin main --tags
```

## 6. Done criteria for v1.0
All Phase 0 to 8 tests pass on a clean Windows 10 and Windows 11 VM, the SRS acceptance criteria (section 7) are met, and the known limitations are stated in the README.

# Lowkey: Technical Specification
Version 0.1 | Platform: Windows 10/11 x64 | Stack: .NET 8 (self-contained), C#

## 1. Architecture

```
 [User] -> Lowkey.UI (WPF, user context)
              | Named pipe \\.\pipe\Lowkey (DACL: Users, per-message auth)
              v
        Lowkey.Service (Windows Service, LocalSystem)
          |- VaultStore     (config, hashes, recovery key)
          |- AclEngine      (apply/remove Deny ACE on profile folders)
          |- ProfileScanner (parses Chrome "Local State")
          |- Watchdog       (process + window monitor)
          |- SessionManager (unlock timers, re-lock rules)
          v
   %ProgramData%\Lowkey\vault.dat (+ vault.sig)
   %LOCALAPPDATA%\Google\Chrome\User Data\<Profile N>
```

| Project | Role |
|---|---|
| Lowkey.Core | Crypto, models, IPC contracts (shared) |
| Lowkey.Service | All privileged work |
| Lowkey.UI | Thin WPF client, no privileges |
| Lowkey.Installer | WiX v4 MSI + bootstrapper |
| Lowkey.Tests | xUnit unit + integration tests |

## 2. Lock mechanism

### 2.1 ACL lock (primary)
- Locked profile folder gets a **Deny** ACE for the interactive user's SID: `ListDirectory | ReadData | ReadAttributes | Traverse`, inherited to children. SYSTEM is untouched so the service keeps access.
- Unlock: remove the Deny ACE after password verification. Re-lock: re-apply it.
- **Ownership matters:** a folder's owner can always rewrite its permissions, so a Deny ACE alone is bypassable by the user (e.g. `icacls`). While locked, the service also sets the folder **owner to SYSTEM** (and disables inheritance), then restores ownership on unlock. This must be covered by a Phase 4 test: the normal user must fail to remove the Deny ACE.
- The User Data root is never denied, only individual profile folders.
- A Deny ACE does not close handles already open. Re-lock therefore happens only after the profile's Chrome window is closed (see 2.3), and always on logoff, service start, and timeout.
- Desired state is persisted. On service start, every protected profile is forced to locked unless an active session exists.

### 2.2 Launcher
Start Menu shortcut opens the Lowkey UI. After unlock, the UI starts `chrome.exe --profile-directory="<dir>"`.

### 2.3 Watchdog (backup)
- Process start events via WMI `Win32_ProcessStartTrace` (ETW later). Parse `--profile-directory` from the command line (default profile when absent).
- If the target profile is locked: suspend the process (NtSuspendProcess), trigger the UI password prompt, then resume or terminate.
- Already-running Chrome: poll top-level `Chrome_WidgetWin_1` windows. Titles include the profile name. A window for a locked profile is minimized and a prompt is shown.
- Chrome-close detection: when no window for a profile remains for N seconds, start the re-lock timer.

## 3. Cryptography

| Purpose | Choice |
|---|---|
| Password hash | Argon2id (m=64 MiB, t=3, p=1), 16-byte random salt per profile, 32-byte output |
| Compare | Constant-time |
| Vault encryption | AES-256-GCM, key from master password via Argon2id |
| Machine binding (optional) | ECC P-256 key in TPM (CNG "Microsoft Platform Crypto Provider") wraps the vault key; software-key fallback if no TPM |
| Integrity | ECDSA P-256 signature over vault; invalid signature = fail closed (all protected profiles locked) |
| Recovery key | 160-bit random, shown once as base32 groups, only its Argon2id hash is stored |

ECC is used for wrapping and signing, never as the password check.

## 4. Data model (vault.dat, JSON before encryption)

```json
{
  "schema": 1,
  "master": { "salt": "...", "hash": "..." },
  "recovery": { "salt": "...", "hash": "..." },
  "profiles": [
    { "dir": "Profile 2", "name": "Work", "protected": true,
      "salt": "...", "hash": "...",
      "failCount": 0, "lockedUntilUtc": null }
  ],
  "settings": { "relockMinutes": 15, "relockOnLogoff": true }
}
```
`%ProgramData%\Lowkey` ACL: SYSTEM and Administrators full control, Users read on nothing (the UI never reads it directly).

## 5. IPC protocol
Length-prefixed JSON over a named pipe. The service verifies the caller's SID via `GetNamedPipeClientProcessId` and impersonation.

| Command | Auth | Effect |
|---|---|---|
| ListProfiles | none | Names + protected/locked flags only |
| Unlock(dir, pw) | profile pw | Remove Deny ACE, start session |
| Lock(dir) | none | Re-apply Deny ACE |
| EnableProtection(dir, newPw) | master pw (first time) | Create hash, lock |
| ChangePassword(dir, old, new) | old pw | Replace hash |
| DisableProtection(dir, pw) | profile pw | Remove protection |
| RecoverWithKey(dir, key, newPw) | recovery key | Reset a forgotten profile pw |

Rate limiting: exponential delay after 3 failures (30 s, 1 m, 5 m ... cap 1 h), stored in the vault.

## 6. Installer
- WiX MSI, per-machine, requires elevation. Self-contained publish (`win-x64`), no runtime prerequisites.
- Actions: copy files, create `%ProgramData%\Lowkey`, register service (auto start, recovery: restart on failure), service DACL denying Stop to non-admins, Start Menu shortcut, first-run wizard (master password + recovery key).
- Upgrade preserves vault. Uninstall custom action prompts for the master password (or recovery key) and aborts if wrong.

## 7. Threat model

| Threat | Covered |
|---|---|
| Coworker opens Chrome/profile via shortcut, chrome.exe, switcher | Yes (ACL + watchdog) |
| Standard-user attacker edits vault or stops service | Yes |
| Wrong-password brute force | Yes (Argon2id + lockout) |
| Local admin (take ownership, Safe Mode, stop service) | **No** |
| Copying profile files offline / second OS | **No** (needs future Phase 9 encryption) |
| Malware running as the user | Partial |

## 8. Known risks
Chrome updates changing profile layout; EDR/AV flagging the service (submit false-positive reports, sign when possible); office IT policy; Chrome's own "Local State" format changes.

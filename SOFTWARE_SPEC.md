# Lowkey: Software Requirements Specification (SRS)
Version 0.1

## 1. Purpose and scope
Lowkey prevents other people from opening a protected Google Chrome profile on a shared Windows PC where OS-level user separation or passwords are not available. It is a deterrent against casual access, not protection against a local administrator.

## 2. Users
- **Owner:** installs the app, sets master password, protects profiles.
- **Other person at the PC:** tries to open a protected profile and must be blocked.

## 3. Assumptions and constraints
- Windows 10/11 x64, Google Chrome installed, admin rights available at install time only.
- Works offline. Settings are per PC and are not synced.
- No external dependencies for the end user (self-contained install).

## 4. Functional requirements

| ID | Requirement |
|---|---|
| FR-01 | The installer shall install the app and background service in one wizard with no manual dependency setup. |
| FR-02 | On first run the app shall require the owner to set a master password and shall display a one-time recovery key. |
| FR-03 | The app shall list all Chrome profiles (name and folder) and shall detect newly created profiles. |
| FR-04 | The owner shall be able to enable password protection on a profile by setting a profile password. |
| FR-05 | A protected profile shall not open until the correct password is entered, regardless of launch method. |
| FR-06 | A password prompt shall appear when a protected profile is launched or switched to. |
| FR-07 | The owner shall be able to change a profile password only by entering the current password. |
| FR-08 | The owner shall be able to disable protection on a profile only by entering that profile's password. |
| FR-09 | The system shall provide no "remove" or "forgot password" path in the UI other than the recovery key. |
| FR-10 | A forgotten profile password shall be recoverable only with the recovery key or by uninstalling with the master password. |
| FR-11 | Uninstallation shall require the master password (or recovery key). |
| FR-12 | A protected profile shall re-lock after Chrome closes it, after a configurable idle timeout, on Windows logoff, and on service start. |
| FR-13 | Repeated wrong passwords shall trigger an increasing delay before the next attempt. |
| FR-14 | The app shall be launchable from Windows Search through its Start Menu entry. |
| FR-15 | If the vault is missing, corrupted or tampered with, protected profiles shall remain locked (fail closed). |
| FR-16 | The app shall support multiple profiles, each with its own password. |

## 5. Non-functional requirements

| ID | Requirement |
|---|---|
| NFR-01 | Unlock (password check + ACL change) shall complete in under 2 s. |
| NFR-02 | Idle service shall use under 50 MB RAM and under 1% CPU. |
| NFR-03 | Passwords shall never be stored or logged. Only salted Argon2id hashes are kept. |
| NFR-04 | The service shall restart automatically on failure and shall not be stoppable by non-admins. |
| NFR-05 | The UI shall run without elevation. |
| NFR-06 | A crash or power loss shall not leave a profile permanently unlocked beyond the next service start. |
| NFR-07 | The installer shall support in-place upgrade without losing the vault. |
| NFR-08 | The app shall be usable with only basic computer skills (three clicks to protect a profile). |

## 6. Use cases

**UC-1 Protect a profile.** Owner opens Lowkey, enters master password, selects a profile, chooses Enable, sets and confirms a password. Result: profile is locked.

**UC-2 Open a protected profile.** Owner selects the profile, enters its password, Chrome opens that profile. When it is closed, the profile re-locks.

**UC-3 Intruder tries to open it.** Intruder double-clicks Chrome or switches profile. Chrome cannot read the profile, or the watchdog blocks it and shows the prompt. Without the password, nothing opens.

**UC-4 Change or disable.** Owner selects profile, chooses Change or Disable, enters the current password. Success updates or removes protection.

**UC-5 Forgotten password.** Owner enters the recovery key, sets a new password.

## 7. Acceptance criteria
1. With a profile protected, launching via shortcut, `chrome.exe --profile-directory`, and the in-browser switcher all fail to show profile data without the password.
2. Ten wrong passwords result in escalating lockout and no unlock.
3. Editing `vault.dat` by hand leaves all profiles locked.
4. Uninstall with a wrong master password is refused.
5. Fresh install on a clean Windows VM works with no pre-installed runtime.
6. Reboot or kill of the service leaves protected profiles locked.

## 8. Out of scope (v1)
Other browsers, macOS/Linux, cloud sync, defense against local administrators, offline profile encryption (planned Phase 9).

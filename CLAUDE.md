# Lowkey: instructions for Claude Code

Lowkey is a Windows app that password-protects Chrome profiles (ACL lock + watchdog + WPF launcher).
Read first, in order: docs/SOFTWARE_SPEC.md, docs/TECHNICAL_SPEC.md, docs/IMPLEMENTATION.md.

## Rules
- Work **one phase at a time** from IMPLEMENTATION.md. Stop after each phase and report test results.
- Branch per phase (`phase/N-name`), Conventional Commits, tests before logic, never skip failing tests.
- Stack: .NET 8, C#, WPF, xUnit, WiX v4, Konscious.Security.Cryptography (Argon2id).
- Never log or store plaintext passwords. Fail closed on any vault error.
- ACL/watchdog code must not run in unit tests against real user folders. Use temp folders, and put real-ACL tests behind `[Trait("Category","VM")]` so they run only in a VM.
- Do not add features outside the specs without asking.
- Logo and branding live in /assets. Use `assets/logo.png` for the app icon (convert to .ico in Phase 6).

## Start with
Phase 0 (scaffold + CI), then Phase 1 (Core crypto + vault).

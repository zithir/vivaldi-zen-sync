# vivaldi-zen-sync

A local macOS bridge that keeps **bookmarks** and **browsing history** consistent between **Zen** (Firefox-based) and **Vivaldi** (Chromium), with Vivaldi Sync carrying the data to and from Android.

**Status: specification only — no implementation yet.** Read [SPEC.md](SPEC.md) for the problem statement, scope, and technical challenges.

## Why this exists

- Zen on the desktop, Vivaldi on Android (password-manager autofill works reliably there).
- Vivaldi Sync already bridges Android ↔ desktop Vivaldi, end-to-end encrypted, including full history.
- Nothing on earth bridges Vivaldi ↔ Zen on the desktop. This project is that bridge — writing to the browsers' local profile data, in the background, without manual steps.

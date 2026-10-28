# Handoff to GitHub Copilot: GNOME Wayland shortcuts and quickpaste

**Document:** `HANDOFF_GNOME_WAYLAND_SHORTCUTS.md`  
**Purpose:** Preserve a proven keyboard-shortcut setup, capture its exact current configuration, and turn it into a maintainable GitHub repository.  
**Version:** 1.0  
**Created:** 2026-10-28  
**Target workstation:** `Razor-ubuntu` (Ubuntu 26.04.1 LTS, GNOME on Wayland)  
**Suggested repository name:** `gnome-wayland-shortcuts`

## 1. Mission and non-negotiable rules

This system is **already working**. Start by documenting and backing up the installed files and GNOME shortcut settings. **Do not replace, rewrite, move, delete, reinstall, or rebind anything on the live workstation** without first demonstrating that a staged replacement works and asking for approval. Work in a separate local repository. Do not rerun the Locate Search installer; it might overwrite GNOME shortcut settings.

The key discovery: GNOME can launch programs instantly through custom shortcuts, but text insertion from separate shell scripts was unreliable. A centralized Python utility that serializes clipboard/paste operations with `fcntl.flock`, verifies the new clipboard content with `wl-paste`, and sends one paste using `ydotool` has been confirmed reliable, including when alternating the timestamp and email shortcuts. **Retain this behavior.**

Every script or source-code file created or modified MUST begin with a comment header containing **filename, purpose, version, and creation/update date**, consistent with the user's convention.

## 2. Current shortcut inventory

| Shortcut | Function | Current command or expected command | Confidence / notes |
|---|---|---|---|
| **Ctrl+Shift+Alt+D** | Paste timestamp at cursor, format `YYYY-MM-DD-HH-MM-SS` | `/usr/bin/python3 /home/johan/.local/bin/quickpaste.py date` | Working reliably with shared Python script. Example: `2026-10-24-21-31-48`. |
| **Ctrl+Shift+Alt+J** | Paste `johan@aanscharius.com` at cursor | `/usr/bin/python3 /home/johan/.local/bin/quickpaste.py email` | Working reliably with shared Python script. Treat the address as user-specific configuration when preparing a public repo. |
| **Ctrl+Shift+Alt+F** | Open **Locate Search**, an Everything-like file search GUI | Likely `/usr/bin/python3 /home/johan/github/repos/Everything_like/locate-search.py` | Search app works. **Read live GNOME configuration to verify actual binding and command/path.** Earlier the command wrongly pointed to `/run/media/johan/DATA/github/repos/Everything_like/locate-search.py`, which did not exist. This was to be moved to the local home disk. |
| **Ctrl+Shift+Alt+P** | Interactive screenshot and annotation in **Gradia** | `flatpak run be.alexandervanhee.gradia --screenshot=INTERACTIVE` | User confirmed it works and supplied a screenshot. |

**Important:** The table preserves the shortcut assignments established in the conversation. The GNOME settings database is the final source of truth. In particular, confirm the actual binding and currently configured executable path for Locate Search; do not infer it from the old ZIP or the earlier broken mount path.

### Distinct workflows

- **D and J:** Insert text in the currently focused application, without an extra manual `Ctrl+V`.
- **F:** Launch a separate existing Python/GUI project (`Everything_like`); do not fold that project into this repo without explicit approval.
- **P:** Launch an existing Gradia Flatpak interactive screenshot session; preserve how Gradia currently behaves.

## 3. Environment and dependencies

- Desktop session: GNOME Wayland (`echo "$XDG_CURRENT_DESKTOP"` and `echo "$XDG_SESSION_TYPE"`).
- Python executable used for the working shortcut: `/usr/bin/python3`.
- Working text insertion tool: `/home/johan/.local/bin/quickpaste.py`.
- Clipboard utilities: `wl-copy`, `wl-paste` from `wl-clipboard`.
- Virtual keyboard input: `ydotool` plus the user daemon/service `ydotool.service` (`ydotoold`).
- The user verified `systemctl --user is-active ydotool.service` returned `active`, and `ydotool` was at `/usr/bin/ydotool`.
- Screenshot tool: Gradia Flatpak app ID `be.alexandervanhee.gradia`.
- External file search project: `Everything_like`, executed as `locate-search.py`.
- Python quickpaste currently needs only standard-library modules: `sys`, `time`, `fcntl`, `subprocess`, `datetime`, `pathlib`. No `pip` dependencies are needed for the core tool.

Avoid tools requiring X11 global input (traditional AutoHotkey or standard AutoKey). GNOME Wayland limits synthetic input; the proven `wl-clipboard` + `ydotool` method is deliberate.

## 4. Working quickpaste implementation (reference from conversation)

The code below records the last Python implementation supplied to the user and subsequently confirmed to work consistently. **First compare it to `/home/johan/.local/bin/quickpaste.py` on the actual machine; the file on disk takes precedence.** Keep an exact backup before any modifications. The header shown below is added for repository documentation; do not treat this reference as a reason to overwrite the live script.

```python
#!/usr/bin/env python3
# File: quickpaste.py
# Purpose: Serialize GNOME Wayland shortcut text insertion using clipboard verification and ydotool.
# Version: 1.0 (working-reference snapshot)
# Created/updated: 2026-10-28 (documentation header; underlying script confirmed working 2026-10-24)

import sys
import time
import fcntl
import subprocess
from datetime import datetime
from pathlib import Path

COMMANDS = {
    "date": lambda: datetime.now().strftime("%Y-%m-%d-%H-%M-%S"),
    "email": lambda: "johan@aanscharius.com",
}

def main():
    if len(sys.argv) != 2 or sys.argv[1] not in COMMANDS:
        print("Usage: quickpaste.py date|email")
        return 1

    # One paste operation at a time
    lock_path = Path.home() / ".cache" / "quickpaste.lock"
    lock_path.parent.mkdir(parents=True, exist_ok=True)

    with open(lock_path, "w") as lock:
        fcntl.flock(lock, fcntl.LOCK_EX)

        # Allow shortcut modifiers to be released
        time.sleep(0.25)

        text = COMMANDS[sys.argv[1]]()

        # Set clipboard
        subprocess.run(["wl-copy", "--", text], check=True)

        # Wait until clipboard contains our text
        for _ in range(15):
            result = subprocess.run(
                ["wl-paste", "-n"],
                capture_output=True,
                text=True,
                timeout=1,
            )
            if result.returncode == 0 and result.stdout == text:
                break
            time.sleep(0.04)
        else:
            print("Clipboard not ready", file=sys.stderr)
            return 1

        # Paste once using Ctrl+V
        subprocess.run(
            ["ydotool", "key", "29:1", "47:1", "47:0", "29:0"],
            check=True,
        )

        time.sleep(0.1)

    return 0

if __name__ == "__main__":
    sys.exit(main())
```

Why it matters:

1. `fcntl.flock` ensures that two simultaneous shortcut invocations do not race over the same clipboard.
2. The short delay gives the physical Ctrl/Shift/Alt keys time to release.
3. `wl-copy` sets the requested string; `wl-paste -n` **checks** it before typing the paste shortcut.
4. `ydotool key 29:1 47:1 47:0 29:0` emits one Ctrl+V (Linux input key codes).
5. The verified shared Python approach solved intermittent failures encountered with independent Bash scripts for D and J.

Limitations: Clipboard contents are replaced; this is the observed current behavior. There may be timing or focus issues outside the two tested shortcuts. Terminal emulators often expect Ctrl+Shift+V, not Ctrl+V. Do not claim these issues are fixed unless tested. Avoid speculative optimizations or shortening delays without measured evidence.

## 5. Legacy scripts and issues to preserve as history

- Earlier scripts at `~/.local/bin/timestamp.sh` and `~/.local/bin/email.sh` independently used `sleep`, `wl-copy`, and `ydotool`.
- The separate scripts sometimes delayed insertion, failed to paste, or inserted twice; switching rapidly between J and D was especially unreliable.
- An attempted Bash `flock` version produced a period when no timestamp appeared; later variants worked intermittently. Do not revive those scripts as the default implementation.
- Locate Search initially pointed to `/run/media/johan/DATA/github/repos/Everything_like/locate-search.py`, which returned `No such file or directory`. Moving the project to a local home-directory repo was proposed. Verify where it actually lives now.
- Gradia is installed via Flatpak. Flathub was configured after an initial `No remote refs found for 'flathub'` installation error. It was then demonstrated working.

## 6. Required first phase: read-only inventory and backup

1. Inspect (do not change) the live Python script; compare it to the reference in section 4. Capture its SHA-256 and file permissions.
2. Read the GNOME custom shortcut path list:

   ```bash
   gsettings get org.gnome.settings-daemon.plugins.media-keys custom-keybindings
   ```

3. For **each** listed custom shortcut path, read these keys from schema `org.gnome.settings-daemon.plugins.media-keys.custom-keybinding` at that path: `name`, `command`, `binding`. GNOME path names can differ from expected labels; do not assume a path like `locate-search/` exists.
4. Export the complete, actual entries in a machine-readable file, plus a human-readable Markdown table. Redact user-private fields in files intended for a public repo; retain a separate local ignored backup if necessary.
5. Back up installed scripts and GNOME shortcuts to a timestamped directory under the user's home folder, without deleting or altering the originals. Preserve file mode.
6. Check `command -v python3 wl-copy wl-paste ydotool`, user service status, Flatpak installation, and the active GNOME Wayland session.
7. Confirm `Locate Search` and `Gradia` paths by looking at the **live** GNOME configuration, not assumptions.

A `gsettings` dump is useful for reference but is **not** a reason to overwrite the current configuration. Avoid applying an unreviewed `dconf load` or broadly resetting GNOME shortcuts.

## 7. Suggested repo structure

```text
gnome-wayland-shortcuts/
  README.md                      # Use cases, current hotkeys, installation, caveats
  HANDOFF_GNOME_WAYLAND_SHORTCUTS.md
  .github/
    copilot-instructions.md      # Keep existing behavior and header conventions
  src/
    quickpaste.py                # Verified copy of the working implementation
  scripts/
    export-shortcuts.sh         # Read-only GNOME shortcut export
    install.sh                  # Safe, opt-in and idempotent, backup-first
    verify.sh                   # Read-only preflight and smoke-test guidance
  tests/
    test_quickpaste.py           # Unit tests with mocked subprocess/time/locking as needed
  config/
    quickpaste.example.toml     # Optional user-specific config example (no personal data)
  .gitignore                     # Secrets/local config, runtime locks, backups, caches
  LICENSE                       # Only if user selects one
```

Repository structure is a proposal, not a mandate to refactor the known-good working file. Keep Locate Search as its own repo and Gradia as an external Flatpak dependency.

## 8. Implementation sequence for Copilot

**Phase A - Snapshot:** Create a repo workspace and copy the exact working `quickpaste.py` into it. Export actual GNOME bindings. Document external app paths and package versions. Make no live changes.

**Phase B - Documentation:** Produce a clear `README.md` documenting GNOME Wayland, prerequisites, all four shortcuts, setup, troubleshooting, and the clipboard overwrite behavior. Include screenshots only if provided by the user. Do not imply other machines have identical paths or shortcut settings.

**Phase C - Maintainability:** Add isolated tests. If making email configurable (recommended for a public repo), keep the live behavior and offer a local ignored config such as `~/.config/quickpaste/config.toml`, with a documented migration plan. Do not silently change the live working command or commit sensitive local configurations. Include a copyable user-specific example only in private backups.

**Phase D - Safe installation:** Optional installer should (a) check prerequisites, (b) show exactly what will change, (c) make timestamped backups, (d) use explicit consent before changing each live file or GNOME shortcut, (e) be repeatable, and (f) support rollback. Do not run `sudo` or overwrite the user's working apps simply to assemble the repo.

**Phase E - Acceptance tests, after consent:**

- D inserts the timestamp once, with correct `YYYY-MM-DD-HH-MM-SS` formatting.
- J inserts the configured email exactly once.
- Alternating J -> D -> J -> D repeatedly works without missing/duplicated text.
- Rapid sequential use is serialized; no unexpected reordering or duplicate paste.
- F still launches the Everything-like Locate Search GUI.
- P still starts Gradia interactive screenshot capture.
- Everything continues to work after log-out/log-in or reboot when services and mounts are available.
- GNOME input focus and key release do not cause sporadic pasted modifiers.
- The README explicitly notes Ctrl+V limitations in terminal applications and clipboard replacement.

Use an ordinary text editor and VS Code for text-insertion testing. Do not paste test strings into commands, secrets, or important documents.

## 9. Privacy, quality, and versioning rules

- The actual email string appears in this handoff because it is a requested current shortcut. **Review before publishing to a public GitHub repository.** Prefer keeping user-specific text in a local config excluded by `.gitignore`.
- Do not log copied clipboard contents or store text in telemetry.
- Do not add unnecessary Python libraries, background services, or desktop extensions.
- Never hardcode `/home/johan` into a general-purpose installer; use `$HOME` or `Path.home()`. Preserve the exact current paths in the *inventory* as evidence.
- Avoid broad GNOME shortcut resets, Flatpak reinstalls, or system-level service changes unless there is a measured need.
- Use clear error reporting and document how to inspect `ydotool.service`.
- Every authored/modified source or script starts with the required header: filename, purpose, version, created/updated date.
- Commit a baseline snapshot **before** experimenting. Changes to tested behavior require a reason, a test, and an explicit rollback path.

## 10. First message / task for GitHub Copilot

> Read `HANDOFF_GNOME_WAYLAND_SHORTCUTS.md` completely. We already have a working GNOME Wayland setup. First, inventory the actual `~/.local/bin/quickpaste.py` and the GNOME custom shortcuts **read-only**, then produce a backup and a proposed repository tree. Verify the four shortcuts and the live Locate Search path. Preserve the current Python clipboard-locking/verification solution as the baseline. Build docs and tests in the repo, but **do not alter active GNOME shortcuts, installed scripts, or services without my approval**. Put filename/purpose/version/date comment headers at the top of each script you create. Report any difference between this handoff and the live machine before proceeding.

---

**Expected result:** A version-controlled, reproducible reference for the user's current GNOME Wayland shortcuts that keeps the proven installation working, rather than a premature rewrite.

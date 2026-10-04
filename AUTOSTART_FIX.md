# AUTOSTART_FIX.md — "Start at Login" investigation & fix plan

**Symptom reported:** *juhradial is not autostarting even when toggling autostart in
the juhradial settings.*

**Investigated on:** Fedora 44 (GNOME Wayland, systemd user session), install layout
`/usr/local/bin/juhradiald` + `/usr/local/sbin/juhradial-mx` + `/usr/share/juhradial/`,
dotfiles-managed `~/.config/autostart/` and `~/.config/systemd/user/` (GNU stow
symlinks into a git repo).

---

## TL;DR

Autostart itself **was working at every login** — journalctl proves both the systemd
unit and the XDG `.desktop` entry fired in every inspected boot. Two separate things
produced the "not autostarting" impression:

1. **A device-attach dead window** (the perceived failure): the MX Master 4 connects
   over Bluetooth and can take seconds to appear after the user starts using the
   machine; until the daemon's hotplug/poll loop attaches and diverts the gesture
   button, the menu is dead → "it didn't autostart".
2. **Real bugs in the settings toggle** (why toggling seemed to do nothing):
   `start_at_login` is never persisted to `config.json`, the switch state ignores
   the autostart file (the actual source of truth), and the write/remove paths are
   hostile to symlinked (dotfiles-managed) autostart entries.

Root-cause fixes are listed in [Proposed fixes](#proposed-fixes).

---

## How autostart is supposed to work

There are **two independent mechanisms**, both intentional (see `packaging/` and the
unit header comment):

| Mechanism | File | Starts | Notes |
| --- | --- | --- | --- |
| systemd user unit | `~/.config/systemd/user/juhradialmx-daemon.service` (enabled in `graphical-session.target.wants/` + `default.target.wants/`) | `juhradiald` only | Ordering-only relationship to `graphical-session.target` (avoids the issue #67 login loop) |
| XDG autostart | `~/.config/autostart/juhradial-mx.desktop` | `scripts/juhradial-mx.sh` launcher → overlay + (fallback) daemon | Launcher defers to systemd when the unit is enabled/active (`daemon_unit_managed`) |

The launcher deliberately **does not** start a second daemon when the systemd unit
owns it (issue #60), so on a systemd setup the `.desktop` entry effectively exists
to start the **overlay** (tray icon + radial menu window) and the systemd unit owns
the **daemon**. The settings toggle only manages the `.desktop` file.

---

## Evidence (from the affected machine)

### 1. Autostart fired in every inspected boot

```
Aug 26 18:27:17 systemd: Started juhradialmx-daemon.service            ← unit
Aug 26 18:27:19 systemd: Started app-gnome-juhradial\x2dmx-4530.scope  ← XDG .desktop
Aug 26 18:27:21 overlay: System tray icon active
```

Same pattern in all prior boots (`journalctl --user -b -N | grep app-gnome-juhradial`).
Conclusion: **the autostart entry and the unit are not the problem.**

### 2. The actual failure the user saw (morning after a 14h session)

The MX Master 4 was never connected during the session booted Aug 26 18:27 — the
daemon logged `Logitech hidraw device not found` every 5s all evening. Next morning:

```
08:39:32  Device hotplug detected: Create(File)            ← mice (re)appear
08:39:32  ERROR Permission denied opening "/dev/hidraw0"   ← see §3, misleading
08:39:39  Connected to MX Master 4 via hidraw (Bluetooth)  ← attaches after ~7s
08:39:39  Gesture buttons diverted (count=2)
08:39:44  Started app-gnome-juhradial\x2dmx-77361.scope - Application launched by gnome-shell
```

The user hit the dead window (mouse present, daemon still attaching), concluded
autostart had failed, and manually launched `juhradial-mx`. The launcher found the
daemon (systemd) and overlay already running, printed `JuhRadial MX started` to an
invisible stdout, and no-op'd — **zero visible feedback**, reinforcing the belief
that nothing had autostarted.

### 3. Misleading hidraw permission errors (red herring)

At hotplug time `/dev/hidraw0` is briefly/wrongly probed and belongs to a non-Logitech
device (here: a Corne keyboard, `1D50:615E`, `root:root 0600`). The daemon's hidraw
fallback scan logs "Found Logitech hidraw device (fallback)" for it and then emits
the issue #52 remediation text ("add yourself to the `input` group…"), even though
the user **is** in `input` and the real Logitech node opens fine moments later.
`99-juhradialmx.rules` only matches `046d` (plus the `0005:046D:*` uhid pattern), so
non-Logitech nodes legitimately stay 0600 — the *error classification* is the bug,
not the permissions.

---

## Root-cause bugs in the settings toggle

All in `overlay/settings_page_settings.py` (identical in the installed
`/usr/share/juhradial/` copy).

### Bug A — `start_at_login` is never persisted

```python
def _on_startup_changed(self, switch, state):
    config.set("app", "start_at_login", state)        # auto_save defaults to False!
    if state:
        self._write_autostart(self._resolve_launcher_path())
    else:
        ...autostart_file.unlink()
```

`ConfigManager.set()` only updates the in-memory dict unless `auto_save=True`
(`overlay/settings_config.py`). Unlike the theme/DE handlers, this handler never
calls `config.save()`. Consequence: the `.desktop` file is written correctly, but
the next time Settings opens the switch re-reads the **stale** `config.json` — on a
fresh install (default `start_at_login: true`, no file yet) or after any external
restore, the switch shows the wrong state and appears to "not stick", inviting
repeated re-toggling.

### Bug B — switch state ignores the source of truth

The autostart *file* is what GNOME session reads; the switch reads `config.json`.
The two can diverge (manual delete/restore, dotfiles sync, reset via "Restore
Defaults" which resets `config.json` but leaves the file). The UI must treat the
file as the source of truth for display.

### Bug C — symlink-hostile write/remove (dotfiles-managed setups)

With `~/.config/autostart/juhradial-mx.desktop` symlinked into a stow-managed repo:

- **Toggle ON:** `Path.write_text()` writes *through* the symlink, mutating the
  dotfiles git repo (uncommitted churn; content divergence).
- **Toggle OFF:** `unlink()` removes only the symlink — the repo file survives and
  the next `stow` re-creates the link → **autostart silently resurrects** after the
  user explicitly disabled it. This alone fully explains "I turned it off/on and it
  doesn't behave".
- **OFF→ON:** `write_text` through the (now dangling or re-stowed) symlink can
  produce a regular file that conflicts with the next `stow`.

Fix: write a temp file in `~/.config/autostart/` + `os.replace()` (atomic,
replaces the symlink itself — same pattern `ConfigManager.save()` already uses),
and on OFF unlink any symlink target's *link* only (never follow it), optionally
also clearing the repo copy is out of scope — but at minimum log the situation.

### Bug D — `_repair_autostart_if_stale()` can't heal a missing entry

It returns early when the file doesn't exist and is gated on the (never-persisted,
Bug A) config flag. After the config flag is fixed, it should **create** a missing
entry when `start_at_login` is true, not merely rewrite a broken `Exec=`.

### Bug E — `_resolve_launcher_path()` can return a nonexistent path

Falls back to `candidates[0]` when nothing exists → writes `Exec=` pointing nowhere
→ status=127 at login, a recurrence of the exact failure issue #32's repair logic
was written to heal. Should never write an entry when no launcher exists; surface
an error instead.

### Bug F (UX) — silent no-op manual launcher

`scripts/juhradial-mx.sh` prints `JuhRadial MX started` even when it started nothing
because the daemon/overlay were already up. During the device-attach dead window
this makes a manual launch *feel* like a fix while providing no state information.

---

## Proposed fixes

Priority order; A–C are the user-visible correctness fixes.

### A. Persist the toggle (`settings_page_settings.py`)

```python
def _on_startup_changed(self, switch, state):
    config.set("app", "start_at_login", state, auto_save=True)
    ...
```

### B. Derive switch state from the autostart file

Active ⇔ file exists ∧ `X-GNOME-Autostart-enabled != false` ∧ `Exec` binary exists.
Keep `config.json` as a hint only (or drop the config key entirely and treat the
file as the single source of truth; see open question below).

### C. Atomic, symlink-safe file management

```python
def _write_autostart(self, exec_path):
    autostart_file = self._autostart_file()
    autostart_file.parent.mkdir(parents=True, exist_ok=True)
    tmp = autostart_file.with_suffix(".desktop.tmp")
    tmp.write_text(..., encoding="utf-8")   # never follows the target symlink
    os.replace(tmp, autostart_file)         # atomically replaces the symlink itself
```

`unlink()` on OFF is already link-safe (never follows symlinks); document that
dotfiles-managed entries should be removed at the source (stow) and detect +
warn when the path is a symlink so the user understands the survival semantics.

### D. Repair a *missing* entry, not just a stale `Exec=`

In `_repair_autostart_if_stale()`: when `start_at_login` is true and the file is
missing → write it. When no launcher candidate exists → log + toast an error and
leave the file untouched (fixes Bug E).

### E. Device-state feedback (fixes the *perceived* failure)

- Tray tooltip / menu item: `Daemon: running (systemd) · Overlay: running ·
  Mouse: waiting for connection…` — the daemon already knows all three states.
- Launcher (`scripts/juhradial-mx.sh`): when it no-ops because everything is
  already running, say so explicitly ("daemon already running via systemd; overlay
  already running; nothing to start") instead of the unconditional success banner.

### F. hidraw fallback scan: don't claim non-Logitech nodes

`daemon/src/hidraw.rs`: the "Found Logitech hidraw device (fallback)" probe matches
any hidraw node; filter by Logitech VID (`046d`) / the uhid `0005:046D:*` pattern
before logging/attempting open, and downgrade the issue #52 remediation text so it
only fires for nodes the daemon actually needs.

---

## Verification checklist (after the fix)

1. Fresh config (no `config.json`, no `.desktop`): open Settings → toggle shows ON
   (file present) → close/reopen → still ON. Toggle OFF → file gone → reopen → OFF.
2. Toggle ON, `rm ~/.config/autostart/juhradial-mx.desktop`, reopen Settings →
   entry recreated (repair path D).
3. With the autostart path as a symlink into a dotfiles repo: toggling ON must
   replace the symlink with a regular file (repo untouched); toggling OFF removes
   the link only, and Settings warns that the repo copy remains.
4. Reboot/login: overlay scope starts (`journalctl --user -b | grep
   app-gnome-juhradial`), daemon starts via the unit, tray icon present.
5. Simulate the dead window (Bluetooth mouse off at login, on later): tray shows
   "waiting for device"; after attach, gesture button works without manual launch.

## Open questions

- Should `app.start_at_login` in `config.json` be retired in favor of the file as
  the single source of truth (avoids Bug B class entirely)? The daemon doesn't read
  it; only Settings does.
- Should the manual launcher gain a `--status` mode reusing the same state report
  as the tray (E), so "did it start?" is answerable from a terminal?

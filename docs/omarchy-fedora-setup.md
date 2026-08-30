# Omarchy on Fedora Atomic (COSMIC) — setup log

Running record of what's been done to get Omarchy's Hyprland config, Quickshell, and themes
working on this machine (Fedora 44 COSMIC Atomic, `rpm-ostree`-based). 

Update this file as we go — append new fixes under "Issues found & fixed", don't rewrite history.

## Repo enabled

```
/etc/yum.repos.d/lionheartp-Hyprland.repo
```
Copr: [lionheartp/Hyprland](https://copr.fedorainfracloud.org/coprs/lionheartp/Hyprland/) —
builds Hyprland ≥0.55 (Lua config) and Quickshell ≥0.3.1 for `fedora-44-x86_64`, which stock
Fedora doesn't ship.

## Packages layered (`rpm-ostree install`)

Confirmed booted, in this deployment:

| Package | Source | Why |
|---|---|---|
| `hyprland` | Copr | the compositor, ≥0.55 for Lua config support |
| `hyprland-uwsm` | Copr | the uwsm-managed session `.desktop` entry — **pick this one at the greeter**, not plain `Hyprland` |
| `uwsm` | Copr | not in Fedora's own repos; session manager Omarchy's `o.launch` wraps everything in |
| `quickshell` | Copr | ≥0.3.1, for `Quickshell.Networking`/`Quickshell.Services.Polkit` the shell imports |
| `hyprland-guiutils` | Copr | **added after the fact** — a Hyprland dialog (plugin/ABI mismatch warning) needed it, wasn't in the original layer list |
| `alacritty` | Fedora | Omarchy's default terminal (matches `config/alacritty` dotfiles) |
| `xdg-desktop-portal-hyprland` | Fedora | screen share / file picker portal for the Hyprland session |
| `xdg-terminal-exec` | Fedora | what `omarchy-launch-terminal` execs through |

## Config on disk

- `~/.omarchy` — fresh clone of `github.com/basecamp/omarchy` (upstream, unmodified). This is
  what `$OMARCHY_PATH` points at instead of the (nonexistent-on-Fedora) `/usr/share/omarchy`.
- `/etc/omarchy.conf`:
  ```
  export OMARCHY_PATH="/home/tobi/.omarchy"
  ```
  Written by hand — this is exactly what `omarchy-dev-link` would write, reusing Omarchy's own
  dev-checkout mechanism rather than inventing something new.
- `~/.config/uwsm/env.d/10-omarchy`:
  ```
  [ -r "$HOME/.omarchy/default/bash/env-bootstrap" ] && . "$HOME/.omarchy/default/bash/env-bootstrap"
  ```
  Hand-created because the real `omarchy-settings` package would normally install
  `default/bash/env-bootstrap` to a fixed system path and a matching `/usr/share/uwsm/env.d/10-omarchy`
  that just sources it from there — neither exists on Fedora, so this sources it straight from the
  checkout instead.
- `~/.config/hypr/` — copied wholesale from `~/.omarchy/config/hypr/` (this directory *is* the
  `/etc/skel` seed a real install would copy for a new user).
- `~/.local/share/fonts/omarchy.ttf` — the bar's icon glyph font, copied by hand + `fc-cache -f`
  (normally placed by `omarchy-settings`).

## Issues found & fixed

1. **`hyprland-guiutils` missing** — a Hyprland-shown dialog needed it. Not in the original layer
   list; added via `sudo rpm-ostree install hyprland-guiutils`. Confirmed installed and booted.
2. **Theme backgrounds not rendering** (theme: `lupine`) — staging was correct (real `.webp`
   files, correct `~/.local/state/omarchy/current/background` symlink pointing at a real file),
   so the break was in Quickshell/Qt actually *decoding* the image, not in Omarchy's theme
   scripts. Root cause: Qt has no built-in WebP decoder — it needs the `qt6-qtimageformats`
   plugin package, which wasn't installed. This matches Omarchy's own upstream package list
   (`install/omarchy-base.packages:111`, `qt6-imageformats`), which exists for exactly this
   reason.
   **Status: fix identified, not yet applied.** Next command to run:
   ```
   sudo rpm-ostree install qt6-qtimageformats
   ```
   then reboot, then re-check that the theme background actually renders.

3. **`SUPER + K` (keybindings picker) did nothing** while `SUPER + SPACE` (root menu) worked fine.
   Traced with `bash -x` directly against the live session: `bin/omarchy-menu-keybindings`'s own
   list-building logic worked perfectly (full, correctly-formatted keybindings list generated),
   but handing that list to the picker UI failed at `bin/omarchy-menu-select:75` — `perl: command
   not found`. Root cause: `omarchy-menu-select` uses Perl (`Encode` + `JSON::PP`) to build a
   UTF-8-safe JSON payload for the Quickshell IPC call; Arch ships this as part of its base `perl`
   install, Fedora doesn't include Perl by default at all. `SUPER + SPACE` doesn't hit this because
   the root menu doesn't go through `omarchy-menu-select`.
   **Status: fix identified, not yet applied.**
   ```
   sudo rpm-ostree install perl-interpreter perl-JSON-PP perl-Encode
   ```
   All three confirmed available in Fedora's own repos, no Copr needed.

4. **`lua` and `xkbcli` also missing** — found while tracing issue 3, though neither actually broke
   anything there (both degrade gracefully in `omarchy-menu-keybindings`: `omarchy-cmd-present lua`
   just skips the Lua-bind-cache supplement if absent, and `xkbcli`-less keycode resolution falls
   back to a small hardcoded symbol table). Adding preemptively rather than waiting for something
   else to hit them.
   ```
   sudo rpm-ostree install lua libxkbcommon-utils
   ```
   `lua` (5.4.8, Fedora's package) is a separate, standalone use from the Lua 5.5 Hyprland embeds
   internally — this is just for `omarchy-menu-keybindings` to `dofile()` your Hyprland config for
   bind-metadata extraction, so the version mismatch doesn't matter here. `libxkbcommon-utils`
   provides `xkbcli`.

5. **"App failure: command not found: udiskie" notification on login — turned out to be a
   non-issue.** `udiskie` (a udisks2 front-end that auto-mounts removable media on insert) is
   launched unconditionally by [`default/hypr/autostart.lua`](default/hypr/autostart.lua) via
   `uwsm-app -- udiskie --automount --no-notify --no-tray`. Initially assumed this needed
   layering, but checked first: `gvfs-udisks2-volume-monitor.service` is already active as a
   systemd `--user` service (part of the base Cosmic image) — those are session-bus-wide, not
   scoped to a particular compositor, so it already handles automount inside the Hyprland session
   too. **Confirmed by testing**: inserted a USB stick, it showed up in Cosmic Files without
   `udiskie` installed. Installing `udiskie` would just be a redundant second automounter — the
   only thing it'd buy is silencing the harmless startup notification.

   To silence it anyway without installing a redundant package: `default/hypr/autostart.lua`
   hardcodes the launch inside one `hl.on("hyprland.start", ...)` block with no per-item toggle,
   so there's no config-level way to skip just this one line. Instead, dropped a no-op shim ahead
   of it on `PATH`:
   ```
   ~/.local/bin/udiskie   # exit 0, chmod +x
   ```
   `~/.local/bin` is on `PATH` via `env-bootstrap`, so `command -v udiskie` now resolves to the
   shim. No root, no reboot — takes effect on next full login (autostart fires once on
   `hyprland.start`, not on `hyprctl reload`).

6. **"Update available" bar icon, clicking it does nothing useful (or errors).** Traced through
   [`shell/plugins/bar/widgets/SystemUpdate.qml`](shell/plugins/bar/widgets/SystemUpdate.qml): it
   polls [`bin/omarchy-update-available`](bin/omarchy-update-available) every 6h and on start, and
   clicking it runs the full `omarchy-update` pipeline (`pacman -Syu`, keyring refresh, etc.) via
   `omarchy-launch-floating-terminal-with-presentation omarchy-update` — meaningless on Fedora,
   nothing that command depends on exists here.

   The icon itself is showing because `omarchy-update-available` (read in full) checks two
   independent things: (1) whether `$OMARCHY_PATH` is a dev checkout behind its git upstream —
   true for us, since `~/.omarchy` is a plain clone and upstream Omarchy moves fast — and (2) a
   `pacman`-based package version check, which silently no-ops here since `pacman` doesn't exist.
   So the icon is actually reporting something real (checkout is behind upstream) via a mechanism
   that isn't ours to act on (we're not doing `omarchy update`-style pulls; see the setup-log's
   own tracking instead).

   Fix: removed the widget via the **real, supported override mechanism**, not a hack —
   `shell/shell.qml` explicitly prefers `~/.config/omarchy/shell.json` over the checkout's default
   at `$OMARCHY_PATH/config/omarchy/shell.json` when present and valid. Copied the default and
   dropped the `{"id": "omarchy.system-update"}` entry from `bar.layout.center`:
   ```
   cp ~/.omarchy/config/omarchy/shell.json ~/.config/omarchy/shell.json
   # then remove the omarchy.system-update entry from bar.layout.center
   omarchy-restart-shell
   ```
   Takes effect immediately on shell restart, no reboot needed. Any other bar widget can be
   dropped or reordered the same way — this file is the canonical, documented place to do it.

7. **"Learn Keybindings" and "Update System" notifications repeating on every login.** Root cause
   traced via `~/.local/state/omarchy/first-run.log`:
   [`bin/omarchy-provision-first-run`](bin/omarchy-provision-first-run) only writes its
   `~/.local/state/omarchy/done/first-run-user` marker if *every* step in its sequence succeeds —
   and one step, `enable-user-units.sh`, was failing every single login, so the whole sequence
   (including the welcome + Wi-Fi/update notification steps) retried from scratch each time.

   `enable-user-units.sh` runs `systemctl --user enable --now` on 6 shipped units
   ([`default/systemd/user/*.service`](default/systemd/user)); three were failing, for three
   different reasons:
   - **`omarchy-fcitx5.service`** — `fcitx5` genuinely not installed (XCompose/accent-key support
     for Wayland clients). Not something Cosmic's own session provides either way.
   - **`bt-agent.service`** — `bt-agent` (from `bluez-tools`) genuinely not installed. **Checked
     first, unlike assuming**: is this actually redundant with Cosmic's own Bluetooth agent, the
     way `udiskie` turned out to be redundant with `gvfs`? No — confirmed empirically that
     `cosmic-settings-daemon`/`cosmic-applets` aren't running at all while logged into the
     Hyprland session (`ps aux` came up empty for them), unlike `gvfs-udisks2-volume-monitor`
     which runs as a genuinely DE-agnostic systemd `--user` service regardless of session. So
     nothing else provides a Bluetooth pairing agent in this session — this one's real, not
     redundant.
   - **`omarchy-sleep-lock.service`** — different kind of problem: its `ExecStart=` hardcodes
     `/usr/bin/omarchy-system-sleep-monitor`, which only exists there on a real package install.
     Ours lives in `~/.omarchy/bin/` instead, and `/usr` is read-only on this rpm-ostree system so
     a symlink into `/usr/bin` isn't an option. **Fixed immediately, no reboot needed**, via a
     systemd user drop-in rather than touching the checkout:
     ```
     ~/.config/systemd/user/omarchy-sleep-lock.service.d/override.conf:
       [Service]
       ExecStart=
       ExecStart=/home/tobi/.omarchy/bin/omarchy-system-sleep-monitor
     ```
     Confirmed running after `systemctl --user daemon-reload && systemctl --user restart
     omarchy-sleep-lock.service`. **Any other shipped unit with the same `/usr/bin/omarchy-*`
     pattern will need the same treatment** — worth checking if new failures show the same
     "No such file or directory" shape.

   Also symlinked all 6 unit files from `default/systemd/user/*.service` into
   `~/.config/systemd/user/` (they weren't anywhere systemd's user instance would find them at
   all — normally `omarchy-settings` installs these to `/usr/lib/systemd/user/`). Symlinked, not
   copied, so `git pull` on the checkout keeps them in sync.

   **Status: fix identified for 2 of 3, `omarchy-sleep-lock` already fixed and confirmed running.**
   Still needed:
   ```
   sudo rpm-ostree install bluez-tools fcitx5
   ```
   Then reboot — `enable-user-units.sh` should succeed fully, and first-run should finally mark
   itself done, stopping the repeating notifications for good.

   Note: while testing this, manually forcing `omarchy-provision-first-run` re-ran the welcome/
   Wi-Fi-update notification steps again (expected side effect of forcing a retry, not a new bug).

8. **"Unit recovered" notification for `omarchy-crash-watch.service`** — the exact recurrence I'd
   flagged as likely in issue 7: same hardcoded `/usr/bin/omarchy-crash-watch` path, doesn't exist
   here, genuine crash-loop this time (restart counter was at 38+, `Restart=always`/`RestartSec=5`
   in the unit means it fails and restarts every 5s indefinitely; "Unit recovered" was presumably a
   Cosmic/systemd flap notification catching one of its brief up-windows). Fixed with the same
   override pattern:
   ```
   ~/.config/systemd/user/omarchy-crash-watch.service.d/override.conf:
     [Service]
     ExecStart=
     ExecStart=/home/tobi/.omarchy/bin/omarchy-crash-watch
   ```
   Confirmed stable after `daemon-reload` + `reset-failed` + `restart` (0 restarts, running
   cleanly 27s+ after).

   **Checked the other two shipped units for the same pattern** rather than waiting for each to
   surface separately: `omarchy-migrate-notify.service` and `omarchy-recover-internal-monitor.service`
   both also hardcode `/usr/bin/omarchy-*` paths, but both are `Type=oneshot` gated by a
   `ConditionPathIsDirectory`/`ConditionPathExists` check that's correctly unmet here (no
   `/usr/share/omarchy` at all; no internal-monitor-disable toggle in use) — so systemd skips them
   entirely without ever hitting the broken path. No fix needed *unless* you start using the
   internal-monitor-disable toggle later, at which point the same override treatment would apply.

   **All units with a hardcoded `/usr/bin/omarchy-*` `ExecStart` now confirmed fine, one way or
   the other**: `omarchy-sleep-lock` (fixed, issue 7) and `omarchy-crash-watch` (fixed, here) were
   the two that actually ran; `omarchy-migrate-notify`/`omarchy-recover-internal-monitor` are
   condition-gated off; `bt-agent`/`omarchy-fcitx5` point at real system binaries, not Omarchy's
   own scripts, so they were never affected by this pattern.

9. **Display panel's disable-monitor toggle silently did nothing.** Not a Fedora/setup issue at
   all — a genuine Hyprland-version compatibility bug. Traced the exact command in
   [`shell/plugins/panels/monitor/Panel.qml`](shell/plugins/panels/monitor/Panel.qml):
   `toggleDisplay()` ran `hyprctl keyword monitor <name>,disable`. Tested that literal command by
   hand — it returns `keyword can't work with non-legacy parsers. Use eval.` Hyprland's Lua-native
   config parser (which is exactly what lets us use `config/hypr/hyprland.lua` at all) dropped
   support for the old `hyprctl keyword` mechanism for this option. Confirmed the modern
   replacement from Hyprland's own Lua API stub at `/usr/share/hypr/stubs/hl.meta.lua`
   (`HL.MonitorSpec` has `output: string` and `disabled?: boolean` fields, called via
   `hl.monitor(spec)`), tested by hand (`hyprctl eval 'hl.monitor({output="eDP-1", disabled=true})'`)
   — confirmed it actually disables/re-enables the monitor, unlike the old command.

   No drop-in override exists for QML the way there is for systemd units or `shell.json`, so this
   needed a direct patch to `Panel.qml`'s `toggleDisplay()`, swapping the `hyprctl keyword monitor
   ...` call for `hyprctl eval 'hl.monitor({output=..., disabled=...})'`. Applied and reloaded via
   `omarchy-restart-shell`.

   **This is a real upstream bug, not Fedora-specific** — anyone running Omarchy against a
   sufficiently new Hyprland (Lua-native config) would hit the same failure. Worth reporting
   upstream at some point rather than treating it as a permanent local patch to carry forward.

10. **Screensaver failed.** Traced through
    [`bin/omarchy-launch-screensaver`](bin/omarchy-launch-screensaver) and
    [`bin/omarchy-screensaver`](bin/omarchy-screensaver) — three separate gaps, none of them
    Fedora packaging issues in the usual sense:
    - **`ttfx` missing** — Omarchy's own bespoke Rust tool ("Terminal text effects as a single
      binary", a parity-exact Rust port of `terminaltexteffects`) that actually renders the
      animated ASCII logo. Not in any Fedora repo (expected — it's Omarchy's own tool, normally
      built via their own OPR pipeline). Found the real upstream: **`github.com/omacom/ttfx`**
      (`omacom` is Omarchy's own org — same one behind `omarchy-iso`). No published release
      binaries, so built it from source:
      ```
      toolbox create --distro fedora --release 44 rust-build
      toolbox run -c rust-build sudo dnf install -y rust cargo git
      toolbox run -c rust-build bash -c 'git clone https://github.com/omacom/ttfx.git ~/ttfx && cd ~/ttfx && cargo build --release'
      cp ~/ttfx/target/release/ttfx ~/.local/bin/ttfx
      ```
      Fedora's own `rust`/`cargo` packages (1.98.0) built it cleanly in ~45s, no patches needed.
      Built in a dedicated toolbox rather than using the pre-existing personal `~/.cargo` rustup
      toolchain on the host (confirmed via `rpm -qa` that toolchain isn't a system package at all
      — just a per-user rustup install from February, unrelated to this work). Verified the binary
      runs correctly directly on the host too (toolbox shares `$HOME`, and toolbox/host are the
      same Fedora 44 release so the dynamic linking lines up despite the "static binary" framing
      in ttfx's own README — it's not actually statically linked, just dependency-free at the
      asset level).
    - **`socat` missing** — used by `omarchy-launch-screensaver` to open Hyprland's raw event
      socket (`.socket2.sock`) so it can detect the screensaver window opening on each monitor
      before moving focus to the next one. Genuine, generic Fedora package, not yet layered:
      ```
      sudo rpm-ostree install socat
      ```
      **Status: not yet applied, needs reboot.**
    - **`~/.config/omarchy/branding/screensaver.txt` didn't exist** — the ASCII art asset `ttfx`
      renders. Normally seeded by `omarchy-branding-screensaver reset` at install/provision time,
      which we never ran (not part of the first-run sequence, an install-time step). Seeded it
      directly: `cp ~/.omarchy/logo.txt ~/.config/omarchy/branding/screensaver.txt`. Confirmed
      `ttfx` renders it correctly with a direct smoke test (`timeout 2 ttfx -i
      ~/.config/omarchy/branding/screensaver.txt ...`, exit 0, no errors).

    Once `socat` lands after reboot, the full multi-monitor screensaver launch path
    (`omarchy-launch-screensaver`) should work end-to-end.

## Keeping `~/.omarchy` updated

Three different categories, each behaving differently under `git pull`:

1. **`bin/`, `default/`, `shell/`, `themes/`** — read live from `$OMARCHY_PATH` at runtime, no
   copy step. A plain `git pull` updates all of them immediately.
2. **`~/.config/hypr/` and `~/.config/omarchy/shell.json`** — one-time seeds, copied once and
   frozen. `git pull` never touches these. **This matches Omarchy's own model, not a gap in
   ours** — a real install only delivers post-seed default changes via the migrations system
   (`omarchy-migrate`), which we deliberately deferred. Manual check when curious:
   `diff ~/.omarchy/config/hypr/<file> ~/.config/hypr/<file>`.
3. **Direct patches to the checkout** (currently: the `Panel.qml` fix from issue 9) — these need
   to be actual git commits, not left as uncommitted working-tree changes, or a future `git pull`
   will refuse outright the moment upstream touches the same file ("local changes would be
   overwritten by merge"), blocking *all* updates until manually resolved. Committed as `ac84a440`
   on the local `quattro` branch (this repo's real default branch — not `main`, which doesn't
   exist here). Confirmed currently 0 commits behind `origin/quattro`.

   Notable: `origin/swap-tte-for-ttfx` already exists as a remote branch — upstream is
   independently moving toward the same `ttfx` we built in issue 10, before we even pulled it.

**Update: forked to `baumhoto/omarchy` and pushed.** Remotes are now:
- `origin` → `https://github.com/baumhoto/omarchy` (the fork — local commits push here)
- `upstream` → `https://github.com/omacom/omarchy.git` (the real project — note: a different org
  than `basecamp/omarchy`, which is what got cloned originally; the canonical location appears to
  have moved to a dedicated org)

Local `quattro` tracks `origin/quattro`, so **a plain `git pull` now pulls from the fork, not
upstream** — it won't see new upstream commits unless the fork is kept in sync. Routine going
forward:
```
git fetch upstream
git merge upstream/quattro    # or rebase
git push origin quattro       # keep the fork in sync
```
`ac84a440` (the `Panel.qml` fix) is pushed to the fork now — since it's a genuine upstream bug,
not Fedora-specific, worth opening a PR from the fork back to `omacom/omarchy` rather than
carrying it as a permanent local patch.

**Update, see issue 22**: the checkout itself was later reverted back to pristine (`09902660`) —
the actual fix now lives in a proper plugin clone at `~/.config/omarchy/plugins/tobi.monitor/`,
not as a live divergence in `~/.omarchy`. `ac84a440` stays in fork history purely as the source
diff for a future PR; the checkout no longer carries it day-to-day.

## Two bugs found via theme-switch side effects (not Fedora-specific)

11. **Monitor-disable state resets on theme switch.** Confirmed by reading the full code path,
    not a bug at all — genuine Omarchy behavior. Theme switching runs `omarchy-restart-hyprctl`
    (plain `hyprctl reload`), which reprocesses `~/.config/hypr/monitors.lua` from scratch. The
    Display panel's disable toggle ([`shell/plugins/panels/monitor/Panel.qml`](shell/plugins/panels/monitor/Panel.qml),
    `toggleDisplay()`) is **purely runtime** — a `hyprctl eval` call with no write-back to any
    config file. Checked thoroughly for a persistence mechanism (the way scale changes get
    persisted via `omarchy-hyprland-monitor-scaling`, and the way laptop-lid/clamshell state gets
    a dedicated toggle file) — none exists for general monitor-disable. Any reload wipes it back to
    `monitors.lua`'s static rules. If you want a disabled monitor to survive reloads, add a real
    `hl.monitor({output="<name>", disabled=true})` line to `~/.config/hypr/monitors.lua` yourself
    rather than relying on the panel toggle.

12. **Theme switch opened two VS Code instances.** Root cause was **entirely pre-existing and
    unrelated to Omarchy or this Fedora setup** — a bug in a personal `~/.bin/code` wrapper script
    that predates all of this work. It pointed at
    `/var/home/tobi/.bin/apps/code/code` — the raw Electron GUI binary from an extracted VS Code
    tarball — and unconditionally backgrounded + silenced every invocation
    (`... "$@" > /dev/null 2>&1 &`), including CLI-only management flags.

    [`bin/omarchy-theme-set-vscode`](bin/omarchy-theme-set-vscode) calls
    `code --list-extensions | grep -Fxq "$extension"` to check whether the current theme's VS Code
    extension (e.g. `TahaYVR.matteblack` for `matte-black`) is already installed, only calling
    `--install-extension` if not. Because the wrapper backgrounded `--list-extensions` and returned
    before real output arrived, the check always saw empty output → always concluded the extension
    was missing → always ran `--install-extension` → which the same broken wrapper turned into
    another backgrounded, undetached process — on every single theme switch.

    Fixed by pointing at the actual right binary: `apps/code/bin/code` is Microsoft's real
    CLI-capable launcher script (confirmed — it's the standard shipped launcher, not a custom
    tool), which already handles CLI flags synchronously and window-opening asynchronously on its
    own. No custom backgrounding/silencing logic needed at all:
    ```
    ~/.bin/code:
      exec /var/home/tobi/.bin/apps/code/bin/code "$@"
    ```
    Verified `code --version` and `code --list-extensions` now return real output synchronously
    (previously both returned nothing), and `code --install-extension TahaYVR.matteblack` installs
    correctly with a normal synchronous success message.

13. **Claude Desktop (`~/.bin/apps/claude`) not following system light/dark appearance** —
    unrelated to Hyprland/Omarchy, a standalone Electron-app-on-Linux issue. Its own setting was
    already correct (`~/.config/Claude/config.json`: `"userThemeMode": "system"`), and the system
    infrastructure it would query was also already correct — confirmed both layers directly:
    - `gsettings get org.gnome.desktop.interface color-scheme` → `'prefer-dark'`
    - `busctl --user call org.freedesktop.portal.Desktop ... org.freedesktop.portal.Settings Read
      "org.freedesktop.appearance" "color-scheme"` → `1` (prefer-dark)

    The portal chain works because `xdg-desktop-portal-gtk` and `xdg-desktop-portal-cosmic` are
    both already installed (came with the base Cosmic image, not something we layered), and
    Hyprland's own portal config (`/usr/share/xdg-desktop-portal/hyprland-portals.conf`:
    `default=hyprland;gtk`) correctly falls back to `gtk` for the Settings interface, which
    Hyprland's own portal doesn't implement.

    Root cause: the value it read at launch genuinely was wrong at the time (this instance had been
    running since before `prefer-dark` was ever set correctly), so it started in the wrong mode.
    **Fixed by quitting and relaunching the app**, confirmed working.

    **Update from issue 16**: this app *does* listen for live portal `SettingChanged` signals —
    confirmed later by flipping Cosmic's theme file while `cosmic-settings-daemon` ran and watching
    an already-open Claude Desktop window update immediately, no restart needed. Originally logged
    here as "untested" — now resolved. So a correctly-running theme pipeline (issue 16) means this
    app should track system theme changes live going forward, not just at launch.

14. **Wanted: prefer external display over internal whenever external is attached, lid open or
    closed** (not just lid-closed, which is what Omarchy's stock clamshell mode does). First
    attempt was wrong and reverted: a static `hl.monitor({output="eDP-1", disabled=true})` in
    `~/.config/hypr/monitors.lua` — this is *unconditional*, so unplugging the external would
    leave zero displays active. Reverted immediately.

    Real fix: the actual hotplug-watching infrastructure already runs continuously —
    [`bin/omarchy-hyprland-monitor-watch`](bin/omarchy-hyprland-monitor-watch) (launched from
    `autostart.lua`) listens on Hyprland's raw event socket for `monitoradded`/`monitorremoved`
    and calls [`bin/omarchy-hyprland-monitor-clamshell`](bin/omarchy-hyprland-monitor-clamshell)
    on every change, which already has correct, careful disable/enable logic (scale
    preservation, DPMS handling, debounced reload). The **only** thing gating it to lid-closed
    was one line:
    ```
    if omarchy-hw-clamshell && omarchy-hyprland-monitor-external-active; then
    ```
    Dropped the `omarchy-hw-clamshell &&` — now it's just `if
    omarchy-hyprland-monitor-external-active; then`. Everything else (the well-tested
    enable/disable functions) is untouched. Verified: manually triggering the script with the
    external attached correctly disabled `eDP-1`; a subsequent `hyprctl reload` (what theme
    switching triggers) left it correctly disabled too — because the disable state now flows
    through Omarchy's own toggle-file system
    (`~/.local/state/omarchy/toggles/hypr/internal-monitor-clamshell.lua`, regenerated and
    reloaded fresh every time via `toggles.lua`), not a static hand-written rule.

    **This is a personal policy change, not an upstream bug fix** — committed locally as
    `17db6918` with that called out explicitly in the message, so it doesn't get confused for
    something to PR back (unlike the `Panel.qml` fix). Needs `git push origin quattro` run from
    an authenticated shell — this session's `git push` failed for lack of credentials.

15. **Cosmic apps (Cosmic Files, etc.) didn't follow Omarchy theme switches.** Expected — Omarchy
    has no idea Cosmic exists, and Cosmic doesn't read the standard `org.freedesktop.appearance`
    portal setting for dark mode; it reads its own dedicated state file directly:
    ```
    ~/.config/cosmic/com.system76.CosmicTheme.Mode/v1/is_dark   (contains: true / false)
    ```
    Fixed the **fully supported, documented way** — no checkout modification needed at all.
    `omarchy-theme-set` already calls `omarchy-hook theme-set "$THEME_NAME"` on every switch, and
    `bin/omarchy-hook` runs anything in `~/.config/omarchy/hooks/theme-set.d/*` — confirmed via the
    real shipped example at `config/omarchy/hooks/theme-set.d/show-theme-notification.sample`.
    Added `~/.config/omarchy/hooks/theme-set.d/cosmic-theme-mode.sh`: reads the active theme's
    `mode` key via `omarchy-theme-color --file .../colors.toml --all` (`dark`/`light`) and writes
    the corresponding `true`/`false` to Cosmic's file.

    Verified both directions through the real `omarchy-theme-set` flow, not just the hook script
    standalone: switching to `catppuccin-latte` (light) flipped the file to `false`; switching back
    to `matte-black` (dark) flipped it to `true`.

    **Side finding, unrelated, not yet investigated:** both test switches printed `Error accessing
    /usr/bin/omarchy-theme-set-browser-policy: No such file or directory` — the same
    hardcoded-`/usr/bin/omarchy-*`-path pattern as issues 7/8 (systemd units), this time from
    `omarchy-theme-set-browser`'s sudoers-based browser policy write. Doesn't block anything found
    so far (theme switching itself completes fine either way), but worth a proper look later if
    browser theming turns out to not be working.

16. **Cosmic apps needed a manual restart to see theme changes from issue 15** (writing the file
    updated the value, but already-open apps didn't notice). Root cause: `cosmic-settings-daemon`
    — the piece that normally live-broadcasts Cosmic theme changes to running apps — only runs as
    part of an actual Cosmic session (`cosmic-session` autostarts it; no systemd unit ships for it
    at all, confirmed via `rpm -ql cosmic-settings-daemon`). It was never running under Hyprland.

    **Fixed by running it manually in this session too**, the same reasoning as
    `gvfs-udisks2-volume-monitor` being genuinely DE-agnostic (issue 5) — tested, not assumed:
    ```
    nohup /usr/bin/cosmic-settings-daemon > /tmp/cosmic-settings-daemon.log 2>&1 &
    ```
    Confirmed via log output reacting to a real file change (new lines at a different code path,
    `theme.rs:272`, on write vs. the startup lines at `171`/`181`) that it actively watches and
    reprocesses the theme file live. Logs two non-fatal `list_button` key-not-found warnings at
    startup and on every reprocess — a missing optional Cosmic widget-style default, never seen
    blocking anything, safe to ignore.

    **Genuine bonus, not assumed**: `cosmic-settings-daemon` also syncs
    `org.gnome.desktop.interface color-scheme` (confirmed via `gsettings get` before/after) — so it
    benefits *any* app using the standard portal-based dark-mode detection, not just Cosmic
    apps. **Confirmed live end-to-end**: flipping Cosmic's `is_dark` file updated Cosmic Files
    immediately (no restart) *and* updated an already-open Claude Desktop window immediately too
    — meaning Claude Desktop's `userThemeMode: "system"` (issue 13) does listen for live portal
    `SettingChanged` signals after all. **Correction to issue 13**: that entry said this was
    "untested" — it's now confirmed working live, not just at launch.

    **Not yet persistent** — currently just a manually-started background process from this
    session, so it will not survive logout/reboot. To make it durable, add it to
    `~/.config/hypr/autostart.lua` (currently an empty user-override template) alongside how
    Omarchy autostarts its own shell:
    ```lua
    o.launch_on_start("cosmic-settings-daemon")
    ```
    (exact `o.launch_on_start` helper signature should be double-checked against
    `default/hypr/helpers.lua` before relying on it — not yet verified against the real API in
    this session.) **Status: works now, needs the autostart addition to survive a reboot.**

## Temporary stopgaps

- **`nvim` → `vi`**: `~/.bin/nvim` is a symlink to `/usr/bin/vi` (`vim-minimal` from the base
  image — confirmed part of the base image, not layered, see the earlier vim-provenance check).
  A stopgap so `nvim` resolves to *something* usable in the meantime, not the real Neovim/LazyVim
  setup. **Superseded once Phase 4 of the plan (`omarchy-nvim`) is actually done** — real `neovim`
  needs to be layered via `rpm-ostree` and this symlink removed/replaced at that point, since
  `vi`/`vim-minimal` doesn't support the Lua config, plugins, or LSP integration the real setup
  needs.

17. **"Screen did not lock before suspend (No lock screen is configured)" notification.** Traced
    through [`bin/omarchy-system-sleep-lock`](bin/omarchy-system-sleep-lock) → the shell IPC reply
    `missing-pam` → [`shell/plugins/lock/Service.qml`](shell/plugins/lock/Service.qml): the lock
    plugin requires `/etc/pam.d/omarchy-lock-password` to exist at all (watches the file path
    directly), and separately checks for `/etc/pam.d/omarchy-lock-fingerprint` (gates the
    fingerprint-unlock UI, requires both the file and an enrolled fingerprint via `fprintd-list`).
    Neither existed — [`bin/omarchy-apply-lock`](bin/omarchy-apply-lock) (the real install-time
    step that creates them) never ran, since we skipped the system-level install sequence.

    **Checked first whether Cosmic already had this covered** (matching the udiskie/bt-agent
    pattern) — it doesn't directly, but the check surfaced something better: Fedora's `authselect`
    already has `with-fingerprint` enabled, and `system-auth` bundles `pam_fprintd.so` (sufficient)
    and `pam_unix.so` (sufficient) in one stack. Omarchy's own PAM content (built for Arch) doesn't
    just not need duplicating — it doesn't fully port: it references `account include
    system-local-login`, a PAM service file Arch ships that **does not exist on Fedora** (confirmed
    via `ls`). Fedora's actual equivalent is `system-auth`, which already gets both password and
    fingerprint in one place.

    Wrote both files with Fedora-native equivalents instead of the Arch original:
    ```
    /etc/pam.d/omarchy-lock-password:
      auth       include      system-auth
      account    include      system-auth

    /etc/pam.d/omarchy-lock-fingerprint:
      auth       required     pam_fprintd.so
      account    include      system-auth
    ```
    (The password file alone already permits fingerprint-or-password since `system-auth` tries
    `pam_fprintd.so` first — the fingerprint file is still worth having separately since the shell
    specifically checks for its existence to decide whether to show a fingerprint prompt at all.)

    Verified via the shell's own status query, not just file existence:
    `omarchy-shell lock status` went from `"passwordPam":false,"fingerprint":false` to
    `"passwordPam":true,"fingerprint":false` immediately after writing the files, then to
    `"passwordPam":true,"fingerprint":true` after `omarchy-restart-shell` — the fingerprint flag
    specifically needed a shell restart to pick up (checked once at shell startup, same pattern as
    issue 13/Claude Desktop).

18. **Battery widget's power-profiles menu listed nothing.** Traced to
    [`shell/plugins/menu/Menu.qml:277`](shell/plugins/menu/Menu.qml)'s `power-profiles` entry: its
    list script calls `powerprofilesctl get` and
    [`bin/omarchy-powerprofiles-list`](bin/omarchy-powerprofiles-list) (which itself wraps
    `powerprofilesctl list`) — that binary doesn't exist on this system, so both calls silently
    fail (`2>/dev/null`) and the loop produces zero rows.

    **Checked whether Cosmic already had this covered first** (same pattern as issues 5/7): power
    profile switching does work under Cosmic, but not via `power-profiles-daemon` — Fedora ships
    `tuned-ppd` instead, already installed and running, already registered on the exact
    `net.hadess.PowerProfiles` D-Bus name `powerprofilesctl` talks to (confirmed via `busctl
    --system list` and by reading `Profiles`/`ActiveProfile` directly — all three profiles and the
    correct active one came back immediately, no gaps in the underlying plumbing at all).

    So the fix isn't "install `power-profiles-daemon`" — doing that would install a second,
    competing implementation of the same D-Bus service `tuned-ppd` already provides, fighting it
    for the same bus name. Instead, **rewrote the three touch points to talk to
    `net.hadess.PowerProfiles` directly via `busctl --json=short` + `jq`**, bypassing
    `powerprofilesctl` entirely:
    - `bin/omarchy-powerprofiles-list` — reads the `Profiles`/`ActiveProfile` properties directly
    - `bin/omarchy-powerprofiles-set` — `busctl set-property ... ActiveProfile s "$profile"`
      instead of `powerprofilesctl set`
    - `shell/plugins/menu/Menu.qml:277` — same `ActiveProfile` read inline

    Verified each in isolation before touching files (`busctl` read/write worked immediately,
    including setting to `power-saver` and back to `balanced` cleanly), then verified all three
    edited files end-to-end: `omarchy-powerprofiles-list` lists all three profiles with correct
    active-state; `omarchy-powerprofiles-set battery power-saver` / `... ac balanced` both worked
    and the invalid-profile rejection path still works unchanged; the exact `Menu.qml` script
    string tested standalone produces the correct three-column output. Committed as `5b23a55d`.

    This is a genuine Fedora-vs-Arch compatibility fix (Arch's `power-profiles-daemon` has no
    `tuned-ppd`-style conflict there), not a bug.

    **Correction, see issue 22**: the `Menu.qml` part of this was initially committed as a direct
    checkout edit — wrong, per Omarchy's own documented workflow. Moved to a proper plugin clone;
    the `bin/omarchy-powerprofiles-*` script edits stay as checkout edits (no equivalent clone
    mechanism exists for `bin/` commands).

19. **Webapp keybindings broken** (`SUPER + SHIFT + Y` for YouTube, and every other `{webapp =
    ...}` binding — Maps, Calendar, Email, ChatGPT, Grok, WhatsApp, Google Messages, Google
    Photos). This is the actual root cause of the very first "App failure: Error: Path
    '--app=https:/maps.google.com' does not exist!" notification from much earlier — never fully
    traced at the time, now fully understood.

    Traced through [`default/hypr/helpers.lua`](default/hypr/helpers.lua)'s `o.launch_webapp` →
    [`bin/omarchy-launch-webapp`](bin/omarchy-launch-webapp) (both the plain and
    `omarchy-launch-or-focus-webapp` focus-variant delegate to this one script). It resolves the
    default browser via `xdg-settings get default-web-browser`, and if that's not one of a
    hardcoded Chromium-family whitelist (`google-chrome*|brave*|microsoft-edge*|...`), falls back
    to a **hardcoded** `chromium.desktop`. Confirmed by reproducing the exact failure by hand: our
    default reports as `chromium-browser.desktop` — not in the whitelist — so it fell back to the
    hardcoded name, which doesn't exist here (Fedora's `chromium` package ships
    `chromium-browser.desktop`; `chromium.desktop` is Arch's naming). The `sed` extraction found
    nothing, so `exec ... --app="$1"` ran with no binary before the flag — bash tried to execute
    the literal string `--app=<url>` as a command, producing exactly the "Path ... does not exist"
    notification.

    Fixed in `bin/omarchy-launch-webapp`: added `chromium*` directly to the whitelist (covers our
    actual default outright), and replaced the hardcoded fallback with a probe across both
    possible filenames (`chromium.desktop`, `chromium-browser.desktop`) so it works regardless of
    which distro's naming is present. Verified by reproducing the broken resolution by hand first
    (confirmed empty command), then confirming the fixed logic resolves
    `/usr/bin/chromium-browser` correctly, then actually launching YouTube end-to-end — a real
    Chromium process came up with `--app=https://youtube.com/` in its command line. Committed as
    `77f5de68`.

20. **Alacritty wasn't following theme switches.** Checked Omarchy's real mechanism first (it does
    work by design): `~/.omarchy/config/alacritty/alacritty.toml` (the actual seed) imports
    `~/.local/state/omarchy/current/theme/alacritty.toml` — the per-theme staged file
    `omarchy-theme-set` regenerates on every switch — and `omarchy-theme-set`'s
    `post_theme_commands` already calls `omarchy-restart-terminal`, which `touch`es the config to
    trigger Alacritty's own `live_config_reload`.

    The gap: `~/.config/alacritty/alacritty.toml` was never actually the Omarchy seed — `config/`
    was only copied for `hypr` back in Phase 1, and this machine already had a personal Alacritty
    config from before any of this work, importing a static file from the third-party
    `alacritty-theme` collection (`alacritty-theme/themes/pencil_light.toml`) instead.

    Fixed surgically rather than reseeding the whole file (real personal customizations were
    present — font, cursor style, tmux-as-default-shell, window position): changed just the
    `import` line to `~/.local/state/omarchy/current/theme/alacritty.toml`, matching Omarchy's real
    seed. Verified end-to-end, not just the config change: switched to `Tokyo Night` and confirmed
    the staged file picked up its colors (`#1a1b26`/`#a9b1d6`), then back to `matte-black`
    (`#121212`/`#bebebe`) — both via the real `omarchy-theme-set` flow, which also re-triggers
    `omarchy-restart-terminal` automatically each time.

21. **System lock works once, then silently does nothing on retry** — a real upstream Omarchy bug,
    confirmed not Fedora-specific, closely related to (possibly the same root cause as) the open
    issue [`omacom/omarchy#7072`](https://github.com/omacom/omarchy/issues/7072) ("WlSessionLock
    reverted by compositor after ~4s"). **Security-relevant**: the IPC handler returns `"ok"` even
    when it silently does nothing, so pressing lock again after this state is reached leaves the
    screen genuinely unlocked while appearing to have succeeded.

    Reproduced twice with genuine interactive use (not just headless testing): first
    lock→unlock (password, then separately fingerprint) cycle works correctly end-to-end each
    time; the *second* lock attempt afterward does nothing — no lock layer appears
    (`hyprctl layers` shows only `omarchy-background`/`omarchy-bar`, no lock namespace).

    Root cause traced precisely in [`shell/plugins/lock/Service.qml`](shell/plugins/lock/Service.qml):
    ```qml
    function lock(): string {
      if (!root.passwordPamConfigured) return "missing-pam"
      if (!root.locked && !root.beginLock()) return "failed"
      return "ok"
    }
    ```
    If `root.locked` is (wrongly) already `true`, `beginLock()` never runs and it still returns
    `"ok"`. `locked` is a computed property (`lockRequested || sessionLock.locked ||
    sessionLock.secure`) that should be fully reactive — but `omarchy-shell lock status` showed
    `"locked":true` while **all three** of its own dependencies (`requested`, `sessionLocked`,
    `secure`) independently read `false`, well after any transition had settled. That's a genuine
    QML/Quickshell reactivity bug, not a simple stuck flag — not something fixable by re-reading
    the source alone; would need live QML debugging to root-cause fully.

    **Workaround for now**: `omarchy-restart-shell` resets all this runtime state cleanly (verified
    — `locked` goes back to `false`, `lastEvent` back to `"init"`). Needed before any lock attempt
    that follows a completed lock/unlock cycle.

22. **Should have used `omarchy plugin clone`, not direct checkout edits, for local shell-plugin
    customizations.** Flagged after watching the Quattro release video, which specifically calls
    this out. Confirmed via [`shell/plugins/README.md`](shell/plugins/README.md) and
    [`agents/skills/omarchy/plugins.md`](agents/skills/omarchy/plugins.md):

    > "To customize a built-in bar widget, never edit `$OMARCHY_PATH/shell/plugins/`. Clone it into
    > the user plugin directory instead: `omarchy plugin clone omarchy.workspaces`"

    `omarchy-plugin-clone` copies a built-in plugin's full source into
    `~/.config/omarchy/plugins/<username>.<id>/`, rewrites its manifest id (`clonedFrom` tracks the
    original), and switches the shell to the clone in place of the built-in — a supported,
    update-proof customization surface that lives entirely outside the git-tracked checkout.

    **Applied to issue 18's `Menu.qml` power-profiles fix** (genuinely Fedora-specific, not
    upstream material): `omarchy-plugin-clone omarchy.menu` — the clone inherited our fix
    automatically since it copies from the checkout's then-current (already-patched) state.
    Reverted `shell/plugins/menu/Menu.qml` back to byte-identical-with-upstream content afterward
    (diffed against `upstream/quattro`, confirmed identical), and reconfirmed the power-profiles
    script still works correctly through the clone. Now lives at
    `~/.config/omarchy/plugins/tobi.menu/`, `omarchy-plugin-list --json` confirms `tobi.menu
    enabled=true` / `omarchy.menu enabled=false`.

    **Correction**: initially planned to leave issue 9's `Panel.qml` fix (the `hyprctl
    eval`/`hl.monitor` disable fix) as a direct checkout edit, reasoning that since it's meant to
    become an upstream PR, a plugin clone couldn't be the basis for that diff. Wrong — cloning and
    PR-ing aren't mutually exclusive, and there's a real, specific future collision a direct edit
    doesn't protect against: when Omarchy eventually bumps its own Hyprland dependency past this
    same "`hyprctl keyword` no longer works" point, upstream will fix this exact code themselves,
    and our direct edit would conflict with their fix on the next `git pull`. A PR can be prepared
    later by diffing this already-made commit against a fresh `upstream/quattro` checkout — that
    never required the *live* checkout to carry the edit directly.

    Applied the same clone-and-revert treatment: `omarchy-plugin-clone omarchy.monitor` (inherited
    the fix automatically), reverted `shell/plugins/panels/monitor/Panel.qml` to byte-identical
    upstream content, reconfirmed the `hl.monitor` disable/enable mechanism still works live. Now
    lives at `~/.config/omarchy/plugins/tobi.monitor/`; `omarchy-plugin-list --json` confirms
    `tobi.monitor enabled=true` / `omarchy.monitor enabled=false`. Checkout commit reverted; the
    original fix commit (`ac84a440`) stays in fork history as the source to diff for the eventual
    PR.

    **Side discovery while investigating this**: `origin/quattro` (our fork) already matched our
    locally-modified `Menu.qml` *before* this revert — meaning GitHub Desktop had already
    successfully pushed our earlier commits (`5b23a55d` etc.) to the fork, silently succeeding
    where this session's own `git push` attempts had failed for lack of credentials. Comparing
    against the wrong remote (`origin` instead of `upstream`) would have shown a false "no diff"
    and hidden the actual change needing reverting — worth remembering to diff against `upstream/*`
    specifically when checking "is this really pristine," not `origin/*`.

    **No equivalent mechanism exists for `bin/` script customizations** (checked: no
    `omarchy-cmd-clone` or similar) — the `omarchy-powerprofiles-list`/`omarchy-powerprofiles-set`
    edits from issue 18, and the `omarchy-launch-webapp` fix from issue 19, have no better home than
    the checkout itself, since `$OMARCHY_PATH/bin` is prepended ahead of anything in `~/.local/bin`
    (confirmed all the way back in issue 5's `udiskie` investigation), so a `~/.local/bin` shim
    can't shadow an *existing* command the way it can supply a *missing* one. Those stay as direct
    checkout edits — the best available option given Omarchy's actual tooling, not a shortcut we
    took.

## Still deferred (per the plan, not bugs)

- `omarchy-pkg-*` pacman shims and anything gated behind them (`omarchy-install-*`, most
  `omarchy-remove-*`, the migrations system)
- Hardware-quirk detection scripts (`install/hardware/*`)
- `omarchy-dev-link`'s sudoers/`secure_path` handling — not needed until something calls
  `sudo omarchy-*`

## Verification checklist

Only checking items you've actually confirmed — inferred-but-unconfirmed items stay unchecked.

- [ ] `echo $OMARCHY_PATH` → `/home/tobi/.omarchy`
- [ ] `hyprctl reload` / `hyprctl configerrors` clean
- [ ] `SUPER + RETURN` opens Alacritty
- [ ] Quickshell bar renders with icons
- [x] Theme switching applies colors (implied by hitting the backgrounds issue — you got far
      enough to notice backgrounds specifically weren't working)
- [ ] Theme backgrounds render (blocked on `qt6-qtimageformats`, see above)
- [ ] Notifications via `omarchy-notification-send` show up

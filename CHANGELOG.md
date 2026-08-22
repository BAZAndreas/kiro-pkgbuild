# CHANGELOG — kiro-pkgbuild

> PKGBUILDs for building custom Calamares packages (`calamares` and `calamares-next`).
> Each build directory is versioned after the upstream Calamares git snapshot.

---

## 2026.08.22 — `calamares-3.4.2.r4.g841b478-17` / `calamares-next-3.4.2.r4.g841b478-16`

### What Changed

- **`kiro-trust-desktop-launchers` was being installed non-executable.** Both PKGBUILDs used
  `install -Dm644` for a `#!/bin/bash` script, so it landed at mode `0644` on the live ISO.
  Running it directly returns **exit 126 / Permission denied**.
- **Consequence:** the XDG autostart entry that is the script's *intended* trigger —
  `~/.config/autostart/trust-desktop-launchers.desktop`, whose `Exec=` points straight at the
  path — has been failing silently on every live boot. XFCE/Thunar then pops the "Untrusted
  application launcher" dialog on the desktop "Install kiro" icon, which is exactly the
  prompt this script exists to prevent.
- **Why it went unnoticed:** `kiro-trust-launchers.service` invokes it as
  `ExecStart=/bin/bash /usr/local/bin/kiro-trust-desktop-launchers`. Passing the script to
  `bash` as an argument bypasses the exec bit entirely, so the unit reported
  `status=0/SUCCESS` and masked the broken permission for as long as it has been wrong.
- Fixed to `install -Dm755` in both tracks. New build dirs rather than in-place edits, because
  `-16` / `-15` are already built and shipped on ISO v26.08.22.

### Technical Details

- `calamares-3.4.2.r4.g841b478-17/PKGBUILD` — `pkgrel=16` → `17`, line 151 `-Dm644` → `-Dm755`.
- `calamares-next-3.4.2.r4.g841b478-16/PKGBUILD` — `pkgrel=15` → `16`, same one-line change.
- Both dirs are otherwise byte-identical copies of their predecessor (the vconsole XKB patch,
  `build.sh`, `install`, `cal-kiro.desktop`, `calamares_polkit`, the unit and the script itself
  all carry over unchanged).
- **Both trigger paths are deliberately kept.** The `.desktop` and the systemd user unit now
  both work, and the script is idempotent — it only ever runs `chmod +x` plus two `gio set`
  calls in a loop, each `|| true`-guarded. Removing either mechanism would risk the feature on
  whichever session type the other one doesn't cover, for no gain.
- `calamares-wayland` in `~/KIROTUX/KIROTUX-PKG-BUILD/` carries the identical `-Dm644` bug at
  line 148 and was **intentionally left untouched** at Erik's instruction.
- Found by `/kiro-check` against a live ISO boot of v26.08.22.

---

## 2026.08.18 — `calamares` / `calamares-next` `-3.4.2.r4.g841b478-15`

### What Changed

- **New build dirs `calamares-3.4.2.r4.g841b478-15` and `calamares-next-3.4.2.r4.g841b478-15`**
  (both `pkgrel=15`) carrying a downstream patch:
  **`0001-keyboard-persist-xkb-state-in-vconsole-conf.patch`**.
- The patch makes Calamares write `XKBLAYOUT=` / `XKBMODEL=` / `XKBVARIANT=` / `XKBOPTIONS=`
  into the installed system's `/etc/vconsole.conf`, next to the `KEYMAP=` line it already wrote.
- **Why:** systemd >= 256 reads the X11/XKB keyboard configuration from `vconsole.conf`, not
  from `/etc/X11/xorg.conf.d/00-keyboard.conf`. Reproduced on a running system with systemd 261:
  `00-keyboard.conf` correctly held `XkbLayout "be"`, yet `localectl status` reported
  `X11 Layout: (unset)`. Everything that asks systemd-localed for the layout — Wayland
  compositors, greeters, desktop settings panels — got nothing on a fresh install.

### Technical Details

- Backported from CachyOS/cachyos-calamares commit `5afd22bd0` (PR #245), **trimmed to the
  vconsole change only**. The upstream commit also swaps X11 variant handling over to a new
  `variantList()` helper; that is a separate behavioural change affecting `writeX11Data()` and
  `writeKeyboardData()`, and was deliberately left out to keep the patch minimal. The vconsole
  variants therefore use the same `removeEmpty( { additionalVariant, m_variant } )` expression
  the tree already uses for X11, so the two files stay consistent.
- The patch also drops any stale `XKB*` lines already present in `vconsole.conf` before
  appending the new ones, so re-running the job cannot leave two conflicting values behind, and
  checks the return value of the read-side `file.open()`, which was ignored.
- `/etc/X11/xorg.conf.d/00-keyboard.conf` is still written exactly as before, for bare X11 sessions.
- Applied as a real patch file rather than a commit on `codeberg.org/erikdubois/calamares`, so
  the fork stays untouched. Wired into `prepare()` with `patch -Np1 -d "$srcdir/$_pkgname"` —
  the `-d` form is required because `prepare()` never `cd`s into the source tree.
- Verified to apply cleanly against a pristine `--depth 1` clone of the fork at `841b478`.
  **Not compile-tested** — `build.sh` (`makepkg`) is the real check.
- Note: `source=` pins no `#commit=` fragment, so `makepkg` builds the fork's default-branch tip.
  The patch was generated against that tip; if the fork moves, the patch may need refreshing.
- Drop the patch once the fix lands in the Calamares tree we build from.

- Both tracks build from the same fork with the same `_pkgname=calamares`, so the patch file is
  byte-identical in both dirs and the two PKGBUILDs now differ only in `pkgname`, `conflicts` and
  `provides`.

### Files Modified

- `calamares-3.4.2.r4.g841b478-15/PKGBUILD` — `pkgrel` 14 -> 15; patch added to `source=` +
  `sha256sums=`; `patch -Np1` step added to `prepare()`
- `calamares-3.4.2.r4.g841b478-15/0001-keyboard-persist-xkb-state-in-vconsole-conf.patch` — new
- `calamares-next-3.4.2.r4.g841b478-15/` — new build dir, copied from `-14` with the same
  `pkgrel` bump, patch file and `prepare()` wiring

---

## 2026-04-26 — `calamares-3.3.14.r132.g841b478-4`

Latest build — both `calamares` and `calamares-next` variants at version `-4`:

- **`PKGBUILD`** — full build definition (124 lines)
- **`build-calamares`** — build automation script (46 lines)
- **`cal-kiro.desktop`** — launcher (240 lines)
- **`calamares-wrapper`** — launch wrapper (38 lines)
- **`calamares_polkit`** — polkit rule (6 lines)
- **`install`** — post-install hook (40 lines)
- **Patched modules included:**
  - `bootloader/main.py` (966 lines) — custom bootloader logic
  - `packages/main.py` (832 lines) — custom package handling

**Previous builds in repo (newest → oldest):**

| Build dir                                   | Notes                                     |
|---------------------------------------------|-------------------------------------------|
| `calamares-3.3.14.r132.g841b478-4`          | Current — both calamares + calamares-next |
| `calamares-3.3.14.r132.g841b478-3`          | Previous iteration                        |
| `calamares-3.3.14.r132.g841b478-2`          | Earlier iteration                         |
| `calamares-3.3.14.r87.g3f6cd83-1`           | Earlier upstream snapshot                 |
| `calamares-3.3.14.r90.g53c70f8-1`           | Earlier upstream snapshot                 |
| `calamares-3.3.14.r81.g55f0c9e-2`           | Earlier upstream snapshot                 |
| `calamares-git-3.3.14.r51.g3b9ef52-2`       | Original `-git` named build               |
| `calamares-next-3.3.14.r132.g841b478-2/3/4` | Next-track parallel builds                |

---

## Build Structure

Each versioned folder contains:
```
PKGBUILD
build-calamares          # builds and installs the package
cal-kiro.desktop         # Calamares desktop launcher
calamares-wrapper        # shell wrapper for launch
calamares_polkit         # polkit authentication rule
install                  # pacman install hook
modules/
  bootloader/            # patched bootloader module
  packages/              # patched packages module
```

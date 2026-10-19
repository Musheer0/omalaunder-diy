# Installing Strata on omalauncher's Arch (ARM)

**Result:** Strata installed for a non-root user, binary at
`/home/<user>/.local/bin/strata`, plus a desktop entry and icon.

Target: Arch Linux ARM (aarch64). Substitute your own non-root account name
for `<user>` throughout.

---

## 1. Why the installer must not run as root

```console
# curl -fsSL https://raw.githubusercontent.com/lgse/strata/main/install.sh | bash
  ×  Run this installer as your normal desktop user, not as root.
```

The shell here is `root`. `install.sh` checks `EUID` early on and refuses to
run as root (`[[ $EUID -ne 0 ]] || die "Run this installer as your normal
desktop user, not as root."`), because it installs into `$HOME/.local/bin` and
needs a real user's home directory.

Run it as your normal non-root user:

```console
# su - <user> -c 'curl -fsSL https://raw.githubusercontent.com/lgse/strata/main/install.sh | bash -s -- --verbose --with-desktop-entry'
```

## 2. The catch: the non-root user may not be able to run `sudo`

The installer installs the runtime dependencies with
`sudo -n pacman -S --needed --noconfirm -- <packages>` when running
non-interactively, and `sudo pacman -S --needed -- <packages> </dev/tty`
when interactive. If the target user has no sudo rights, the installer dies
at the dependency step.

**Fix:** install the shared runtime packages once as `root` (system packages
are root's job anyway), then let the installer see them as already present and
skip `sudo` entirely. No sudoers change is needed.

The exact runtime packages the installer requires
(`REQUIRED_PACKAGES` in `install.sh`), all available in the Arch ARM repos:

```console
# pacman -S --needed --noconfirm \
    bubblewrap desktop-file-utils ffmpeg ffmpegthumbnailer fontconfig \
    gst-libav gstreamer gst-plugins-base gst-plugins-good gtk4 \
    gtksourceview5 gvfs poppler-glib xdg-utils
```

> Note: `pacman -Sy` may print
> `error: failed retrieving file 'extra.db' from <mirror> : 403`
> (some mirrors refuse the `.db`, while the package files themselves download
> fine). If a current `extra.db` is already in `/var/lib/pacman/sync/`, plain
> `pacman -S` works. Putting
> `Server = http://mirror.archlinuxarm.org/$arch/$repo` first in
> `/etc/pacman.d/mirrorlist` fixes the refresh.

## 3. Flags used

* `--with-desktop-entry` — the installer's "standard" install: application +
  launcher icon, with the default apps and keybinds left alone. It also implies
  `--non-interactive`, so no TTY menu is needed.
* `--verbose` — stream the download/verify steps into the output.

What the installer does:

```
  Arch Linux ARM · aarch64
  Install to /home/<user>/.local/bin/strata
  +  Application launcher
  ✓  Download Strata 1.0.0
  ✓  Download release checksums
  ✓  Verify SHA-256 checksum          strata-1.0.0-aarch64-unknown-linux-gnu.tar.gz: OK
  ✓  Unpack the verified release
  ✓  All selected runtime packages are already installed.
  ✓  Install the application
  ✓  Added Strata to the desktop application menu.
  !  /home/<user>/.local/bin is not on PATH in this shell.
  ✓  Strata 1.0.0 is installed.
```

The release is fetched from
`https://github.com/lgse/strata/releases/download/v1.0.0/strata-1.0.0-aarch64-unknown-linux-gnu.tar.gz`
and the SHA-256 check against the published `.sha256` passes. If GitHub CLI is
installed but not authenticated for the target user, the installer skips the
build-attestation check with a warning (HTTPS + checksum still protect the
download).

Full log: `/tmp/strata-install.*.log`.

## 4. Post-install touches

`~/.local/bin` is not in the default Arch `PATH`, and the installer only warns
about it. Add it to the user's shell startup files
(`/home/<user>/.bashrc` for interactive shells, `/home/<user>/.bash_profile`
for login shells, before sourcing `.bashrc`):

```bash
# ~/.local/bin (user-installed binaries, e.g. strata)
[[ :$PATH: == *":$HOME/.local/bin:"* ]] || export PATH="$HOME/.local/bin:$PATH"
```

Additionally, when the *session* runs as `root` (see §6) and root's `PATH`
already contains `/root/.local/bin`, link Strata in there as well:

```console
# ln -sfn /home/<user>/.local/bin/strata /root/.local/bin/strata
# command -v strata
/root/.local/bin/strata
```

That is what makes the plain command `strata` work in the session (see §7).

## 5. Verification

```console
# su - <user> -c 'command -v strata'
/home/<user>/.local/bin/strata

# su - <user> -c 'strata --help'
Usage:
  io.github.lgse.Strata [OPTION…]
...
Application Options:
  --unlock-volume=DEVICE     Unlock an encrypted volume and show the password prompt
```

No unresolved shared libraries (`ldd` clean), binary is ARM aarch64 ELF, ~37 MB.

Installed files:

| path | what |
|---|---|
| `/home/<user>/.local/bin/strata` | the application |
| `/home/<user>/.local/share/applications/io.github.lgse.Strata.desktop` | launcher entry (`Exec=/home/<user>/.local/bin/strata %U`) |
| `/home/<user>/.local/share/icons/hicolor/scalable/apps/io.github.lgse.Strata.svg` | icon |

## 6. Caveats / follow-ups

* **If the launcher/session runs as `root`** (`HOME=/root`, where the live
  `~/.config/omalauncher` files are bind-mounted from the Android app) with
  only the launcher's Wayland socket available, a non-root uid may not be able
  to open the display at all. In that case the *usable* copy for anything
  launched from the phone is the one on root's `PATH` (the symlink in §4).
  Alternative: re-run the installer with `HOME=/root` as a non-root uid (the
  installer's root check is on `EUID`, not `HOME`), e.g.
  `su -s /bin/bash <user> -c 'HOME=/root bash -s -- --with-desktop-entry' < install.sh`.
* **GUI**: Strata is a GTK 4 file manager; it needs the Wayland display.
* Optional extras the installer can add later: `--with-smb` (Windows shares),
  `--with-raw` (camera RAW previews), `--with-file-manager` ("Open file location"),
  `--with-file-chooser` (portal dialogs). (`--with-omarchy-keybinds` does not apply here:
  there is no Omarchy version file and no `~/.config/hypr/bindings.*`, and the launcher
  drives the home screen itself.)
* If you would rather give the non-root user real sudo and let the installer
  manage packages itself, that is a one-line NOPASSWD policy in a sudoers drop-in
  (validate with `visudo -cf` first); it was not needed here because the
  dependencies were installed up front.

## 7. How to launch it

If everything in the session is one user on the launcher's Wayland socket, the
GUI must be started from a shell in that session — the **terminal tile** on the
home screen (a real `foot` shell) or any `foot` window.

```console
~ strata                 # opens in ~  (start folder = the working directory)
~ strata /some/folder    # or any folder to open directly
```

`strata` resolves because of the symlink in root's `PATH` (§4); the full path
`/home/<user>/.local/bin/strata` always works too. Verified by running it:

```console
~ timeout 12 strata /some/folder    # (only to test: it ran for 12s, then timeout killed it)
 Gtk-WARNING: Unable to acquire session bus: Cannot spawn a message bus without a machine-id
 INFO strata::metrics: application window presented elapsed_ms=1202
 INFO strata::adapters::local_files: first directory batch ready request_id=1 entries=2
 INFO strata::adapters::local_files: directory load finished request_id=1 entries=2
```

(`entries=2` matched the test folder's contents, so the folder argument was
honoured.)

It takes ~1–2 s to paint the first frame. Its stderr goes to the terminal, so
launching it from a shell is also how you see its warnings.

Running it via `su` usually does **not** work for the GUI: the session's Wayland
socket is typically mode `600` and owned by the session user, so the target uid
has no access — run it as the session user instead.

The Gtk warning about the session bus is harmless when there is no session D-Bus
daemon and `/etc/machine-id` is empty (0 bytes) — D-Bus autolaunch can't work.
Strata continues without it. Filling in `/etc/machine-id`
(`dbus-uuidgen --ensure`) silences the warning. Separately, the launcher's app
drawer lists **OWL apps** (`app "name" { }` files in
`~/.config/omalauncher/widgets/`), not XDG `.desktop` files, so Strata's desktop
entry is not what puts it in the drawer; a small OWL app that calls
`run("strata")` can be added if you want a drawer icon.

### Uninstall

```console
# su - <user> -c '
  rm -f  ~/.local/bin/strata
  rm -f  ~/.local/share/applications/io.github.lgse.Strata.desktop
  rm -f  ~/.local/share/icons/hicolor/scalable/apps/io.github.lgse.Strata.svg
  update-desktop-database ~/.local/share/applications
'
# rm -f /root/.local/bin/strata        # the symlink made in §4
```

(The runtime packages from §2 are shared system libraries and were left installed.)

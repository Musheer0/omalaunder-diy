# Installing VS Code on omalauncher's Arch (ARM)

This system is Arch Linux ARM (aarch64) running inside the omalauncher Android
launcher, entirely as **root**, with a **Wayland-only** display (no X server).
Microsoft ships an official Linux ARM64 build, so VS Code installs without any
AUR or x86 emulation tricks — but two things need handling: VS Code refuses to
run as root, and it must be told to use Wayland.

## 1. Install the runtime libraries

VS Code's tarball ships its own Electron but uses system GTK/NSS.

```sh
pacman -S --needed gtk3 nss nspr libnotify libxkbfile
```

Two hiccups you may hit:

- **`libgbm` "target not found"** — gbm ships in `mesa`, not a `libgbm`
  package. `mesa` is already installed here; just drop it from the list.
- **`bubblewrap: /usr/bin/bwrap exists in filesystem`** — a stray unowned
  `bwrap` is in the way. It is safe to replace with the packaged one:
  ```sh
  pacman -S --needed --overwrite /usr/bin/bwrap bubblewrap
  ```
- **Mirror returns 403** for `extra.db` — a stale/blocked mirror. Retry, or
  switch mirrors in `/etc/pacman.d/mirrorlist`.

## 2. Download and extract the official ARM64 build

```sh
curl -L -o /tmp/code.tar.gz \
  "https://update.code.visualstudio.com/latest/linux-arm64/stable"

mkdir -p /opt/vscode
tar -xzf /tmp/code.tar.gz -C /opt/vscode --strip-components=1
```

The URL is the same API for any platform:
`https://update.code.visualstudio.com/latest/<linux-arm64|linux-x64>/<stable>`.

The tarball's `bin/code` expects `/usr/share/code` layout, but it resolves its
own path from `$0`, so running it from `/opt/vscode` works fine as-is.

## 3. Make it runnable as root, on Wayland

`/opt/vscode/bin/code` aborts when uid is 0 **unless** `--user-data-dir` (or
`--file-write`) is passed. This environment is always root, so bake that into a
wrapper rather than typing it every time:

```sh
cat > /usr/local/bin/code <<'EOF'
#!/bin/sh
# VS Code on omalauncher:
#  - root: bin/code refuses to start without --user-data-dir; --no-sandbox too
#  - no X server, only Wayland: let ozone pick it up via the env var
export ELECTRON_OZONE_PLATFORM_HINT=auto
exec /opt/vscode/bin/code --no-sandbox --user-data-dir=/root/.config/Code "$@"
EOF
chmod 755 /usr/local/bin/code
```

Notes:

- `ELECTRON_OZONE_PLATFORM_HINT=auto` (an Electron env var) is preferred over
  the `--ozone-platform-hint=auto` flag: the flag works but makes every CLI
  run print `Warning: 'ozone-platform-hint' is not in the list of known
  options...`. With the env var the output stays clean and the GPU process
  still comes up with `--ozone-platform=wayland`.
- **Do not symlink `/usr/local/bin/code` to `/opt/vscode/bin/code`.** The
  wrapper and the real launcher have the same basename; a symlink between them
  makes the wrapper exec *itself*, appending its flags forever in a tight
  loop (the process grows an enormous command line and never starts). Keep them
  as two separate regular files.
- If you ever reinstall the tarball over `/opt/vscode`, only `/opt/vscode` is
  replaced; the wrapper in `/usr/local/bin` survives and needs no change.

## 4. Register it with the desktop (drawer / menus)

```sh
mkdir -p /usr/share/icons/hicolor/1024x1024/apps
curl -L -o /usr/share/icons/hicolor/1024x1024/apps/code.png \
  "https://raw.githubusercontent.com/microsoft/vscode/main/resources/linux/code.png"
chmod 644 /usr/share/icons/hicolor/1024x1024/apps/code.png

cat > /usr/share/applications/code.desktop <<'EOF'
[Desktop Entry]
Version=1.0
Type=Application
Name=Visual Studio Code
GenericName=Text Editor
Comment=Code editing. Redefined.
TryExec=/usr/local/bin/code
Exec=/usr/local/bin/code %F
Icon=code
Terminal=false
Categories=Utility;TextEditor;Development;IDE;
Keywords=vscode;code;editor;ide;
StartupNotify=true
StartupWMClass=Code
EOF
chmod 644 /usr/share/applications/code.desktop

desktop-file-validate /usr/share/applications/code.desktop   # optional: sanity check
update-desktop-database /usr/share/applications
gtk-update-icon-cache -q /usr/share/icons/hicolor
```

The tarball contains **no icon** (the `.deb`/`.rpm` builds add it separately),
so fetch the official 1024 px PNG from the VS Code repo as above. From a
widget, start it with the `app("code")` action (README → actions table).

## 5. Verify

CLI — should print three lines and exit 0:

```sh
code --version
```

GUI — start it, wait ~10 s, and look for the full process tree:

```sh
code &
sleep 10
ps -eo args | grep '/opt/vscode/code' | grep -v grep
```

You should see the main process plus `--type=zygote`, `--type=gpu-process`
carrying `--ozone-platform=wayland`, `--type=renderer`, and the
`node.mojom.NodeService` extension-host utilities. The window opens in the
launcher's Wayland session.

Logs (for diagnosing) are in `~/.config/Code/logs/<timestamp>/`. Expected
benign warnings on this platform:

- `Unable to retrieve mac address` — no real NIC format Android exposes.
- `Failed to read /proc/sys/fs/inotify/max_user_watches` — file watching still
  works, with fewer watchers.
- `Extension 'github.copilot-chat' appears in product.json but enables LESS API
  proposals...` — bundled-extension metadata noise.

Stop a test instance with `pkill -f '/opt/vscode/code'`.

## Uninstall

```sh
pkill -f '/opt/vscode/code'
rm -rf /opt/vscode /usr/local/bin/code \
       /usr/share/applications/code.desktop \
       /usr/share/icons/hicolor/1024x1024/apps/code.png \
       /root/.config/Code /root/.vscode
```

(`~/.config/Code` holds settings/extensions — keep it if you plan to reinstall.)

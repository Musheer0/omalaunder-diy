# omalaunder-diy

Do-it-yourself notes for installing desktop apps on **omalauncher's** Arch Linux
ARM system — the aarch64 Arch chroot that runs inside the omalauncher Android
launcher.

Every guide here was written from a real, working install on this system and
covers the parts that break first: root-only sessions, a Wayland-only display
(no X server), non-standard `PATH`s, and the launcher's own app drawer rules.

## Guides

| Guide | What you get |
|---|---|
| [vscode-install.md](vscode-install.md) | Official VS Code ARM64 build in `/opt/vscode`, with a root-safe, Wayland-aware `code` wrapper, icon and `.desktop` entry. |
| [strata-install.md](strata-install.md) | [Strata](https://github.com/lgse/strata) (GTK 4 file manager) installed per-user into `~/.local/bin`, with the root/`sudo` and Wayland caveats worked out, plus launch and uninstall steps. |

## The environment, in short

* **Arch Linux ARM (aarch64)** — `pacman` only; no AUR helpers, no x86
  emulation. Microsoft's and upstream projects' ARM64 builds install cleanly
  from their own tarballs.
* **Runs as root** — `HOME=/root`, and the live `~/.config/omalauncher` tree is
  bind-mounted from the Android app. Several installers (VS Code's, Strata's)
  refuse uid 0 and need handling.
* **Wayland only** — the display is the launcher's Wayland socket; there is no
  X server and usually no session D-Bus. GUI apps must be started from a shell
  in that session (the home screen's `foot` terminal tile).
* **The app drawer lists OWL apps**, not XDG `.desktop` files: entries in
  `~/.config/omalauncher/widgets/` written in the launcher's little language
  (see `~/.config/omalauncher/widgets/README.md`). Registering a `.desktop`
  file is still worth it for menus/tools; to get a drawer icon, add a small OWL
  app that launches the command (`run("code")`, `run("strata")`, …).

## Common snags across the guides

* `pacman -Sy` may 403 on `extra.db` while package downloads still work — a
  mirror issue; fix `/etc/pacman.d/mirrorlist` or rely on an already-synced
  database.
* `~/.local/bin` is not on the default `PATH`; add it to `.bashrc` /
  `.bash_profile`.
* GUI apps launched via `su` usually can't reach the Wayland socket
  (mode `600`, owned by the session user) — run them as the session user.

## Repo layout

```
.
├── README.md          # this file
├── vscode-install.md  # install VS Code (ARM64, root, Wayland)
└── strata-install.md  # install Strata file manager (non-root user)
```

## Adding a guide

Write it the way the existing ones are: exact commands in copy-pasteable
blocks, the failure modes you actually hit (with the error text) and how you
resolved them, a verification section that proves it works, and an uninstall
section. Target path, uid and `PATH` assumptions should be stated up front.

```sh
git add <new-guide>.md
git commit -m "add <app> install guide"
```

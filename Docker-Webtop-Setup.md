# Docker-Webtop Configuration Setup

This document covers all steps taken to configure a `linuxserver/docker-webtop` container running inside an unprivileged LXC with `fuse`, `keyctl`, and `nesting` flags. The container provides a KDE/plasma desktop accessible via Selkies (WebRTC streaming) with nginx frontend.

---

## 1. Base System Update & Packages

```bash
sudo dnf update
sudo dnf upgrade
sudo dnf group install -y "Development Tools"
sudo dnf install nano cmatrix
sudo dnf install kgpg kwallet kwalletmanager
sudo dnf install iproute   # for `ip` command
```

---

## 2. Homebrew Installation

Homebrew provides access to `opencode` and other tools not in the DNF repos.

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"' >> ~/.bash_profile
brew doctor
```

---

## 3. Shell Environment Setup

### `~/.bash_profile`
```bash
eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"

# Source .profile for environment variables (POSIX standard)
if [ -f ~/.profile ]; then
    source ~/.profile
fi

# Source .bashrc for interactive bash settings
if [ -f ~/.bashrc ]; then
    source ~/.bashrc
fi
```

### `~/.profile`
```bash
export PATH="$HOME/.local/bin:$PATH"
export OPENCODE_ENABLE_EXA=1
export GTK_THEME=Adwaita:dark
```

### `~/.bashrc`
```bash
# Source .profile if not already loaded (safety net for non-login shells)
if [ -z "${PROFILE_SOURCED:-}" ] && [ -f ~/.profile ]; then
    source ~/.profile
    export PROFILE_SOURCED=1
fi

export PATH="$HOME/.local/bin:$PATH"
```

**Why `~/.profile` for GTK_THEME:** Variables in `~/.bashrc` are only available in interactive shells. Graphical apps launched via desktop files don't source `~/.bashrc`. Putting `GTK_THEME=Adwaita:dark` in `~/.profile` makes it available to all shells and graphical sessions.

**For proot-apps specifically:** Environment variables set in `~/.profile` must be prepended manually when launching, e.g.:
```bash
GTK_THEME=Adwaita:dark orcaslicer-pa
```
Alternatively, patch the `.desktop` files in the proot-app launcher configs to include the env vars.

---

## 4. OpenCode Installation & Configuration

### Install
```bash
brew install anomalyco/tap/opencode
```

This installs `opencode` v1.14.41. The binary lives at `/home/linuxbrew/.linuxbrew/bin/opencode`.

### Configuration (`~/.config/opencode/opencode.json`)
```json
{
  "$schema": "https://opencode.ai/config.json",
  "permission": {
    "websearch": "allow",
    "edit": "allow",
    "webfetch": "allow",
    "glob": "allow",
    "bash": "allow",
    "write": "allow",
    "grep": "allow"
  }
}
```

### npm Dependency (`~/.config/opencode/package.json`)
```json
{
  "dependencies": {
    "@opencode-ai/plugin": "1.14.41"
  }
}
```

### AGENTS.md (`~/.config/opencode/AGENTS.md`)

Provides OpenCode with context about the container environment. See file at `~/.config/opencode/AGENTS.md` for full contents.

---

## 5. Proot-Apps Installation

`proot-apps` runs GUI applications in a PROOT-based namespace inside the container. Installed apps:

```bash
proot-apps install brave
proot-apps install braveorigin      # Brave Origin Nightly
proot-apps install vlc
proot-apps install filezilla
proot-apps install orcaslicer
proot-apps install remmina
proot-apps install vscodium
proot-apps update
```

### Brave SingletonLock Fix

If Brave refuses to start due to leftover lock files:
```bash
rm ~/.config/BraveSoftware/Brave-Browser/SingletonLock
rm ~/.config/BraveSoftware/Brave-Origin-Nightly/SingletonLock
```

### OrcaSlicer SSH Key

For connecting to printers over SSH, copy your key into the proot-apps namespace:
```bash
mkdir -p proot-apps/ghcr.io_linuxserver_proot-apps_orcaslicer/home/ubuntu/.ssh
cp ~/.ssh/id_ed25519 proot-apps/ghcr.io_linuxserver_proot-apps_orcaslicer/home/ubuntu/.ssh/
chmod 600 proot-apps/ghcr.io_linuxserver_proot-apps_orcaslicer/home/ubuntu/.ssh/id_ed25519
```

---

## 6. GPU & Device Permissions

### Problem

`/dev/dri/renderD128` and `/dev/dri/card1` are not accessible to the `abc` user inside the container. This is because the container runs in an unprivileged LXC where container GIDs are offset by a kernel mapping — the `abc` user inside the container does not share GIDs with the `video` (26) and `render` (303) groups on the host.

Workaround: Set `ATTACHED_DEVICES_PERMS` env var on the docker run/compose to include GIDs 26 and 303.

### Boot-Time Init Scripts

The container uses **s6-overlay**. Custom init scripts in `/config/custom-cont-init.d/` execute at boot via the `init-custom-files` service. These scripts run after `init-mods-end` but before `init-services`, so group memberships are active before any user-facing services start.

> **Important:** `/config/` is the container's persistent storage mount point. Files in `/config/custom-cont-init.d/` persist across container restarts because `/config` is bind-mounted from the host.

#### `/config/custom-cont-init.d/10-gpu-groups`

Adds `abc` user to GPU device groups (GID 26, 303) on every boot:

```bash
#!/bin/bash
for gid in 26 303; do
    gname=$(getent group "${gid}" | cut -d: -f1)
    if [[ -z "${gname}" ]]; then
        gname="group${gid}"
        if ! getent group "${gname}" >/dev/null 2>&1; then
            groupadd -g "${gid}" "${gname}" 2>/dev/null || true
        fi
        gname=$(getent group "${gid}" | cut -d: -f1)
    fi
    if [[ -n "${gname}" ]] && ! id -G abc | grep -qw "${gid}"; then
        usermod -a -G "${gname}" abc
        echo "[custom-init] Added user abc to group ${gname} (GID ${gid})"
    fi
done
```

#### `/config/custom-cont-init.d/99-gpu-permissions`

Fallback: make DRI devices world-readable/writable:

```bash
#!/bin/bash
chmod 666 /dev/dri/card1
chmod 666 /dev/dri/renderD128
```

#### `/config/custom-cont-init.d/99-gpu-setup`

GPU setup with GStreamer VAAPI support:

```bash
#!/bin/bash
chmod 666 /dev/dri/card1 /dev/dri/renderD128

if ! gst-inspect-1.0 vaapi 2>/dev/null | grep -q "plugin"; then
    echo "[gpu-setup] Installing gstreamer1-vaapi..."
    dnf5 install -y gstreamer1-vaapi 2>&1 || true
fi

if [ -r /dev/dri/renderD128 ]; then
    echo "[gpu-setup] GPU render node accessible"
else
    echo "[gpu-setup] WARNING: GPU render node NOT accessible"
fi
```

#### `/config/custom-cont-init.d/00-debug-env`

Dumps environment at init time for debugging:

```bash
#!/bin/bash
env | sort > /tmp/init-env-debug.log
echo "DRI_NODE=$DRI_NODE" >> /tmp/init-env-debug.log
echo "DRINODE=$DRINODE" >> /tmp/init-env-debug.log
echo "PWD=$PWD" >> /tmp/init-env-debug.log
```

### Manual GPU Fix (Without Reboot)

```bash
sudo usermod -a -G $(getent group 26 | cut -d: -f1) abc
sudo usermod -a -G $(getent group 303 | cut -d: -f1) abc
# Then re-login or restart the container
```

Verify:
```bash
id abc
ls -la /dev/dri/
```

---

## 7. SSH Configuration

### Key Generation
```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519
```

### Known Hosts
SSH connections configured for:
- `v@10.0.2.49` (Forgejo runner host)
- `v@10.0.2.51`
- `root@hermes.mgmt` / `v@hermes.mgmt`
- `root@container-nix.mgmt` / `v@container-nix.mgmt`
- `v@worknix.mgmt`

---

## 8. Desktop Environment

- **DE:** KDE Plasma (KWin Wayland)
- **Streaming:** Selkies (WebRTC)
- **Web frontend:** nginx
- **Audio:** PulseAudio
- **X settings:** xsettingsd

### Debugging Desktop Crashes

If applications close unexpectedly:
```bash
# Check s6 service status
sudo s6-svstat /run/service/svc-de
sudo s6-svstat /run/service/svc-xorg

# Check desktop logs
sudo tail -100 /run/service/svc-de/log/current

# Check for death tally (restart count)
sudo cat /run/service/svc-de/log/death_tally

# Check s6 uncaught logs
sudo strings /var/log/s6-uncaught-logs/current

# Check running display processes
ps aux | grep -E "kwin_wayland|plasmashell|startwm"

# Check Wayland socket
stat /config/.XDG/wayland-0 /config/.XDG/wayland-1
```

---

## 9. Immich (Downloaded)

Docker Compose files downloaded to `/config/Downloads/`:
- `docker-compose.yml` — rootful deployment
- `docker-compose.rootless.yml` — rootless using uid 1000:1000

---

## 10. S6-Overlay Init Services Reference

| Service | Purpose |
|---------|---------|
| `init-adduser` | Creates `abc` user with PUID/PGID |
| `init-custom-files` | Executes scripts from `/config/custom-cont-init.d/` |
| `init-device-perms` | Handles GPU/device permissions via `ATTACHED_DEVICES_PERMS` |
| `init-envfile` | Processes environment files |
| `init-nginx` | Nginx web server setup |
| `init-selkies` | Selkies desktop streaming init |
| `init-selkies-config` | Selkies configuration |
| `init-selkies-end` | Post-selkies initialization |
| `init-video` | Video/graphics init |
| `svc-de` | Desktop environment |
| `svc-xorg` | X11/Wayland display |
| `svc-selkies` | Streaming server |
| `svc-nginx` | Web server |
| `svc-pulseaudio` | Audio |
| `svc-docker` | Docker-in-Docker |
| `svc-cron` | Cron scheduler |
| `svc-watchdog` | Watchdog monitor |
| `svc-xsettingsd` | X settings daemon |

---

## 11. Known Issues & Workarounds

| Issue | Workaround |
|-------|-----------|
| `Permission denied` on `/dev/dri/renderD128` | `ATTACHED_DEVICES_PERMS` env var with GIDs 26,303 + init scripts above |
| `abc` not in groups after reboot | `/config/custom-cont-init.d/10-gpu-groups` script |
| Brave SingletonLock error | `rm ~/.config/BraveSoftware/Brave-Browser/SingletonLock` |
| Proot-apps not reading shell env vars | Put vars in `~/.profile`, not `~/.bashrc`; patch `.desktop` files |
| Desktop/window manager crashing | Check s6 logs; container restart usually resolves |
| `GTK_THEME` not applied | Ensure it's in `~/.profile` — graphical sessions source login shell profile |

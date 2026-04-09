# Docker Storage Driver for Docker in LXC on Proxmox

## Recommendation

For Docker running inside an **LXC container on Proxmox**, the best practical default is usually **`fuse-overlayfs`**.

It is the safest choice when you want Docker to run inside an **unprivileged, nested LXC** without depending on native OverlayFS behavior inside that container.

## Why `fuse-overlayfs`

- Works well in **unprivileged** and **nested** container setups
- Avoids many of the permission and kernel capability issues that can affect native OverlayFS in LXC
- More efficient than `vfs`, which should generally be treated as a fallback only

## When `overlay2` May Be Fine

If your specific Proxmox host, LXC config, and backing storage support native OverlayFS cleanly, `overlay2` can work.

That said, for **Docker inside LXC**, **`fuse-overlayfs` is still the safer default recommendation** unless you have already verified `overlay2` works reliably in that exact setup.

## Proxmox LXC Requirements

Create the container as **unprivileged** and enable these LXC features:

- `nesting=1`
- `keyctl=1`

If you plan to use `fuse-overlayfs`, also enable:

- `fuse=1`

A typical Proxmox LXC config line looks like:

```ini
features: fuse=1,keyctl=1,nesting=1
```

## Setup Steps

### 1. Enable required LXC features on Proxmox

On the Proxmox host, edit the container config:

```bash
nano /etc/pve/lxc/<CTID>.conf
```

Add or update:

```ini
unprivileged: 1
features: fuse=1,keyctl=1,nesting=1
```

Then restart the container.

### 2. Install Docker and `fuse-overlayfs` inside the LXC

Inside the container:

```bash
apt update
apt install -y docker.io fuse-overlayfs
```

If your distro uses a separate FUSE 3 package, install it as well if needed.

### 3. Configure Docker to use `fuse-overlayfs`

Create or edit:

```bash
nano /etc/docker/daemon.json
```

Add:

```json
{
  "storage-driver": "fuse-overlayfs"
}
```

### 4. Restart Docker

```bash
systemctl restart docker
```

### 5. Verify

```bash
docker info
```

You should see:

```text
Storage Driver: fuse-overlayfs
```

## Notes for Proxmox Users

- `vfs` is usually a last resort because it is slower and uses much more disk space.
- `overlay2` may work in some Proxmox LXC setups, especially on newer kernels and supported backing filesystems, but it should be treated as **verified-per-container**, not assumed.
- If Docker fails to start with storage-driver errors, confirm the LXC has `fuse=1`, `nesting=1`, and `keyctl=1`, then restart both the container and Docker.

## Rule of Thumb

- **Use `fuse-overlayfs`** for Docker in Proxmox LXC
- **Use `overlay2`** only if you have confirmed native OverlayFS works in that exact container
- **Avoid `vfs`** unless nothing else works

## Summary

For a typical Proxmox homelab LXC deployment, **`fuse-overlayfs` is the recommended default** because it is the most reliable option for Docker in an unprivileged nested container.

## Sources

- Docker notes that `overlay2` is now considered a legacy storage driver relative to the newer containerd image store and overlayfs snapshotter: https://docs.docker.com/engine/storage/drivers/overlayfs-driver/
- Docker storage driver selection guidance: https://docs.docker.com/engine/storage/drivers/select-storage-driver/
- `fuse-overlayfs` project requirements and usage notes: https://github.com/containers/fuse-overlayfs
- Proxmox community references showing common LXC Docker requirements such as `nesting=1`, `keyctl=1`, and often `fuse=1`: https://forum.proxmox.com/threads/lxc-zfs-docker-overlay2-driver.122621/ and https://forum.proxmox.com/threads/lxc-with-nested-docker-overlayfs-upper-fs-missing-required-features.102362/

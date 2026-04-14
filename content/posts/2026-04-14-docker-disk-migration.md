---
title: "Docker Data Root Migration to External Disk"
date: 2026-04-14
tags: [docker, elasticsearch, disk, ntfs, ext4, loop-mount]
summary: "Elasticsearch hit the flood-stage disk watermark on a 32GB root partition. Solved by creating a loop-mounted ext4 image on an NTFS NVMe drive to relocate Docker's data-root."
---

## Problem

Elasticsearch `listings_general` index hit the **flood-stage disk watermark** (95%+) on root partition `/dev/sda2` (32GB, 94% used). ES automatically set the index to read-only-allow-delete, blocking all writes (indexing new listings failed with `cluster_block_exception`).

Root cause: Docker data (`/var/lib/docker`) lived on the small root partition.

---

## Options Considered

| Option | Description | Pros | Cons |
|--------|-------------|------|------|
| A | Shrink NTFS on NVMe, create new ext4 partition | Clean, native performance | Risk of data loss during partition resize |
| B | Format NVMe partition as ext4 | Simplest, best performance | Destroys existing Windows data |
| C | Create ext4 image file on NTFS, loop-mount it | Preserves Windows data, no repartitioning | Slight I/O overhead from loop device + NTFS layer |

---

## Choice: Option C

**Reason**: NVMe SSD (`/dev/nvme0n1p2`, 476GB) contained Windows files that could not be lost. Option C allows using the disk for Docker without touching existing data.

---

## Steps Executed

### 1. Install NTFS support
```bash
sudo pacman -S ntfs-3g
```
- `ntfs-3g`: userspace NTFS driver for Linux, enables read/write access to NTFS partitions.

### 2. Mount the NVMe NTFS partition
```bash
sudo mkdir -p /mnt/nvme
sudo mount -t ntfs-3g /dev/nvme0n1p2 /mnt/nvme
```
- `mkdir -p`: create directory and any missing parents; no error if it already exists.
- `mount -t ntfs-3g`: mount using the NTFS-3G driver.

### 3. Create a 100GB image file
```bash
sudo dd if=/dev/zero of=/mnt/nvme/docker.img bs=1M count=102400 status=progress
```
- `dd`: low-level byte copying tool.
- `if=/dev/zero`: input is an infinite stream of zero bytes.
- `of=...`: output file path.
- `bs=1M`: block size of 1 megabyte.
- `count=102400`: write 102400 blocks = 100GB.
- `status=progress`: show transfer progress during copy.

### 4. Format the image as ext4
```bash
sudo mkfs.ext4 /mnt/nvme/docker.img
```
- `mkfs.ext4`: create an ext4 filesystem inside the image file. This does NOT affect the NTFS partition — it only formats the contents of the image file.

### 5. Loop-mount the image
```bash
sudo mkdir -p /mnt/docker
sudo mount -o loop /mnt/nvme/docker.img /mnt/docker
```
- `mount -o loop`: treat a regular file as a block device (loop device) and mount it. The kernel creates `/dev/loopN` automatically.

### 6. Stop Docker and copy data
```bash
docker stop $(docker ps -q)
sudo systemctl stop docker
sudo systemctl stop docker.socket
sudo rsync -aP /var/lib/docker/ /mnt/docker/
```
- `docker ps -q`: list running container IDs (quiet mode).
- `docker stop $(...)`: stop all running containers before stopping the daemon.
- `systemctl stop docker.socket`: stop the Docker socket (prevents auto-restart of daemon).
- `rsync -aP`: archive mode (`-a` preserves permissions, ownership, timestamps, symlinks) + progress (`-P`).

### 7. Configure Docker to use new data root
```bash
sudo tee /etc/docker/daemon.json <<'EOF'
{
  "data-root": "/mnt/docker"
}
EOF
```
- `daemon.json`: Docker daemon configuration file.
- `data-root`: tells Docker where to store images, containers, volumes, etc.

### 8. Start Docker and verify
```bash
sudo systemctl start docker
docker info | grep "Docker Root Dir"
# Output: Docker Root Dir: /mnt/docker
df -h /mnt/docker
# Output: /dev/loop0  98G  1.3G  92G  2%  /mnt/docker
```

### 9. Persist mounts across reboots
```bash
sudo blkid /dev/nvme0n1p2
# UUID=26CCDAA5CCDA6F15

echo "UUID=26CCDAA5CCDA6F15 /mnt/nvme ntfs-3g defaults,nofail 0 0" | sudo tee -a /etc/fstab
echo "/mnt/nvme/docker.img /mnt/docker ext4 loop,defaults,nofail 0 0" | sudo tee -a /etc/fstab
```
- `/etc/fstab`: filesystem table; defines mounts that are applied at boot.
- `nofail`: system continues booting even if this mount fails (e.g., disk not plugged in).
- `0 0`: no dump backup, no fsck ordering.

### 10. Fix Elasticsearch index block
```bash
curl -X PUT "localhost:9200/listings_general/_settings" \
  -H 'Content-Type: application/json' \
  -d '{"index.blocks.read_only_allow_delete": null}'
```
- Removes the read-only block that ES applied when disk was full.

### 11. (Optional) Clean up old Docker data
```bash
sudo rm -rf /var/lib/docker
```
- Frees space on root partition after confirming Docker works from the new location.

---

## Key Concepts

| Concept | Explanation |
|---------|-------------|
| **Loop device** | A pseudo-device that makes a file accessible as a block device (`/dev/loopN`). Allows mounting filesystem images without dedicated partitions. |
| **Flood-stage watermark** | Elasticsearch disk threshold (default 95%). When exceeded, indices become read-only to prevent cluster instability. |
| **data-root** | Docker config option that sets the base directory for all Docker state (images, containers, volumes, networks). Default: `/var/lib/docker`. |
| **fstab** | `/etc/fstab` — static file system information. The OS reads this at boot to auto-mount filesystems. |
| **nofail** | fstab option that prevents boot failure if a mount point is unavailable. |

---

## Disk Layout After Migration

```
/dev/sda2 (32G, root /)        -> OS + apps (no longer holds Docker data)
/dev/nvme0n1p2 (476G, NTFS)    -> mounted at /mnt/nvme (Windows files preserved)
  └── docker.img (100G, ext4)  -> loop-mounted at /mnt/docker (Docker data-root)
```

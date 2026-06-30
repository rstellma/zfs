Type | Version/Name
--- | ---
Distribution Name	| OpenSuSE Leap
Distribution Version	| 16.0
Kernel Version	| 6.12.0-160000.35-default
Architecture	| x86\_64
OpenZFS Version	| zfs-2.4.3-1

<!-- toc -->
- [1. CREATE POOL](#1-create-pool)
  - [1.1. Create Pool with custom mountpoint](#11-create-pool-with-custom-mountpoint)
  - [1.2. Encrypted base dataset](#12-encrypted-base-dataset)
  - [1.3. Dataset tuning](#13-dataset-tuning)
- [2. MOUNTING DATASETS](#2-mounting-datasets)
  - [2.1. Adjust permissions for unprivileged user](#21-adjust-permissions-for-unprivileged-user)
  - [2.2. Mount directly to $HOME](#22-mount-directly-to-home)
  - [2.3. Mount after user login](#23-mount-after-user-login)
- [3. CHECKS TO PERFORM](#3-checks-to-perform)
  - [3.1. zpool list](#31-zpool-list)
  - [3.2. zpool status](#32-zpool-status)
  - [3.3. zfs list](#33-zfs-list)
  - [3.4. Error correction from 3. II.](#34-error-correction-from-3-ii)
    - [3.4.1. Shortcut](#341-shortcut)
- [4. ADJUST READ/WRITE CACHE SIZE](#4-adjust-readwrite-cache-size)
  - [4.1. /etc/modprobe.d/zfs.conf](#41-etcmodprobedzfsconf)
  - [4.2. Rebuild initrd](#42-rebuild-initrd)
- [5. EXPANDING THE POOL](#5-expanding-the-pool)
  - [5.1. Add new disk](#51-add-new-disk)
  - [5.2. Rewrite data to wider parity](#52-rewrite-data-to-wider-parity)
  - [5.3. Throttling ZFS to maintain system responsiveness during operation](#53-throttling-zfs-to-maintain-system-responsiveness-during-operation)
  - [5.4. Discrepancy between "zpool list" and "zfs list"](#54-discrepancy-between-zpool-list-and-zfs-list)
- [6. SNAPSHOTS](#6-snapshots)
  - [6.1. Creating daily snapshots](#61-creating-daily-snapshots)
    - [6.1.1. `bash` script](#611-bash-script)
    - [6.1.2. Cron job](#612-cron-job)
  - [6.2. Rolling back a snapshot](#62-rolling-back-a-snapshot)
    - [6.2.1. Rolling back the latest snapshot](#621-rolling-back-the-latest-snapshot)
    - [6.2.2. Roll back to an older snapshot](#622-roll-back-to-an-older-snapshot)
  - [6.3. Create a snapshot manually](#63-create-a-snapshot-manually)
  - [6.4. Restore individual files from a snapshot](#64-restore-individual-files-from-a-snapshot)
- [7. FAQ / Known Issues](#7-faq--known-issues)
- [8. ZFS BEST PRACTICES FOR HOME LABS](#8-zfs-best-practices-for-home-labs)
  - [8.1. Workload-based ZFS dataset tuning (copy-paste template)](#81-workload-based-zfs-dataset-tuning-copy-paste-template)
    - [8.1.1. Containers (distrobox / podman / incus root filesystem)](#811-containers-distrobox--podman--incus-root-filesystem)
    - [8.1.2. Container logs](#812-container-logs)
    - [8.1.3. VirtualBox / VM disk images (.vdi)](#813-virtualbox--vm-disk-images-vdi)
  - [8.2. General rule of thumb](#82-general-rule-of-thumb)
  - [8.3. Important ZFS behavior](#83-important-zfs-behavior)
<!-- tocstop -->


# 1. CREATE POOL

## 1.1. Create Pool with custom mountpoint

```bash
sudo zpool create -f -o ashift=12 -m /storage storage raidz1 /dev/disk/by-id/foo /dev/disk/by-id/bar /dev/disk/by-id/baz
```

## 1.2. Encrypted base dataset

> \[!IMPORTANT]
> Passphrase is requested here

```bash
sudo zfs create -o encryption=aes-256-gcm -o keyformat=passphrase storage/secured
```

## 1.3. Dataset tuning

```bash
sudo zfs create -o compression=zstd-3 -o atime=off -o recordsize=128k storage/secured/distrobox
sudo zfs create -o compression=zstd-3 -o atime=off -o recordsize=64k storage/secured/incus
sudo zfs create -o compression=zstd-3 -o atime=off -o recordsize=64k storage/secured/podman
sudo zfs create -o compression=zstd-5 -o atime=off -o recordsize=1M -o sync=disabled storage/secured/backup
sudo zfs create -o compression=zstd-1 -o atime=off -o recordsize=1M -o sync=disabled storage/secured/vbox
```

\[↑ Back to table of contents]\(#table of contents)

# 2. MOUNTING DATASETS

## 2.1. Adjust permissions for unprivileged user

```bash
sudo chown -R <user>:<group> /storage/secured
sudo chmod 700 /storage/secured

sudo chown -R <user>:vboxusers /storage/secured/vbox
sudo chmod 770 /storage/secured/vbox
```

## 2.2. Mount directly to $HOME

```bash
sudo zfs set mountpoint=/home/<user>/__data/backup storage/secured/backup
sudo zfs set mountpoint=/home/<user>/__data/distrobox storage/secured/distrobox
sudo zfs set mountpoint=/home/<user>/__data/incus storage/secured/incus
sudo zfs set mountpoint=/home/<user>/__data/podman storage/secured/podman
sudo zfs set mountpoint=/home/<user>/__data/vbox storage/secured/vbox
```

## 2.3. Mount after user login

> \[!IMPORTANT]
> Datasets are not mounted automatically!

> \[!WARNING]
> Starting VirtualBox, Podman, Incus, etc., would result in writing directly to the partition holding /home (depending on configuration). We do not want that.

> \[!TIP]
> Create a bash alias to save typing.

```bash
alias zfs-mount='sudo zfs load-key storage/secured; sudo zfs mount -a'
```

[↑ Back to Table of Contents](#table-of-contents)

# 3. CHECKS TO PERFORM

## 3.1. zpool list

```bash
NAME SIZE ALLOC FREE CKPOINT EXPANDSZ FRAG CAP DEDUP HEALTH ALTROOT
storage 10.9T 2.66T 8.25T - - 0% 24% 1.00x ONLINE -
```

## 3.2. zpool status

```bash
pool: storage 
state: ONLINE
status: One or more devices has experienced an unrecoverable error. To 
attempt was made to correct the error. Applications are unaffected.
action: Determine if the device needs to be replaced, and clear the errors 
using 'zpool clear' or replace the device with 'zpool replace'. 
see: https://openzfs.github.io/openzfs-docs/msg/ZFS-8000-9P 
scan: scrub repaired 0B in 01:26:08 with 1 errors on Sat Jun 27 23:33:05 2026
expand: expanded raidz1-0 copied 2.71T in 03:12:16, on Sat Jun 27 22:06:57 2026
config: 

NAME STATE READ WRITE CKSUM 
storage ONLINE 0 0 0 
raidz1-0 ONLINE 0 0 0 
ata-TOSHIBA_HDWD130_Y28GXZ9AS ONLINE 0 0 4 
ata-TOSHIBA_HDWD130_Y28GYBSAS ONLINE 0 0 4 
ata-TOSHIBA_HDWD130_Y28GVHGAS ONLINE 0 0 2 
ata-TOSHIBA_HDWD130_Y28GXUYAS ONLINE 0 0 2

errors: No known data errors
```

## 3.3. zfs list

```bash
NAME                        USED  AVAIL  REFER  MOUNTPOINT
storage                    1.77T  5.37T   128K  /storage
storage/secured            1.77T  5.37T   234K  /storage/secured
storage/secured/backup     1.51T  5.37T  1.51T  $HOME/__data/backup
storage/secured/distrobox   828M  5.37T   828M  $HOME/__data/distrobox
storage/secured/incus      13.3G  5.37T  13.3G  $HOME/__data/incus
storage/secured/podman     1.50G  5.37T  1.50G  $HOME/__data/podman
storage/secured/vbox        247G  5.37T   247G  $HOME/__data/vbox
```

## 3.4. Error correction from 3. II.

> \[!IMPORTANT]
> Only necessary if errors occur

> \[!TIP]
> Restore the faulty file(s) from backup or delete them entirely

* Reset ZFS error log

```bash
sudo zpool clear storage
```

* Force ZFS scrub (mandatory)

```bash
sudo zpool scrub storage
```

### 3.4.1. Shortcut

> \[!CAUTION]
> Not recommended; quick and dirty

* Start ZFS scrub
* Cancel ZFS scrub

```bash
sudo zpool scrub -s storage
```

* Clear ZFS error log

[↑ Back to Table of Contents](#table-of-contents)

# 4. ADJUST READ/WRITE CACHE SIZE

## 4.1. /etc/modprobe.d/zfs.conf

```bash
# Minimum cache: 2 GB (2147483648 bytes)
options zfs zfs_arc_min=2147483648

# Maximum cache: 6 GB (6442450944 bytes)
options zfs zfs_arc_max=6442450944
```

## 4.2. Rebuild initrd

```bash
sudo dracut -f --regenerate-all
```

[↑ Back to Table of Contents](#table-of-contents)

# 5. EXPANDING THE POOL

## 5.1. Add new disk

```bash
sudo zpool attach storage storage-0 /dev/disk/by-id/new-disk
```

## 5.2. Rewrite data to wider parity

The command must always be applied to a filesystem path (files or folders), not the logical dataset name.

```bash
zfs rewrite -rv $HOME/__data/<mount_point>
```

## 5.3. Throttling ZFS to maintain system responsiveness during operation

The default value is often 10. We throttle this significantly to a maximum of 2 concurrent I/Os per disk:

> \[!NOTE]
> /sys/module/zfs/parameters/zfs\_vdev\_async\_write\_min\_active

```
sysctl -w kstat.zfs.misc.zio.zio_vdev_async_write_max_active=2
```

Additionally, we limit the amount of unsaved data in RAM to prevent system "stuttering":

> \[!NOTE]
> /sys/module/zfs/parameters/zfs\_vdev\_async\_write\_max\_active

```
sysctl -w kstat.zfs.misc.zio.zio_vdev_async_write_min_active=1
```

> \[!IMPORTANT]
> Revert these settings after the 'rewrite'.

## 5.4. Discrepancy between "zpool list" and "zfs list"

> \[!IMPORTANT]
> At the time of writing (June 2026), there is a discrepancy regarding the reported available free space.

A ticket has been created in the OpenZFS bug tracker: https://github.com/openzfs/zfs/issues/18711

[↑ Back to Table of Contents](#table-of-contents)

# 6. SNAPSHOTS

## 6.1. Creating daily snapshots

### 6.1.1. `bash` script

```bash
#!/bin/bash

# 1. Hier genau eintragen, welche Datasets gesichert werden sollen (mit Leerzeichen getrennt)
DATASETS=(
    "storage/secured/foo"
    "storage/secured/bar"
    "storage/secured/baz"
)

DATE=$(date +%Y-%m-%d)
RETENTION_DAYS=30

# Schleife durch alle definierten Datasets
for DS in "${DATASETS[@]}"; do
    # Prüfen, ob das Dataset überhaupt existiert
    if zfs list "$DS" >/dev/null 2>&1; then
        
        # Snapshot für das einzelne Dataset erstellen (ohne -r)
        zfs snapshot "${DS}@daily-${DATE}"
        
        # Alte Snapshots für dieses spezifische Dataset löschen
        zfs list -t snapshot -o name -s creation | grep "^${DS}@daily-" | while read -r SNAPSHOT; do
            SNAP_DATE=$(echo "$SNAPSHOT" | sed "s/.*@daily-//")
            SNAP_EPOCH=$(date -d "$SNAP_DATE" +%s 2>/dev/null)
            NOW_EPOCH=$(date +%s)
            
            if [ -n "$SNAP_EPOCH" ]; then
                AGE_DAYS=$(( (NOW_EPOCH - SNAP_EPOCH) / 86400 ))
                if [ "$AGE_DAYS" -ge "$RETENTION_DAYS" ]; then
                    zfs destroy "$SNAPSHOT"
                fi
            fi
        done
    fi
done
```

### 6.1.2. Cron job

* Scheduled

```
0 2 * * * /path/to/script
```

* @daily (corresponds to 00:00)

## 6.2. Rolling back a snapshot

### 6.2.1. Rolling back the latest snapshot

Format from "/root/zfs-daily-snapshot.sh"

```bash
sudo zfs rollback storage/secured/vbox@daily-<YYYY>-<MM>-<DD>
```

### 6.2.2. Roll back to an older snapshot

> \[!CAUTION]
> deletes all snapshots from YYYY-MM-DD to the present

```bash
sudo zfs rollback -r storage/secured/vbox@daily-<YYYY>-<MM>-<DD>
```

## 6.3. Create a snapshot manually

(e.g., before an update or configuration changes)

```bash
sudo zfs snapshot storage/secured/vbox@pre-update
```

Everything went well?

```bash
sudo zfs destroy storage/secured/vbox@pre-update
```

Something went wrong?

```
sudo zfs rollback storage/secured/vbox@pre-update
```

## 6.4. Restore individual files from a snapshot

Copy the file from

```
<mountpoint>/.zfs/<YYYY>-<MM>-<DD>
```

back to its original location

[↑ Back to Table of Contents](#table-of-contents)

# 7. FAQ / Known Issues

Why do `zpool list`, `zfs list` and `df -h` show different values?

* `zpool list` reports the physical pool capacity, while `zfs list` and `df` report the logically available space for datasets. These values are not expected to match. After a RAIDZ expansion, the difference may be larger than expected due to current OpenZFS space accounting. See issue [#18711](https://github.com/openzfs/zfs/issues/18711). for details.

When should I run `zfs rewrite`?

* A rewrite is only necessary after changing dataset properties that affect newly written data, such as `recordsize` or `compression`, or after a RAIDZ expansion if you want existing data to benefit from the new stripe width. Simply changing the property does not rewrite existing blocks.

What happens if I replace one drive with a larger one?

* The additional capacity is not available immediately. A RAIDZ vdev can only grow once all drives have been replaced with the larger size (unless you are using the new RAIDZ expansion feature to add disks). Until then, the extra space on the larger drive remains unused.

Which dataset properties affect existing data?

* Most dataset properties only affect future writes. Existing blocks keep their original layout until they are rewritten. This includes properties such as `compression`, `recordsize` and `checksum`.

Why is my pool almost full at 80%?

* Unlike traditional filesystems, ZFS performance can degrade significantly when pools become too full. Keeping at least 20% free space is a commonly recommended guideline, especially for pools hosting virtual machines or databases.

Why does changing recordsize or compression not change existing files?

* Because these properties are applied when blocks are written. Existing data keeps its original layout until it is copied, rewritten or recreated.

[↑ Back to Table of Contents](#table-of-contents)

# 8. ZFS BEST PRACTICES FOR HOME LABS

## 8.1. Workload-based ZFS dataset tuning (copy-paste template)

ZFS performance depends heavily on matching dataset properties to workload patterns.

Below are practical presets for common home-lab use cases.

### 8.1.1. Containers (distrobox / podman / incus root filesystem)

Best for: many small files, frequent metadata operations, random I/O

```bash
zfs create \
  -o compression=zstd-3 \
  -o atime=off \
  -o recordsize=64K \
  storage/secured/containers
```

Notes:

* 64K improves small-file and metadata-heavy workloads
* zstd-3 is a good balance of speed and compression
* atime=off reduces unnecessary writes

### 8.1.2. Container logs

Best for: append-heavy text logs, high write rate, highly compressible data

```bash
zfs create \
  -o compression=zstd-3 \
  -o atime=off \
  -o recordsize=128K \
  storage/secured/logs
```

Notes:

* sequential writes benefit from larger recordsize
* logs compress extremely well
* minimal tuning required beyond compression + atime

### 8.1.3. VirtualBox / VM disk images (.vdi)

Best for: random block I/O, virtual disk workloads

```bash
zfs create \
  -o compression=zstd-1 \
  -o atime=off \
  -o recordsize=64K \
  -o sync=disabled \
  storage/secured/vbox
```

Notes:

* VM workloads behave like block devices
* 64K reduces write amplification
* sync=disabled improves performance (only if safe for your risk model)
* zstd-1 avoids CPU overhead under heavy I/O

## 8.2. General rule of thumb

* Small random I/O → 64K recordsize
* Sequential data → 128K or larger
* Always disable atime unless specifically needed
* Compression is almost always beneficial (lz4 or zstd)

## 8.3. Important ZFS behavior

* Dataset property changes only affect new writes
* Existing data is not rewritten automatically
* Use `zfs rewrite` (or copy/rewrite) if you want old data to benefit

[↑ Back to Table of Contents](#table-of-contents)

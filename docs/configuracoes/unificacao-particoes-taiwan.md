# taiwan — Disk Partition Unification: Post-Change Report

> **Purpose:** context file to hand to any AI assistant helping with the Linux computer **taiwan**.
> **Status date:** 2026-11-17 (state read live from the machine on this date).
> **Owner:** Igor (non-technical Linux learner; prefers plain-language, step-by-step answers with a glossary; uses `gedit`, not `nano`).

---

## 1. TL;DR (read this first)

- On **2026-11-16** the internal NVMe disk of `taiwan` was reduced from **6 partitions (Windows dual boot + shared NTFS data partition)** to **2 partitions: EFI + one ext4 Ubuntu partition (1.8 TB)**.
- **Windows was removed.** Ubuntu was **not** reinstalled or reformatted; its partition was moved and grown with GParted Live.
- The old NTFS data partition `taiwan_com` no longer exists. **`/mnt/taiwan_com` is now a plain folder on the root filesystem** holding the same data (restored from backup), so every script/app path that used it still works.
- **Do not assume `/mnt/taiwan_com` is a mount point anymore.** Anything that tests for a mount (`mountpoint -q`, `ConditionPathIsMountPoint=`) now fails — see §7 #1 (fixed on 2026-11-17 for the known units and scripts).
- Old guideline *"install new content on shared partitions instead of the Linux partition"* is **obsolete**: there is only one partition. `/mnt/taiwan_com` remains the conventional place for large data.

---

## 2. Machine

| Item | Value |
|---|---|
| Hostname | `asus-taiwan` (called "taiwan") |
| OS | Ubuntu 24.04.5 LTS |
| Kernel | 7.0.0-43-generic |
| User | `icm` (uid/gid 1000) |
| Disk | `nvme0n1`, 1.8 TiB NVMe, GPT, UEFI boot |
| Backup disk (external, USB) | `ssd-japao` (953.9 GB), mounts at `/media/icm/ssd-japao` when plugged in |

---

## 3. Partition layout — before vs. after

### Before (until 2026-11-16)

| Partition | FS | Label | Size | Role | Mount |
|---|---|---|---|---|---|
| `nvme0n1p1` | vfat | — | 100 MiB | EFI System Partition | `/boot/efi` |
| `nvme0n1p2` | (none) | — | 16 MiB | Microsoft Reserved | — |
| `nvme0n1p3` | ntfs | `taiwan_win` | 124 GiB | Windows (dual boot) | `/mnt/taiwan_win` (noauto) |
| `nvme0n1p4` | ntfs | — | 518 MiB | Windows Recovery | — |
| `nvme0n1p5` | ext4 | `taiwan_ubu` | ~120 GB (98–99% full) | Ubuntu root | `/` |
| `nvme0n1p6` | ntfs | `taiwan_com` | 1.58 TiB | Shared data partition | `/mnt/taiwan_com` (ntfs3) |

### After (current)

| Partition | FS | Label | UUID | Size | Mount |
|---|---|---|---|---|---|
| `nvme0n1p1` | vfat | — | `5E9B-4ABB` | 100 MiB | `/boot/efi` |
| `nvme0n1p5` | ext4 | `taiwan_ubu` | `6a4845fa-114f-457a-b228-d226dba78c48` | 1.8 TiB | `/` |

- Partition **numbers were not renumbered**: Ubuntu is still `p5`; `p2`, `p3`, `p4`, `p6` no longer exist.
- Usage on 2026-11-17: **646 GB used / 1.1 TB free (38%)**. `/mnt/taiwan_com` holds ~541 GB of that.
- Swap: `/swap.img`, 8 GB swap file (unchanged).

---

## 4. What was done (timeline)

| Date | Action |
|---|---|
| 2026-11-14 | Disk diagnosis. Most Steam games uninstalled to shrink the backup. Proton save data archived to `/mnt/taiwan_com/Backup/compatdata-backup.tar` (~30 GB). |
| 2026-11-14/15 | Old backup `taiwan_com-before-ntfs` deleted from `ssd-japao`. Full copy of `taiwan_com` → `ssd-japao/backups/taiwan_com` with `rsync -aHxh` (excluded `$RECYCLE.BIN`, `.Trash-1000`, `System Volume Information`, `/nuvens_win`). A system freeze interrupted the copy once (2026-11-15 10:38, cause not found in logs); it was resumed and verified (535 GB, ~365k files). |
| 2026-11-15 | Extra safety copies: `/home/icm` → `ssd-japao/backups/home_icm` (without `.cache`) and `/etc` → `ssd-japao/backups/etc`. |
| 2026-11-15 | GParted Live 1.8.1-6 (amd64 zip) extracted onto the 1 GB FAT partition of `ssd-japao` (`sda1`, UUID `F6CA-24EF`) — no USB stick used. One-time boot into it via `efibootmgr --bootnext 0003` ("UEFI: Removable Device"). |
| 2026-11-16 | Final delta backup (captured new `t_resgate/madrid.img`, 8.2 GB). In GParted Live: **deleted p6, p4, p3, p2**; **moved and resized p5** to fill the disk. Ubuntu booted normally afterwards. |
| 2026-11-16 | `/etc/fstab` backed up to `/etc/fstab.bak-pre-unificacao`; the two NTFS lines commented out; `/mnt/taiwan_com` created as a normal directory; data restored with `rsync -aHh` from `ssd-japao` (582 GB, exit code 0, ~3h45 at ~41 MB/s); `chown -R icm:icm /mnt/taiwan_com`. |
| 2026-11-17 | Reboot check: loop images (Edge, Steam compatdata) auto-mounted. `kz-icloud-bisync.timer` re-enabled. `update-grub` run. **Windows Boot Manager UEFI entry removed** (`efibootmgr -b 0000 -B`). |
| 2026-11-17 | Fixed user units/scripts that required `/mnt/taiwan_com` to be a mount point (§7 #1). Moved `/boot/efi/EFI/Microsoft` to backup; GRUB menu now has no Windows entry (§7 #2). |

---

## 5. Current configuration

### 5.1 `/etc/fstab` (verbatim, 2026-11-17)

```fstab
/dev/disk/by-uuid/6a4845fa-114f-457a-b228-d226dba78c48 / ext4 defaults 0 1
/dev/disk/by-uuid/5E9B-4ABB /boot/efi vfat defaults 0 1
/swap.img	none	swap	sw	0	0
#UUID=2060932757C968BE /mnt/taiwan_com ntfs3 defaults,nofail,uid=1000,gid=1000,umask=022,windows_names,x-gvfs-show,x-gvfs-name=taiwan_com 0 0
#UUID=9CE69DCDE69DA852 /mnt/taiwan_win ntfs-3g noauto,nofail,uid=1000,gid=1000,windows_names,x-gvfs-show,x-gvfs-name=taiwan_win 0 0
/home/icm/.local/share/Steam-NTFS/compatdata /mnt/taiwan_com/t_jogos/t_steam/steamapps/compatdata none bind,nofail,x-systemd.requires-mounts-for=/mnt/taiwan_com 0 0

/mnt/taiwan_com/t_jogos/t_steam/t_compatdata_linux.ext4 /home/icm/.local/share/Steam-NTFS/compatdata ext4 loop,nofail,x-systemd.requires-mounts-for=/mnt/taiwan_com 0 0

# Microsoft Edge stored on taiwan_com
/mnt/taiwan_com/t_apps/microsoft-edge.ext4 /opt/microsoft/msedge ext4 loop,nofail,x-systemd.requires-mounts-for=/mnt/taiwan_com 0 0
```

Older fstab backups exist in `/etc` (`fstab.backup-2026-06-*`, `fstab.bak-edge`, `fstab.bak-pre-unificacao`).

### 5.2 `/mnt/taiwan_com` (now a folder, owner `icm:icm`, mode 777)

Top-level folders: `Apps`, `Backup`, `Installers`, `Offline`, `iCloudDriveCache`, `kz_reports`, `msdownld.tmp`, `t_apps`, `t_apps_win`, `t_bdmg`, `t_docs`, `t_downloads`, `t_jogos`, `t_linux`, `t_quartz`, `t_resgate`, `t_rpg`, `t_zotero`, `winboat`, `.rclone-cache`.

Other dirs under `/mnt`: `taiwan_win` (empty, obsolete), `gparted-usb` (empty, temporary), `edge-staging` (older Edge work).

### 5.3 Apps that depend on `/mnt/taiwan_com`

| App / component | How it is set up | Status 2026-11-17 |
|---|---|---|
| **Microsoft Edge** | 2 GB ext4 image `/mnt/taiwan_com/t_apps/microsoft-edge.ext4` loop-mounted at `/opt/microsoft/msedge` (fstab). Binary: `/usr/bin/microsoft-edge`. | Mounted ✅ |
| **Steam** | Flatpak `com.valvesoftware.Steam`. Libraries: default (`~/.var/app/com.valvesoftware.Steam/.local/share/Steam`) and `/mnt/taiwan_com/t_jogos/t_steam`. Flatpak override: `filesystems=/mnt/taiwan_com/t_jogos/t_steam`. | Library present ✅ (launch not re-tested by assistant) |
| **Steam Proton prefixes** | 64 GB ext4 image `t_compatdata_linux.ext4` (label `t_compatdata`) loop-mounted at `~/.local/share/Steam-NTFS/compatdata`, then **bind-mounted** to `.../t_steam/steamapps/compatdata`. Legacy workaround for NTFS not supporting Proton prefixes. | Mounted ✅ |
| **WinBoat** (Windows in Docker, used for Excel) | Docker container `WinBoat`; `~/.winboat/docker-compose.yml` maps `/mnt/taiwan_com/winboat:/storage`; disk `winboat/data.img` (64 GB). | Running ✅ |
| **iCloud offline sync** | rclone remote `icloud:`; scripts `~/.local/bin/kz-icloud-bisync`, `kz-icloud-sync`; user units `kz-icloud-bisync.service/.timer` (every 15 min). Obsidian vault "Horizontes" at `/mnt/taiwan_com/Offline/icloud/Obsidian/Horizontes`. | Fixed 2026-11-17 ✅ (was skipped 11-16 → 11-17, see §7 #1) |
| **kz-craft-update** | User timer (daily) updating DeckCraft/WordCraft/GridCraft AppImages on `/mnt/taiwan_com`. | Fixed 2026-11-17 ✅ (next run 2026-11-18 04:08) |
| Other scripts in `~/.local/bin` | `kz-apps`, `kz-drive`, `wordcraft-taiwan`, `gridcraft-taiwan`, `deckcraft-taiwan`, `rstudio-taiwan`, `t_czepeku_extract`; `~/.bashrc` functions use `/mnt/taiwan_com/t_rpg/...`. | Paths still valid (only plain paths, no mount check) |

### 5.4 Boot

| Item | Value |
|---|---|
| UEFI `BootOrder` | `0001` (Ubuntu, `\EFI\ubuntu\shimx64.efi`), `0002` CD/DVD, `0003` Removable Device, `0004` Network |
| Windows UEFI entry | Removed (was `Boot0000`) |
| `/boot/efi/EFI` | `Boot`, `ubuntu` (Windows `Microsoft` folder moved to backup on 2026-11-17) |
| GRUB | `GRUB_TIMEOUT_STYLE=hidden`, `GRUB_TIMEOUT=0`, `os-prober` installed and enabled (`#GRUB_DISABLE_OS_PROBER=false`) |

---

## 6. Backups (on external disk `ssd-japao`, `/dev/sda`)

| Path on `ssd-japao` | Content |
|---|---|
| `sda1` (1 GB FAT, UUID `F6CA-24EF`) | GParted Live 1.8.1-6 (bootable as "UEFI: Removable Device") |
| `backups/taiwan_com/` | Full pre-deletion copy of the old NTFS partition (~582 GB) |
| `backups/home_icm/` | Copy of `/home/icm` (without `.cache`), 2026-11-15 |
| `backups/etc/` | Copy of `/etc`, 2026-11-15 |
| `backups/nvme0n1-partition-table.txt` | `sfdisk -d` dump of the **old** 6-partition table, 2026-11-15 |
| `backups/EFI-Microsoft/` | Windows bootloader files removed from the EFI partition (26 MB), 2026-11-17 |

Keep these until everything is confirmed working. The disk was not connected on 2026-11-17.

---

## 7. Open issues and pending tasks

### #1 — iCloud sync and craft updates silently skipped — ✅ RESOLVED 2026-11-17 18:09

**Fix applied:** in the four units below `ConditionPathIsMountPoint=` → `ConditionPathIsDirectory=`; in both scripts `mountpoint -q "$BASE"` → `[[ -d "$BASE" ]]`; `systemctl --user daemon-reload`. Originals saved with suffix `.bak-unificacao` (next to each file). Manual run of `kz-icloud-bisync.service` exited `0/SUCCESS`; the 15-min timer is firing again.

Original problem (kept for reference): four user units required `/mnt/taiwan_com` to be a mount point, which is no longer true:

```
~/.config/systemd/user/kz-icloud-bisync.service   ConditionPathIsMountPoint=/mnt/taiwan_com
~/.config/systemd/user/kz-craft-update.service    ConditionPathIsMountPoint=/mnt/taiwan_com
~/.config/systemd/user/kz-icloud-sync.service     ConditionPathIsMountPoint=/mnt/taiwan_com  (disabled)
~/.config/systemd/user/rclone-icloud-offline.service ConditionPathIsMountPoint=/mnt/taiwan_com (disabled)
```

Journal: `kz-icloud-bisync.service ... was skipped because of an unmet condition check (ConditionPathIsMountPoint=/mnt/taiwan_com)` every 15 min since the unification. The scripts also abort on the same test:

- `~/.local/bin/kz-icloud-bisync` line 17: `mountpoint -q "$BASE" || { echo "ERROR: Shared partition not mounted." ...; exit 1; }`
- `~/.local/bin/kz-icloud-sync` line 8: `if ! mountpoint -q "$BASE"; then ... exit 1`

### #2 — Leftover Windows bootloader files — ✅ RESOLVED 2026-11-17 18:20

`/boot/efi/EFI/Microsoft` was moved to `ssd-japao/backups/EFI-Microsoft`; `sudo update-grub` no longer finds Windows. `os-prober` is still enabled (harmless now).

### #3 — Final app tests (LOW)

Edge, Steam and WinBoat launch tests were requested but not formally confirmed (WinBoat container is up; Edge and compatdata images mount).

### #4 — Optional cleanup (no functional impact)

- `x-systemd.requires-mounts-for=/mnt/taiwan_com` and `nofail` in fstab are now redundant (harmless).
- The Steam compatdata **ext4 image + bind mount** was only needed because of NTFS; it could be simplified to a plain folder later.
- Folder name `Steam-NTFS` is now misleading (kept for path compatibility).
- Windows-era folders: `/mnt/taiwan_win`, `/mnt/gparted-usb`, `t_apps_win`, `msdownld.tmp`.
- `/mnt/taiwan_com` is mode 777 (world-writable); 755 would be enough.
- With 1.1 TB free, the 8 GB swap file could be enlarged if needed.
- Commented NTFS lines in fstab can be deleted once confident.

---

## 8. Rules for any AI helping with taiwan

1. There is **one** data partition. Do not suggest "shared partitions" or NTFS options for this machine.
2. Keep using `/mnt/taiwan_com/...` paths for large data; never remove that folder.
3. Do not use `mountpoint`/`ConditionPathIsMountPoint` checks on `/mnt/taiwan_com`; use directory checks.
4. Windows dual boot no longer exists; Windows apps run only in **WinBoat** (Docker).
5. Steam is a **Flatpak**; library paths outside `$HOME` need a Flatpak filesystem override.
6. `sudo` commands must be run by Igor; give them as one consolidated block, explain each line, edit files with `gedit`.
7. Primary documentation source: repository `kaze12/tech-linux` on GitHub (this file lives there as `docs/configuracoes/unificacao-particoes-taiwan.md`). Docs written before 2026-11-16 (e.g. `aplicativos-apt-taiwan-com.md`, `rclone-icloud-bisync.md`) describe the old NTFS layout; where they conflict, this report wins. The repo copies `config/systemd/user/kz-icloud-bisync.service` and `scripts/kz-icloud-bisync` were updated with the §7 #1 fix on 2026-11-26.

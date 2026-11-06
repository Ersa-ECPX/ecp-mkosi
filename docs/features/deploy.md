# `foundation deploy`: install the system on a disk

**Status:** built. Tested with real `systemd-repart` and mtools on sparse image files, the refusal rules, the daemon and
command line end to end on a file target, and by running it from the installer image onto a 32-40 GiB virtual disk
which then booted alone through UEFI. Not tested: a real physical disk or USB stick, non-512-byte sectors, Secure Boot.

## What it is

`foundation deploy install DISK` copies the **running** system onto a whole disk it first erases: partitions, root
image, kernel image, and the data volume, laid out for A/B updates. It is what the [graphical installer](installer-app.md)
calls. It is called `deploy`, not `install`, because installing packages is `foundation compat` and `foundation apps`.

## Using it

```
foundation deploy list                           # whole disks, and whether each may be installed to (and why not)
foundation deploy install /dev/nvme1n1 --dry-run # the plan, nothing changed
foundation deploy install /dev/nvme1n1 \
    --language de_DE.UTF-8 --timezone Europe/Berlin
```

The command prints the plan (partitions and sizes, the partitions that exist now and will be erased) and makes you type
the disk path back. `--yes` skips the prompt; the daemon still requires the path as confirmation. `--language` and
`--timezone` are written into the installed system's `/etc`; the time zone must exist in the tz database.

## What gets created

| Partition | Size | Notes |
| --- | --- | --- |
| ESP | 1 GiB | systemd-boot, the fallback boot path, one kernel image |
| Recovery | 1 GiB | reserved, empty |
| Root slot A | 8 GiB | the running system, label `ecp-root_<version>` |
| Root slot B | 8 GiB | empty (`_empty`), for the next update |
| `ECP-DATA` | the rest | btrfs with `var` and `home` subvolumes |

The smallest usable disk is about 28 GiB. Sizes and labels live in one place, `backends/layout.py`, and a test compares
them with `mkosi.repart.system/`.

## What it refuses

A target must be an absolute path to a whole block device. Refused: partitions, plain files (except in tests, through
a flag the system bus rejects), paths with `..`, the disk holding the running root, any disk with a mounted partition, a
read-only disk, a disk that is too small, a running slot with no kernel image, a version too long for a partition name.
Polkit's `deploy` action always asks for an administrator and never remembers the answer. One deployment runs at a time.

## The steps

1. Validate, then stage under the daemon's state directory: copy the running ESP, put the kernel image there as
   `ecp_<v>+3.efi.partial` (systemd-boot ignores it) **with its command line rewritten**, and build the data volume seed.
   The rewrite names the new disk's own root and data partitions by PARTUUID (see
   [storage](storage-and-persistence.md)); the seed holds the `/etc` overlay directories labelled `etc_t`, an empty
   `/home`, and `locale.conf` and `localtime` if chosen.
2. **Commit** (point of no return). `systemd-repart --empty=force --offline=yes` creates the partitions with those UUIDs
   pinned. A fresh partition table makes the old contents unreachable; on a plain file nothing of the old data survives.
   On a real disk repart sends TRIM, which is a hint, not a secure erase.
3. Copy the root image into slot A, read both back from the disk (bypassing the page cache) and compare SHA-256.
4. Check the created UUIDs match the command line, then rename the kernel image to `ecp_<v>+3.efi` with `mren` (no
   mount). Only now can the disk boot, so an interrupted or failed install never looks bootable.
5. `fsync`.

No firmware boot entry is written; firmware must find the disk through the fallback path `EFI/BOOT/BOOTX64.EFI`.

## Verified by booting

On an installed disk booted alone through UEFI: `/etc/locale.conf` and `/etc/localtime` as chosen, `liveuser` absent,
a user created through AccountsService gets its home, the entry is blessed after the first boot, no failed units.

## Limits

- No `--image FILE` source, so writing an installer USB from an installed system is not possible yet.
- No unattended user creation (`--user`, `--password-file`); the account is created by GNOME Initial Setup on first boot.
- No encryption or choice of data filesystem.
- The installed kernel image cannot be re-signed; a Secure Boot system would reject the edited command line.

See also: [installer](installer-app.md), [installer medium](live-medium.md), [docs/installer.md](../installer.md).

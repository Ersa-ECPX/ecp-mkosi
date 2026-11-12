# `foundation deploy`: install the system on a disk

**Status:** rewritten for dm-verity. Tested with real `systemd-repart` and mtools on sparse image files (the partition
layout, the payload streamed into the slot, the signed kernel image installed byte for byte, the refusal rules, the daemon and
command line end to end on a file target). **Not yet run from a real installer medium, and no system installed by it has been
booted** (the older version, which copied the live root, was tested that way). Not tested: a real physical disk or USB stick,
non-512-byte sectors, Secure Boot.

## What it is

`foundation deploy install DISK` installs the operating system onto a whole disk it first erases: partitions, root image
and its dm-verity hash tree, kernel image, and the data volume, laid out for A/B updates. The system to install is the
**payload** of the installer medium: the update archive of the normal image (`/usr/share/ersa/payload/*.tar.zst`, made by
`scripts/stage-payload`), the same file `foundation update install` takes. It is what the [graphical installer](installer-app.md)
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
| Root slot A | 8 GiB | the payload's root filesystem, label `ecp-root-a`, UUID = first half of the root hash |
| Hash tree A | 96 MiB | its dm-verity hash tree, label `ecp-verity-a`, UUID = second half of the root hash |
| Root slot B | 8 GiB | empty, label `ecp-root-b`, for the next update |
| Hash tree B | 96 MiB | empty, label `ecp-verity-b` |
| `ECP-DATA` | the rest | btrfs with `var` and `home` subvolumes |

The smallest usable disk is about 28 GiB. Sizes and labels live in one place, `backends/layout.py`, and a test compares
them with `mkosi.repart.system/`.

## What it refuses

A target must be an absolute path to a whole block device. Refused: partitions, plain files (except in tests, through
a flag the system bus rejects), paths with `..`, the disk holding the running root, any disk with a mounted partition, a
read-only disk, a disk that is too small, a running slot with no kernel image, a version too long for a partition name.
Polkit's `deploy` action always asks for an administrator and never remembers the answer. One deployment runs at a time.

## The steps

1. Validate, then stage under the daemon's state directory: read the payload's manifest and its kernel image (the first
   files of the archive), copy the medium's ESP (boot loader, configuration, Secure Boot keys) and put the kernel image there
   as `ecp_<v>+3.efi.partial` (systemd-boot ignores it) **exactly as built**: it is signed, and its command line is the same on
   every machine. Build the data volume seed: the `/etc` overlay directories labelled `etc_t`, an empty `/home`, and
   `locale.conf` and `localtime` if chosen.
2. **Commit** (point of no return). `systemd-repart --empty=force --offline=yes` creates the partitions; slot A's two
   get the UUIDs the root hash names. A fresh partition table makes the old contents unreachable; on a plain file nothing of
   the old data survives. On a real disk repart sends TRIM, which is a hint, not a secure erase.
3. Stream the root filesystem and the hash tree from the archive straight into slot A's partitions (their checksums are
   compared with the manifest as they go), then read both back from the disk (bypassing the page cache) and compare SHA-256.
4. Rename the kernel image to `ecp_<v>+3.efi` with `mren` (no mount). Only now can the disk boot, so an interrupted or
   failed install never looks bootable.
5. `fsync`.

The payload is trusted because it is part of the medium's own root, which dm-verity protects; the checksums against the
manifest are still checked.

No firmware boot entry is written; firmware must find the disk through the fallback path `EFI/BOOT/BOOTX64.EFI`.

## Verified by booting

On an installed disk booted alone through UEFI: `/etc/locale.conf` and `/etc/localtime` as chosen, `liveuser` absent,
a user created through AccountsService gets its home, the entry is blessed after the first boot, no failed units.

## Limits

- No `--image FILE` source, so writing an installer USB from an installed system is not possible yet.
- No unattended user creation (`--user`, `--password-file`); the account is created by GNOME Initial Setup on first boot.
- No encryption or choice of data filesystem.
- The installed data volume is still found by the generic `LABEL=ECP-DATA` tokens in the kernel command line, so two
  disks with that label attached at once are ambiguous. A generator that finds the volume on the booted disk is planned.

See also: [installer](installer-app.md), [installer medium](live-medium.md), [docs/installer.md](../installer.md).

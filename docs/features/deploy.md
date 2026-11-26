# `foundation deploy`: install the system on a disk

## What it is

`foundation deploy install DISK` erases a whole disk and installs the operating system on it: partitions, root image with
its dm-verity hash tree, kernel image and data volume, laid out for A/B updates. The system comes from the **payload** of
the installer medium, the update archive of the normal image (`/usr/share/ersa/payload/*.tar.zst`, made by
`scripts/stage-payload`), the same file `foundation update install` takes. The [graphical installer](installer-app.md) calls
it. It is `deploy`, not `install`, because installing packages is `foundation apps` or `compat`.

## Using it

```
foundation deploy list                            # whole disks, and whether each may be installed to (and why not)
foundation deploy install /dev/nvme1n1 --dry-run  # the plan, nothing changed
foundation deploy install /dev/nvme1n1 --language de_DE.UTF-8 --timezone Europe/Berlin
```

The command prints the plan and makes you type the disk path back (`--yes` skips this; the daemon still requires the path
as confirmation). Language and time zone are written into the installed `/etc`; the zone must exist in the tz database.

## What is created

| Partition | Size | Notes |
| --- | --- | --- |
| ESP | 1 GiB | systemd-boot, fallback boot path, one kernel image |
| Recovery | reserved, empty | |
| Root slot A + hash tree | 8 GiB + 96 MiB | the payload; UUIDs are the two halves of the root hash |
| Root slot B + hash tree | 8 GiB + 96 MiB | empty, for the next update |
| `ECP-DATA` | the rest | btrfs, subvolumes `var` and `home` |

The smallest usable disk is about 28 GiB. Sizes and labels live in `backends/layout.py`. The recovery size there (2 GiB)
differs from `mkosi.repart.system/05-recovery.conf` (1G): a known mismatch that makes the comparison test fail.

## How it works

Refused: partitions, plain files (except in tests, through a flag the system bus rejects), paths with `..`, the disk of the
running root, a disk with a mounted partition, a read-only or too small disk, a running slot with no kernel image. Polkit's
`deploy` action always asks for an administrator. One deployment runs at a time.

1. Validate, then stage: read the payload's manifest and kernel image, copy the medium's ESP and add the signed kernel image
   exactly as built as `ecp_<v>+3.efi.partial` (systemd-boot ignores it). Build the data-volume seed (the `/etc` overlay
   directories labelled `etc_t`, an empty `/home`, `locale.conf` and `localtime`).
2. **Commit** (point of no return): `systemd-repart --empty=force --offline=yes`. On a real disk repart sends TRIM, which
   is a hint, not a secure erase.
3. Stream root and hash tree into slot A (checksums compared with the manifest), then read both back and compare SHA-256.
4. Rename the kernel image to `ecp_<v>+3.efi` (`mren`, no mount). Only now can the disk boot, so an interrupted install never
   looks bootable. Then `fsync`.
5. Register the disk with the firmware: `efibootmgr` creates a boot entry "Ersa Compute Platform" for the new ESP
   (`\EFI\BOOT\BOOTX64.EFI`) and puts it first in the boot order, replacing entries of earlier installs. Without it the
   firmware finds the disk only through its fallback path, and with the installer medium still plugged in it often starts the
   medium again, whichever disk was chosen. Best effort: a failure is reported as a note and the disk stays bootable. Not
   done for a file target or a computer not started through UEFI. Unit-tested with a fake `efibootmgr`; not run on a real firmware or in QEMU yet.

## Unattended installs: preseed

```
foundation deploy check-preseed office.conf                          # validate it; changes nothing
sudo foundation deploy install /dev/sdb --preseed office.conf --yes  # install without questions
```

A preseed file is `key=value` lines (`#` for comments): any setting of the configuration schema (`foundation config list`:
`ota.url`, `ota.auto_update`, `apps.default_apps`, ...) plus `deploy.language`, `deploy.timezone` and `deploy.hostname`.
Settings are written to `/etc/foundation/foundation.conf.d/50-preseed.conf` (the API's own file sorts after it, so they can be
changed later); the others become `/etc/locale.conf`, `/etc/localtime` and `/etc/hostname`. The whole file is checked before
anything is erased and all errors are reported together. `--language` and `--timezone` win over the file. Daemon methods:
`Deploy.DeployPreseeded` and `CheckPreseed`. Put nothing secret in it: it is copied as is.

`installer.timezone_url` is a setting of the installer program only (checked, never written to the installed system). The installer asks
that `https://` service for the time zone of the network address and preselects it (default `https://ipapi.co/timezone/`; empty turns it
off; the person can always change it). It reads `/etc/foundation/installer.preseed`, else `/usr/share/ersa/installer.preseed` on the medium.
Unit-tested with a fake service; not run against the real service from an installer medium.

Tested: parsing, the files written into the seed and the daemon's refusal of a bad file (14 tests). Not built: finding the
file by itself (a labelled USB partition or an `ecp.preseed=` kernel argument), creating the first user, network and Wi-Fi
settings, enrolling in management.

## Limits

- No `--image FILE` source, so an installer USB cannot be written from an installed system yet.
- No unattended user creation; GNOME Initial Setup creates the account on first boot.
- No encryption or choice of data filesystem.
- The firmware boot entry is made with `efibootmgr` (best effort); on a computer not started through UEFI, or when it fails, the disk
  is still bootable through the firmware's fallback path but may have to be chosen in the boot menu.

See also: [graphical installer](installer-app.md), [installer medium](live-medium.md), [design notes](../installer.md).

## Tested

Tested with real `systemd-repart` and mtools on sparse image files (layout, payload streamed into the slot, kernel image installed byte for byte, refusal rules, daemon and CLI end to end on a file target) and, in QEMU, from the installer medium onto a blank disk, which then booted (`scripts/qemu-fallback-test`). Not tested: a real disk or USB stick, non-512-byte sectors, Secure Boot, and a preseed install end to end.

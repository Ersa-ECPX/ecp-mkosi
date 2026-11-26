# Storage and persistence

## What it is

The OS is separated from what must survive updates (settings, data). Two root slots allow A/B updates
([boot and slots](boot-and-slots.md)).

| Partition | Size | Filesystem | Holds |
| --- | --- | --- | --- |
| ESP | 1 GiB | vfat | systemd-boot, one kernel image per slot |
| Recovery | reserved, empty | none | |
| Root slot A / B | 8 GiB each, with a 96 MiB hash partition | EROFS, read-only, dm-verity | the operating system, the next or previous version |
| `ECP-DATA` | the rest (at least 10 GiB) | btrfs | subvolumes `var` (at `/var`) and `home` (at `/home`) |

The recovery size is inconsistent: `foundation/foundationd/backends/layout.py` says 2 GiB and
`mkosi.repart.system/05-recovery.conf` says 1G (a unit test fails on it, unresolved).

- `/usr` and the rest of the root are read-only; an update replaces the image whole.
- `/etc` is an overlay: the image's `/etc` is the lower layer, local changes (users, network, machine ID, SSH host keys) are
  the upper layer at `/var/lib/ecp/etc/upper`. New defaults from an update show through; local changes stay on top.
- `/var` and `/home` share one btrfs volume: shared free space, separate lifecycles. `/tmp` is a tmpfs.

## How it is mounted

`/var` and `/home` are mounted in the initrd (there is no `/etc/fstab`) from the data volume of the disk the system started
from, not from a label on the kernel command line. `ecp-data-device.service` runs `/usr/libexec/ecp/data-link`, which asks
systemd-boot which partition it started from (the EFI variable `LoaderDevicePartUUID`, the ESP, whose PARTUUID is random per
disk), takes that partition's disk, and links its partition named `ECP-DATA` (or `ECP-LIVE` on the installer medium) at
`/run/ecp/data`. `sysroot-var.mount` and `sysroot-home.mount` mount `subvol=var` and `subvol=home` from that link before
switch-root, as `initrd-fs.target` requirements. The root is found by its dm-verity root hash (`roothash=`; the partition UUIDs
are its two halves), so the signed kernel image is identical on every machine ([dm-verity](dm-verity.md)).

Because the choice is by disk, two disks that both carry `ECP-DATA` (an installed disk and a leftover install) no longer
compete, and an installed disk attached next to the installer medium cannot supply its `/var` ([installer medium](live-medium.md)).
If the boot loader did not record a partition, or the disk has no data volume, the helper fails with a reason on the console
instead of guessing, and the boot stops in the emergency target.

`sysroot-etc.mount` (in `mkosi.initrd.extra`) builds the overlay in the initrd, once `/sysroot/var` is mounted. Without a
writable `/etc` this early, PID 1 starts with the image's placeholder machine ID and `dbus-broker` refuses to start. The
overlay's directories are made by the image build or by `foundation deploy`'s seed.

## Limits

No quotas, compression policy or encryption yet. ZFS is not planned (out-of-tree kernel module).

See also: [architecture.md](../architecture.md#storage-layout), [SELinux](selinux.md), [deploy](deploy.md).

## Tested

The layout, the `/etc` overlay and the data volume were tested in QEMU, including a user created through AccountsService that kept its home across a reboot. The disk-based volume lookup (`data-link`) is unit-tested against fake sysfs trees, including two disks with the same label, but **no image with it has been built or booted yet**; the earlier label-based mount is what the QEMU tests above used. Not tested: two ECP disks attached at once in a VM, or real hardware.

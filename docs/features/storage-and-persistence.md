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

`/var` and `/home` come from `systemd.mount-extra=` on the kernel command line (there is no `/etc/fstab`). The data volume
is found by `LABEL=ECP-DATA`, set in `mkosi.conf.d/30-data-mounts.conf`. The root is found by its dm-verity root hash
(`roothash=`; the partition UUIDs are its two halves), never by label, so the signed kernel image is identical on every
machine ([dm-verity](dm-verity.md)). The installer medium uses `ECP-LIVE` and `ecp-live` so an installed disk cannot be
taken for it ([installer medium](live-medium.md)).

`/var` carries `x-initrd.mount`: a `mount-extra=` entry is only mounted before switch-root with it. Without it the `/etc`
overlay's directories do not exist and the machine drops to emergency mode.

`sysroot-etc.mount` (in `mkosi.initrd.extra`) builds the overlay in the initrd, once `/sysroot/var` is mounted. Without a
writable `/etc` this early, PID 1 starts with the image's placeholder machine ID and `dbus-broker` refuses to start. The
overlay's directories are made by the image build or by `foundation deploy`'s seed.

## <a id="known-gap"></a>Known gap

Two disks that both carry the label `ECP-DATA` (for example an installed disk and a leftover installed system) are
ambiguous: the wrong one can supply `/var` and `/home`. This once left a user with an account but no home. Planned fixes: a
generator that finds the volume on the booted disk, and a per-install command-line addon on the ESP (which `systemd-stub`
appends to every kernel image) for kernel images that arrive by OTA. Neither is written.

## Limits

No quotas, compression policy or encryption yet. ZFS is not planned (out-of-tree kernel module).

See also: [architecture.md](../architecture.md#storage-layout), [SELinux](selinux.md), [deploy](deploy.md).

## Tested

The layout, the `/etc` overlay and the data volume were tested in QEMU, including a user created through AccountsService that kept its home across a reboot. Not tested: two ECP disks attached at once, or real hardware.

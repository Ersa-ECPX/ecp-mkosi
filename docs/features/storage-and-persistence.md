# Storage and persistence

**Status:** built. The layout, the `/etc` overlay and the data volume were tested in QEMU, including a user created
through AccountsService that kept its home across a reboot. The mount-by-PARTUUID design was verified in unit tests
and on an installed disk; its behaviour with two ECP disks attached is not yet re-tested on a rebuilt image.

## What it is

The system separates what must never change (the OS) from what must survive updates (settings, data).

| Partition | Size | Filesystem | Holds |
| --- | --- | --- | --- |
| ESP | 1 GiB | vfat | systemd-boot and one kernel image per slot |
| Recovery | 1 GiB | none | reserved, empty |
| Root slot A | 8 GiB | EROFS, read-only | the operating system |
| Root slot B | 8 GiB | EROFS, read-only | the next or previous version |
| `ECP-DATA` | the rest (at least 10 GiB) | btrfs | subvolume `var` (at `/var`) and subvolume `home` (at `/home`) |

- **`/usr` and the rest of the root are read-only.** Nothing writes to the image, so an update replaces it whole.
- **`/etc` is an overlay.** The image's `/etc` is the lower layer; local changes (users, network settings, the
  machine ID, SSH host keys) live in the upper layer at `/var/lib/ecp/etc/upper`, so new defaults from an update
  appear and local changes stay on top.
- **`/var` and `/home` share one btrfs volume** as subvolumes: shared free space, separate lifecycles.
- **`/tmp` is a tmpfs.**

## How the right disk is chosen

`/var` and `/home` are mounted from **`systemd.mount-extra=` on the kernel command line** (honoured in the initrd and
in the running system), not from `/etc/fstab`, which does not exist. Mounting `LABEL=ECP-DATA` let a second disk with
the same labels, such as a leftover installer USB, supply `/var` and `/home` by mistake. That is how a user once
ended up with an account but no home directory.

- A **built image** (QEMU development, the root image updates are made from) names its data volume by label, which is
  fine with one disk (`mkosi.conf.d/30-data-mounts.conf`).
- An **installed system** names its root and data volume by **PARTUUID**. `foundation deploy` creates the partitions
  with UUIDs it picked, then rewrites the copied kernel image's command line to `root=PARTUUID=...` and
  `systemd.mount-extra=PARTUUID=...:/var:btrfs:subvol=var,x-initrd.mount` (and `/home`). The padding argument `ecp.pad=` in every
  built kernel image reserves room for the longer text; a test fails if the room ever stops being enough.
- The **installer medium** also names its small data volume by label. Its root is labelled `ecp-live_<version>` (and
  the kernel command line says so), not `ecp-root_<version>`, so an installed disk attached at the same time cannot be
  taken for it ([installer medium](live-medium.md)).

**`/var` carries `x-initrd.mount`.** A `mount-extra=` entry is only mounted before switch-root with that option (as the old
`fstab` line did). Without it `/sysroot/var` is never mounted in the initrd, the `/etc` overlay's directories do not
exist, the overlay fails to mount, and the machine drops to emergency mode with a locked root account. That is exactly
what happened to the first image built with this scheme; it was found by booting headless and reading the console, and
the fix was verified on the same image by adding the option to its command line.

## The `/etc` overlay in the initrd

`sysroot-etc.mount` (in `mkosi.initrd.extra`) builds the overlay before switch-root, once `/sysroot/var` is mounted from the
kernel command line. Without a writable `/etc` this early, PID 1 starts with the image's placeholder machine ID and
`dbus-broker` refuses to start. The overlay's directories exist on the data volume from the start (made by the image
build, or by `foundation deploy`'s seed).

## <a id="known-gap"></a>Known gap

Kernel images that arrive later by OTA update are built with the label form. After an update the confusion between
disks can return if another ECP disk is attached at boot. The planned fix is a per-install command-line addon on the
ESP, which `systemd-stub` appends to every kernel image.

## Limits

- Two *installed* ECP disks of the same version both carry the label `ecp-root_<version>`; only the data mounts and the
  root of a deployed system are protected by PARTUUID, which covers the common cases (installer USB, second install).
- No quotas, compression policy or encryption yet. ZFS is not planned (out-of-tree kernel module).

See also: [architecture.md](../architecture.md#storage-layout), [SELinux](selinux.md), [deploy](deploy.md).

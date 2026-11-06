# The installer medium (live image)

**Status:** built. This layout (a small writable data volume holding the live session and the sysext) booted in QEMU and
installed a disk. A **read-only variant** (RAM disks, a read-only extension partition) was tried and **removed**: on its
first boot it fell into emergency mode (see below).

## What it is

A bootable image that runs the installer: the **same operating system** as an installed one, in a small layout, started
with automatic sign-in as a live user and the [graphical installer](installer-app.md) opened. It is an mkosi profile, not
a separate distribution.

## Building it

```
scripts/build-foundation-package            # the foundation RPMs, including foundation-installer
scripts/build x86-64 --profile installer    # builds the sysext first, then the image
```

`scripts/build` runs `scripts/build-installer-sysext` before an installer build. If the staging folder
`mkosi.installer.extra` is owned by root from an earlier `sudo` build, remove it once: `sudo rm -rf mkosi.installer.extra`.

## Layout

| Partition | Size | Holds |
| --- | --- | --- |
| ESP | 512 MiB | systemd-boot and the kernel image |
| Root | 5 GiB | the operating system, EROFS, labelled `ecp-live_<version>` |
| `ECP-DATA` | 512 MiB | btrfs: the live session's `/var` and `/home`, and the installer sysext |

About 6 GiB, so it fits an 8 GB stick. The root is labelled `ecp-live_<version>`, not `ecp-root_<version>`, and the kernel
command line says so (the last `root=` wins), so an installed disk attached at the same time is never taken for it.
`/var` and `/home` are mounted by label from the kernel command line like any built image
([storage](storage-and-persistence.md)).

**The medium is written to** (the live session's `/var`, logs, the overlay for `/etc`), so a live session leaves traces on a
stick until it is wiped; installed systems get a fresh data volume and carry none of it.

### The read-only variant that did not work

The idea: never write to the medium. `/var`, `/home` and the `/etc` overlay's storage in RAM, the sysext on a read-only
partition, a kernel image with no boot counter. It built, and its pieces tested, but the first real boot stopped in
emergency mode (`You are in emergency mode ... the root account is locked`): the RAM disk for the `/etc` overlay,
`sysroot-var-lib-ecp.mount`, **failed to mount in the initrd**, so the overlay and then the initrd file systems failed.
That mount carried an SELinux `context=` (the overlay's directories need `etc_t`, and a directory created in the initrd
has no label because the policy is not loaded yet). The exact error was not captured; the likeliest cause is that a
`context=` mount cannot be made before the policy loads, which is unverified. It was removed at the maintainer's request.
If it is tried again, boot it headless with a debug shell first and read the failing unit.

## The live session

The installer is a **system extension** built from the `foundation-installer` RPM (`scripts/build-installer-sysext`),
labelled for SELinux and stored on the data volume at `/var/lib/extensions`. At boot `systemd-sysext.service`
merges it; a one-line drop-in then starts `ersa-live-session.service` (a unit merged mid-boot is not in the boot's job
list, so it has to be started). That service creates:

- the user `liveuser` (password `liveuser`) in `wheel`, with passwordless sudo,
- a polkit rule letting `liveuser` run deploy, jobs and reboot without a password,
- GDM automatic sign-in (both `custom.conf` and `DISPLAYMANAGER_AUTOLOGIN`, which SUSE's gdm reads first),
- the installer's autostart, and no first-boot "welcome" for that user.

The root image itself contains **none** of this, and `foundation deploy` writes a fresh data volume, so an installed
system never has `liveuser`, the polkit rule or the autostart. The only trace in every image is the one-line drop-in for
`systemd-sysext.service`, which does nothing without extensions.

## Trying it in QEMU

```
cp /usr/share/edk2/x64/OVMF_VARS.4m.fd /var/tmp/ecp-vars.fd
qemu-system-x86_64 -enable-kvm -cpu host -m 4096 -smp 2 -machine q35 \
  -drive if=pflash,format=raw,readonly=on,file=/usr/share/edk2/x64/OVMF_CODE.4m.fd \
  -drive if=pflash,format=raw,file=/var/tmp/ecp-vars.fd \
  -drive file=mkosi.output/ecp-installer_0.1.0.raw,format=raw,if=virtio,snapshot=on \
  -drive file=/var/tmp/ecp-test-disk.img,format=raw,if=virtio \
  -device virtio-vga-gl -display gtk,gl=on -serial mon:stdio
```

Boot the installed disk afterwards **on its own** (without the installer drive).

## If something is wrong

| What you see | Cause |
| --- | --- |
| GNOME Initial Setup instead of the installer | the sysext was not merged: look at `systemd-sysext status` |
| Emergency mode with a locked root account | a required mount failed in the initrd; boot headless and read the console (it was the removed read-only layout, then `/var` missing `x-initrd.mount`) |
| Black screen, no login | SELinux denials on an unlabelled extension (fixed: extensions are labelled for their merged paths) |
| An installed user with no home | two disks with the same labels mounted `/home` from the wrong one (fixed: installed systems name their disks by PARTUUID) |

A shell on a headless boot without a login: boot the kernel directly with `systemd.debug_shell=/dev/ttyS1` and a second
serial port ([building and testing](building-and-testing.md)).

## Limits

- Not tested on a real USB stick or real hardware; no Secure Boot.
- The live session writes to the stick's data volume, so its files and logs stay on the stick until it is wiped; there is no
  read-only mode.

See also: [storage](storage-and-persistence.md), [system extensions](system-extensions.md), [SELinux](selinux.md).

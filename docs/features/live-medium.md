# The installer medium (live image)

## What it is

A bootable image that runs the [graphical installer](installer-app.md): the same operating system as an installed one,
signed in automatically as a live user. It is an mkosi profile, not a separate distribution.

## Building and trying it

```
scripts/build-foundation-package            # the foundation RPMs, including foundation-installer
scripts/build x86-64 --profile installer    # builds the sysext first, then the image
```

If `mkosi.installer.extra` is owned by root from an earlier `sudo` build, remove it once. Output is
`mkosi.output/ecp-installer_<version>.raw`. To try it in QEMU:

```
scripts/qemu-try install     # the medium plus a blank target disk
scripts/qemu-try boot        # then the installed disk alone
```

## Layout

| Partition | Size | Holds |
| --- | --- | --- |
| ESP | 512 MiB | systemd-boot and the kernel image |
| Root `ecp-live` (+ `ecp-live-verity`) | as large as its contents (about 5.8 GiB measured) + 96 MiB hash | the operating system under dm-verity, plus the payload to install |
| `ECP-LIVE` | 512 MiB | btrfs: live `/var`, `/home` and the installer sysext |

It fits an 8 GB stick. The labels differ from `ECP-DATA` and the slot labels, so an installed disk attached at the same
time is never taken for the medium. The medium is written to (live `/var`, logs, the `/etc` overlay), so a live session
leaves traces on the stick until it is wiped. Installed systems get a fresh data volume.

## The live session

The installer is a system extension built from the `foundation-installer` RPM and stored at
`/var/lib/extensions/ersa-installer.raw`. `systemd-sysext.service` merges it at boot; a one-line drop-in in the root image
then starts `ersa-live-session.service`, which creates:

- `liveuser` (password `liveuser`, in `wheel`, passwordless sudo),
- a polkit rule letting `liveuser` run deploy, jobs and reboot without a password,
- GDM automatic sign-in (`custom.conf` and `DISPLAYMANAGER_AUTOLOGIN`, which SUSE's gdm reads first),
- the installer autostart.

The root image contains none of this, so an installed system has no `liveuser`, rule or autostart. The sysext's SELinux
labels are written into the image by `scripts/build-installer-sysext`.

## If something is wrong

| What you see | Cause |
| --- | --- |
| Initial Setup instead of the installer | the sysext was not merged: check `systemd-sysext status` |
| Emergency mode, root account locked | a required mount failed in the initrd; boot headless and read the console |
| Black screen, no login | SELinux denials, for example an unlabelled extension |

For a shell on a headless boot, use `systemd.debug_shell=/dev/ttyS1` with a second serial port
([building and testing](building-and-testing.md)).

## Limits

- A read-only variant (RAM disks, a read-only extension partition) was tried and removed: its `/etc` overlay mount failed in
  the initrd and the machine fell into emergency mode. The cause was not captured.
- No real hardware and no Secure Boot testing.

See also: [storage](storage-and-persistence.md), [system extensions](system-extensions.md), [SELinux](selinux.md).

## Tested

The layout booted in QEMU and installed a blank disk (`scripts/qemu-fallback-test`). Not tested on a real USB stick or real hardware, and not with Secure Boot.

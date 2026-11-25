# Installer design notes

Usage lives on the feature pages: [deploy](features/deploy.md), [graphical installer](features/installer-app.md),
[installer medium](features/live-medium.md). This page keeps the design in one place.

## Goals

- Put a complete Ersa Compute Platform system on a disk: the A/B layout, a bootable ESP and a fresh data volume.
- One backend for the graphical installer and the command line. Policy stays in `foundationd`
  (`org.ersa.Foundation1`, polkit-gated, runs jobs as root). `foundation` and `ersa-installer` are only D-Bus clients;
  neither implements deployment logic. The contract is `foundation/data/dbus/org.ersa.Foundation1.xml`.
- Installed systems carry nothing of the live session.

## The medium

The medium is an mkosi profile of the same OS (`scripts/build x86-64 --profile installer`), not a separate distribution.
Its layout (`mkosi.repart.installer/`): ESP 512 MiB, one dm-verity EROFS root `ecp-live` (+ `ecp-live-verity`, 96 MiB)
that also holds the payload to install, and a 512 MiB btrfs volume `ECP-LIVE` for the live `/var`, `/home` and the
installer sysext. The label differs from `ECP-DATA` so an installed disk is never taken for the medium.

The installer app, `liveuser`, the polkit rule, automatic sign-in and the autostart come from a **system extension**
(`ersa-installer.raw`, built by `scripts/build-installer-sysext`) on the data volume at `/var/lib/extensions/`. A sysext
is used because the root image is shared with installed systems. The ESP does not work: `systemd-sysext.service` only
looks in `/etc`, `/run` and `/var/lib/extensions`. The sysext's SELinux labels are written into its image from the policy's
`file_contexts`; an unlabelled one turned `/usr` into `unlabeled_t` and broke the desktop.
A read-only variant (RAM disks) was tried and removed: it dropped to emergency mode in the initrd.

## Deployment (`backends/deploy.py`)

1. Validate, then stage: read the payload (the update archive of the normal image in `/usr/share/ersa/payload/`),
   copy the medium's ESP with the payload's kernel image as `ecp_<v>+3.efi.partial`, build the data-volume seed (`/etc`
   overlay directories labelled `etc_t`, empty `/home`, `locale.conf` and `localtime` if chosen).
2. Commit (point of no return): `systemd-repart --empty=force --offline=yes` with definitions generated from
   `backends/layout.py`.
3. Stream the root image and hash tree into slot A, read both back and compare SHA-256.
4. Rename the kernel image to `ecp_<v>+3.efi` (`mren`, no mount). Only now can the disk boot, so an interrupted
   install never looks bootable.
5. `fsync`.

Layout: ESP 1 GiB, recovery (reserved, empty), root slots A and B 8 GiB each with 96 MiB hash partitions, `ECP-DATA` btrfs
with `var` and `home` taking the rest. Known mismatch: `layout.py` sets recovery to 2 GiB and
`mkosi.repart.system/05-recovery.conf` says 1G, so the test that compares them fails. No firmware boot entry is written;
firmware must use the fallback path `EFI/BOOT/BOOTX64.EFI`.

## Safety rules

- The target must be an absolute path to a whole block device. Partitions, plain files and `..` are refused (a daemon
  started with `--allow-file-targets` on a session bus accepts files, for tests).
- Refused: the disk of the running root, a disk with a mounted partition, a read-only disk, a disk under about 28 GiB,
  a running slot with no kernel image, a version too long for a partition name.
- The caller passes the disk path again as `confirm`. Polkit action `org.ersa.foundation.deploy` always asks for an
  administrator. One deployment at a time (job lock `disk`).

## D-Bus API (`org.ersa.Foundation1.Deploy`)

- `ListDisks() -> aa{sv}`: path, name, size, model, removable, boot_medium, mounted, usable, reason.
- `PlanDeploy(s disk) -> a{sv}`: disk, size, version, partitions, will_erase, problems, possible.
- `Deploy(s disk, s confirm, s language, s timezone) -> o`: a job. Language and time zone are written into the installed
  `/etc`; an unknown zone is refused before anything is written.
- `DeployPreseeded` and `CheckPreseed` for unattended installs ([deploy](features/deploy.md#unattended-installs-preseed)).

## Tested

- Unit tests: layout against the repart files, the staging, byte-identical root and kernel image, a corrupt copy leaving only
  the `.partial` file, refusal rules, daemon and CLI end to end on a file target, the installer window on a virtual display.
- QEMU: `scripts/qemu-fallback-test` installs from the installer medium onto a blank disk, boots it, and exercises a broken
  update with fallback. An earlier hand run booted an installed disk alone and checked locale, time zone, no `liveuser`
  and a created user's home.

## Not built

- `--image FILE` (writing an installer USB from an installed system), unattended user creation, a live-medium preseed
  finder, firmware boot entries, encryption, a data filesystem choice. Ventoy is a separate spike.

## Tested

The deploy backend and the installer window are covered by unit tests (real `systemd-repart` and mtools on sparse files, a real `foundationd` on a private bus). Installing from the installer medium onto a blank disk was tested in QEMU (`scripts/qemu-fallback-test`). Not tested: real hardware, Secure Boot, an installer USB written from an installed system.

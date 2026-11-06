# Ersa Compute Platform: architecture

Status date: 2026-11-05. This is the consolidated picture. Each section says what is built, what has
been tested and what is still a plan, because the three are different things. Detail lives in
[README.md](../README.md), [foundation/README.md](../foundation/README.md),
[foundationd.md](foundationd.md), [provisioning.md](provisioning.md) and [installer.md](installer.md).

## In one paragraph

The Ersa Compute Platform is an **immutable, image-based GNOME desktop OS** built with mkosi from
openSUSE Tumbleweed. The operating system is a read-only EROFS image in one of two **A/B root
partitions**, started by systemd-boot through a **unified kernel image (UKI)** with boot counting and
automatic fallback. Everything writable lives in separate persistent areas that survive OS
replacement. A single root service, **`foundationd`**, is the only supported way to change the system
(slots, applications, compatibility packages, system extensions, settings); `foundation` and any
graphical client are thin D-Bus clients of it.

## Layers

```
 ┌───────────────────────────────────────────────────────────────────────────────────┐
 │ Applications   Flatpak (system-wide, /var) · toolbox/distrobox/podman · compat sysexts │
 ├───────────────────────────────────────────────────────────────────────────────────┤
 │ Desktop        GNOME Shell + GDM · GNOME Initial Setup (first boot) · Ghostty         │
 ├───────────────────────────────────────────────────────────────────────────────────┤
 │ Management     foundationd (root, polkit, jobs/workers)  ←  foundation CLI, GUIs, MDM  │
 ├───────────────────────────────────────────────────────────────────────────────────┤
 │ System         systemd · dbus-broker · NetworkManager · PipeWire · SELinux (targeted)  │
 ├───────────────────────────────────────────────────────────────────────────────────┤
 │ Base image     read-only EROFS root, built from Tumbleweed RPMs by mkosi               │
 ├───────────────────────────────────────────────────────────────────────────────────┤
 │ Boot           UEFI → systemd-boot → UKI (kernel + initrd + command line) → initrd     │
 └───────────────────────────────────────────────────────────────────────────────────┘
```

## Boot chain

1. **UEFI** starts `systemd-boot` from the ESP.
2. **systemd-boot** lists the UKIs `ecp_<version>+<tries>.efi`, picks the default (or a one-shot
   choice), and **counts tries down**. A UKI that never reaches a good boot is skipped after its
   tries run out, so boot falls back to the previous entry.
3. The **UKI** carries the kernel, the initrd and a command line containing
   `root=PARTLABEL=ecp-root_<version>`, so booting a UKI selects its slot.
4. The **initrd** mounts the root (EROFS, read-only) and `/var` (btrfs subvolume), both named on the kernel
   command line (`root=` and `systemd.mount-extra=`; an installed system uses its own PARTUUIDs, see below),
   then builds the **`/etc` overlay** (`sysroot-etc.mount`), and switches root.
5. **systemd** starts the system. `systemd-bless-boot` marks the entry good once the graphical target
   is reached with no failed units (the counter disappears from the file name).

Secure Boot, TPM measurement and signing are **not enabled** (see Security).

## Storage layout

| Partition | Filesystem | Holds | Survives OS replacement |
| --- | --- | --- | --- |
| ESP, 1 GiB | FAT | systemd-boot, one UKI per slot | yes |
| Recovery, 1 GiB | none (reserved) | the future recovery system | n/a |
| Root slot A, 8 GiB | EROFS, read-only | the running OS image, label `ecp-root_<version>` | replaced |
| Root slot B, 8 GiB | EROFS or empty (`_empty`) | the other version | replaced |
| `ECP-DATA`, 10 GiB or more, last | btrfs | subvolume `var` → `/var` (logs, Flatpak apps and runtimes, container storage, **the `/etc` overlay's upper layer**, foundationd state) and subvolume `home` → `/home` (user data) | yes |

Total about 28 GiB. `/tmp` is a tmpfs.

**`/etc` is an overlay.** The image's `/etc` is the read-only lower layer; local changes (users,
network settings, the machine ID, SSH host keys) are the upper layer on `/var`. New defaults from an
OS update therefore appear automatically while local changes stay on top.

**One data volume, two lifecycles.** `/var` is mostly *system state* (regenerable) and `/home` is user
data (irreplaceable); they share one btrfs volume so free space is shared, but as separate subvolumes
they are snapshotted, backed up or reset independently (a factory reset wipes `var`, keeps `home`).
Each subvolume is a real mount (`subvol=var`, `subvol=home`), so SELinux labels `/home` by its own
path; a bind mount of `/var/home` would not. Not set up yet: quotas (btrfs has no reserved space, so a
full `/home` can starve system services without them), compression (left off: read speed first), and
first-boot growth of the volume to fill a larger disk.

**Installer image.** The same OS in a 6 GiB layout (ESP 512 MiB, one 5 GiB root, a 512 MiB data volume for the live
session) built with `scripts/build x86-64 --profile installer`; its root is labelled `ecp-live_<version>` so an installed
disk is never mistaken for it. The normal image stays for QEMU testing ([installer.md](installer.md)).

**Which disk is mounted.** Nothing is found by a label that another disk could share. A built image's
command line names its root and data volume by label (fine with one disk). `foundation deploy` rewrites the copied
kernel image's command line for the new install: `root=PARTUUID=…` and `systemd.mount-extra=PARTUUID=…:/var…` /
`…:/home…` with UUIDs it generated and pinned when creating the partitions (`deploy.py`, `installed_cmdline`), so an
installer USB left in, or a second install, cannot supply the wrong `/var` or `/home`. The installer medium's root
is labelled `ecp-live_<version>`, not `ecp-root_<version>`, so the two can be told apart. **Known gap:** kernel images that arrive later by OTA
update are built with the label form again, so after an update the ambiguity can return when another ECP disk is
attached at boot; the fix is a per-install command line addon on the ESP, not done yet.

## Updates and rollback (A/B)

- **Model:** the running slot is never modified. A new version is written to the other slot together
  with its UKI (`+3` tries); the next boot tries it and falls back if it fails.
- **Rollback:** boot the previous slot (`foundation slot boot <slot>`, once or persistently).
- **Slot tools (built, tested in a VM):** `slot status`, `slot boot [--once]`, `slot copy`
  (clones a slot as `<name>-copy` with its own label and kernel image), `slot release`.
- **Update payload (built, not tested):** `scripts/build x86-64 --profile root` produces the compressed
  root image, the kernel image and checksums named for the existing transfer files; see the README.
- **Over-the-air updates: not built.** The transfer files exist but point at a placeholder URL, the
  `Updates` interface is only designed, and there is no update server or signing. `systemd-sysupdate`
  and its D-Bus service `sysupdated` are shipped by Tumbleweed and are the intended engine.
- **Later:** zstd/lz4 compressed EROFS; delta updates (chunk-based) once images are measured.

## Management plane

`foundationd` (Python, GLib/GDBus) is a D-Bus service `org.ersa.Foundation1`. It validates each
request, checks polkit **asynchronously** (one action per kind of change), takes a named lock and
starts a **worker process** for the real work, then reports steps, progress and the result as a Job
object. The service itself never blocks. `foundation` (the CLI) and any future client use only the
contract in `foundation/data/dbus/org.ersa.Foundation1.xml`.

| Interface | What it does | Backend |
| --- | --- | --- |
| `Manager` | status, system facts, base packages, jobs, provisioning | files, rpm |
| `Config` | settings with a schema | layered files |
| `Boot` | slots, boot selection, slot copy/release, reboot | GPT + ESP + EFI variables, `bootctl`, `sfdisk`, logind (D-Bus) |
| `Apps` | system-wide Flatpaks | libflatpak (search/info via the `flatpak` command) |
| `Compat` | legacy RPMs as system extensions | `rpm`, a strict RPM unpacker, `systemd-sysext` |
| `Extensions` | sysext/confext images | `systemd-sysext` / `systemd-confext` |
| `Job` | progress, logs, cancel | the broker |

Design rule: prefer D-Bus, a library or reading files; run a command only where nothing else exists.
Not yet present: `Updates`, `Deploy` (installer), `Diagnostics`.

## Software model

| Kind | Where | How it changes |
| --- | --- | --- |
| **Base system** | the read-only root image | only by replacing the image (build or update); never on a running system |
| **Applications** | Flatpak, system-wide, `/var/lib/flatpak` | `foundation apps ...` |
| **Legacy/third-party RPMs** | system extensions in `/var/lib/extensions`, overlaid on `/usr` | `foundation compat ...`; scriptlets are never run; checked against the base first |
| **Development environments** | toolbox, distrobox, podman containers | the user |
| **Local configuration** | the `/etc` overlay and `/var` | the user and `foundation config` |

## Provisioning and identity

- GNOME Initial Setup runs on first boot with every page kept; the image's defaults are
  `en_US.UTF-8` and the `us` keyboard (the time zone defaults to UTC).
- `system.provisioned` (Android's `DEVICE_PROVISIONED`) is a one-way latch set when the setup is done:
  a user unit calls `Manager.CheckProvisioning` after login (evidence: an administrator's
  `gnome-initial-setup-done` file), or a front end calls `CompleteProvisioning`.
- No secrets and no machine identity are in the image. The machine ID is created on first boot into
  the `/etc` overlay.
- **MDM/enrolment** is a plan: an agent as one more `foundationd` client, a first-boot front end
  chosen by whether enrolment must precede the account ([provisioning.md](provisioning.md)).

## Security model

| Layer | State |
| --- | --- |
| Read-only EROFS root, writable state elsewhere | built |
| SELinux (targeted policy, relabelled at build) | configured; behaviour of `foundationd` and compat files under enforcing **untested** |
| polkit on every state-changing call, files passed as descriptors, strict RPM unpacking | built and tested |
| Signed update artifacts, signed UKI, Secure Boot | **not built**; plan: `mkosi` signing, root derived from the booted entry so a copied UKI stays signed |
| dm-verity sealed root (macOS-style) | **not built** |
| TPM measurement, disk encryption | not built |

## Build and release

- **mkosi 27.1** builds the image from the Tumbleweed OSS repository with a Tumbleweed tools tree;
  `scripts/build x86-64|arm64` is the entry point (`scripts/prepare-build` builds the logos and the
  RPMs first; `scripts/clean` removes outputs).
- **Custom packages** (`packages/`): `foundation`, `foundationd`, `ersa-boot-splash`,
  `ersa-bluetooth`, `ersa-dialer`, built as RPMs and consumed from `mkosi.packages/`.
- **Tests:** about 260 automated tests (`foundation/tests`), run on the host and on Tumbleweed, and in
  the package build (`%check`). Boot behaviour is verified by hand in a QEMU/OVMF VM.
- A pinned repository snapshot, signed repository publication and a CI-built release do not exist yet.

## Decision log

| Date | Decision | Why |
| --- | --- | --- |
| 2026-11-04 | Base is openSUSE Tumbleweed (moved from Rocky 10) | rolling base, current GNOME and kernel; reconfirmed 2026-11-05 |
| 2026-11-03 | GNOME replaces KDE | the KDE image did not reach a login |
| 2026-11-05 | Two fixed 8 GiB A/B root partitions, EROFS | robustness: the kernel finds the root with no custom code; a shared image pool was prototyped and set aside |
| 2026-11-05 | `/etc` is an overlay, no `/etc` partition | OS updates deliver new defaults; local changes persist |
| 2026-11-05 | One btrfs data volume (`/var` and `/home` subvolumes), at least 10 GiB | Flatpak data lives in `/var` (2 GiB made an install fail); shared space, separate lifecycles, real mounts for SELinux |
| 2026-11-05 | `foundationd` D-Bus service is the single management plane | one authorized, audited path |
| 2026-11-05 | "ECP" only in code; "Ersa Compute Platform" in anything a person reads | naming rule |
| later | compression (lz4 for read speed), delta updates, signing | deferred until the boot path is proven |

## What is verified, and what is not

**Verified in a VM:** boot through systemd-boot, boot counting and blessing, `foundationd` started by
D-Bus activation, `slot copy`, booting a copy (one-shot, counter-free entry id), `apps setup`; the
btrfs data volume (`/var` mounted from the `var` subvolume in the initrd) and the `/etc` overlay
(assembled in the initrd, booting under enforcing SELinux). Two bugs found and fixed on the way: my
overlay unit conflicted with `initrd-switch-root.target` (it unmounted `/etc` before switch-root), and
the overlay's root takes the label of its upper directory, so `/etc` was `var_lib_t` and SELinux
denied `useradd` and `dbus-broker`. Relabelling the upper layer on the running disk fixed it.

**Built, fix not yet seen in a fresh build:** the build-time label for the overlay's upper layer
(`file_contexts.subs` plus `mkosi.finalize.chroot`); the provisioning hook; `compat` merging;
persistent `slot boot`; Flatpak installs on the larger `/var`; Firefox from the image.

**Installer, deploy, repositories (added since):** `foundation deploy` and the graphical installer were run in QEMU
(install onto a virtual disk, boot it alone through UEFI: language and time zone applied, no live user, a user's home created,
the entry blessed); the live medium booted and installed. A read-only live medium was tried and removed (emergency mode,
[features/live-medium.md](features/live-medium.md)). Repositories and bundles were built and verified against the live
openSUSE and Packman repositories in a container, not yet on a booted image.

**Not built:** OTA updates (scaffolding only), diagnostics, recovery system, signing and Secure Boot, dm-verity, MDM,
per-worker sandboxes, encryption, aarch64 and device kernels.

## Order of work

1. Rebuild and boot; install a bundle (`foundation compat install @codecs`) on a booted image under enforcing SELinux.
2. Prove OTA against a local server in QEMU (`systemd-sysupdate`, then `Updates` via `sysupdated`), including the
   per-install kernel command line surviving an update.
3. Signing, then the "root from the booted entry" change that keeps a copied UKI signed; then dm-verity.
4. An automated install-then-boot check in `scripts/qemu-smoke`.
5. Encryption and a choice of data filesystem; quotas; compression and deltas when measured.
6. First-boot front end and MDM, once the enrolment requirement is known.

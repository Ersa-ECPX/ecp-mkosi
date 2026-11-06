# ECP — Ersa Compute Platform

Initial **experimental** mkosi configuration for an openSUSE Tumbleweed GNOME desktop (rolling release; the base moved from Rocky Linux 10 on 2026-11-04). EAP (Ersa Automotive Platform) is a separate Yocto project.

Device-specific ECP sources will be organized under [devices/](devices/README.md) after the x86-64 foundation boots. No phone target is enabled yet.

Documentation for every feature, with what is and is not tested: [docs/README.md](docs/README.md).

## Architecture

The consolidated picture (layers, boot chain, storage, updates, management plane, security, decisions and what is verified) is in [docs/architecture.md](docs/architecture.md).

## Repositories

The platform is three repositories under [Ersa-ECPX](https://github.com/Ersa-ECPX):

| Repository | Contents |
| --- | --- |
| `ecp-mkosi` (this one) | the mkosi configuration (`mkosi.*`), `branding/`, `scripts/` (build, test and release tools), `docs/` (architecture and feature documentation, the wiki source), `AGENTS.md` |
| [`foundation`](https://github.com/Ersa-ECPX/foundation) | `foundation` (command line), `foundationd` (the D-Bus service), the graphical installer, the D-Bus contract and the unit tests |
| [`packages`](https://github.com/Ersa-ECPX/packages) | RPM specs and the small sources of Ersa packages (`ersa-boot-splash`, ...), and the workflow that builds the RPM repository |

A fresh clone of this repository builds an image with no other setup. `scripts/prepare-build` (run by `scripts/build`) calls `scripts/fetch-sources`, which clones what is missing: `foundation/` and `packages/` (git-ignored checkouts inside this one) and the sources of the compiled packages into `.sources/` (`ersa-bluetooth`, `dialer`, built at the commit each spec pins; a checkout in `~/Dev/` is used instead if you have one). Existing checkouts are never touched, so your own changes survive; `ECP_UPDATE=1 scripts/build ...` fast-forwards them first.

```sh
git clone https://github.com/Ersa-ECPX/ecp-mkosi.git && cd ecp-mkosi
scripts/build x86-64 --profile installer      # installer medium, mkosi.output/
scripts/build x86-64 --profile root --image-version 0.1.1    # the payload of an update
```

`scripts/build-foundation-package` builds the `foundation` RPMs from the working tree of `foundation/` using the spec in `packages/foundation/`; `scripts/build-splash-package` builds `ersa-boot-splash` from `packages/ersa-boot-splash/` and `branding/`; `scripts/build-wiki` reads `foundation/README.md`. The RPM repository published by the `packages` workflow is **not** consumed by the image build (nothing here sets it as an mkosi `Repositories=` source): packages are compiled locally from the cloned sources.

### Releases

`scripts/release 0.1.1` builds the installer image and the update for that version and uploads them to a GitHub release `v0.1.1` of this repository (`--draft` to review first, `--no-build` to reuse existing output). Images are compressed with zstd and split into `.partNN` files if they exceed GitHub's 2 GiB per-file limit. Releases are marked pre-release and are **not signed**: install an update with `foundation update install FILE --unsigned` until an image signing key exists.

The foundation tests that need this repository (kernel command line, partition layout, wiki generator, a built RPM) skip unless `ECP_MKOSI_DIR` points at a checkout of it: `ECP_MKOSI_DIR=$PWD python3 -m unittest discover -s foundation/tests`. Run them this way after changing `mkosi.conf.d/`, `mkosi.repart.system/` or `docs/`.

## Status

This repository defines a raw UEFI GPT image with systemd-boot and a destructive raw writer. It has **not** yet passed an mkosi build or boot test. Do not treat the installation path as a released installer or the update design as an operational atomic updater. The raw writer is suitable only for a dedicated disk whose data can be erased.

| Target | mkosi identifier | RPM architecture | CPU requirement |
| --- | --- | --- | --- |
| AMD/Intel | `x86-64` | `x86_64` | x86-64 (see the v3 note below) |
| ARM | `arm64` | `aarch64` | ARMv8-A or newer |

**x86-64-v3:** Tumbleweed is not built with an x86-64-v3 baseline. In the repository snapshot I checked (2026-11-02), 124 of 27,166 `x86_64` packages (about 0.5%) had optional `x86-64-v3` builds (compression, crypto and codec libraries, boost, Python's library) installed through `patterns-glibc-hwcaps-x86_64_v3`. Mesa, GNOME Shell, Mutter, GLib, GTK 4, PipeWire and the kernel are not v3 builds. A v3-optimized kernel is a requirement that is not met yet. The Rocky/EL10 family (full v3 baseline, GNOME 49, kernel 6.12) remains the fallback; the previous Rocky configuration is in git history.

## Prerequisites and build

Use Linux with user namespaces, systemd >= 254, Python, `rpmbuild`, `rsvg-convert` (librsvg), podman (from the RPM build tools), QEMU/OVMF for VM tests, and enough disk space for a raw image of about 28 GiB (sparse: only written blocks use space) plus build caches. Install **mkosi 27.1** from its upstream Git tag in an isolated environment: `pipx install 'git+https://github.com/systemd/mkosi.git@v27.1'`; `scripts/build` rejects other versions. A foreign architecture build also needs registered `qemu-user-static` binfmt support. The tools tree is openSUSE Tumbleweed (`ToolsTreeDistribution=opensuse` in `mkosi.conf`), so the image is installed by Tumbleweed's own `zypper` and `rpm`; mkosi downloads it on the first build. See [mkosi installation](https://github.com/systemd/mkosi#installation) and [mkosi requirements](https://github.com/systemd/mkosi/blob/v27.1/mkosi/resources/man/mkosi.1.md).

```sh
scripts/build x86-64
scripts/build arm64
```

The tools tree supplies zypper and rpm for the openSUSE build. A Fedora 45 tools tree (rpm 6.1.0) stalled for hours installing the `perl` package and rejected some Tumbleweed signatures, which is why the tools tree matches the target; delete `mkosi.tools` and `mkosi.tools.manifest` after changing it (or run `scripts/clean --all`). `scripts/build` keeps its scratch space in `/tmp/ecp-work` (override with `ECP_WORK_DIR`; roughly 15 GiB). If your home or `/var/tmp` is restricted, place mkosi's cache and temporary files in writable paths:

```sh
mkdir -p /tmp/ecp-cache /tmp/ecp-tmp
XDG_CACHE_HOME=/tmp/ecp-cache TMPDIR=/tmp/ecp-tmp \
  MKOSI_FORCE_USERNS_FALLBACK=1 scripts/build x86-64
```

### Podman builder

`scripts/build-podman x86-64` runs mkosi in the pinned Fedora 45 container from `Containerfile.build` instead of on the host. The prep steps (logos, locally built RPMs) run on the host first. The container is privileged because mkosi creates filesystems and nested sandboxes, and it mounts this repository read/write, so use it only with a trusted ECP checkout.

`scripts/build` and `scripts/build-podman` both run `scripts/prepare-build`, which renders the logos (`scripts/build-branding`), builds the noarch RPMs (`scripts/build-splash-package`, `scripts/build-foundation-package`) and builds the compiled RPMs in a Tumbleweed container (`scripts/build-rpms`). A `mkosi.branding/` directory left behind by an earlier build that ran as root must be removed with `sudo rm -rf mkosi.branding`; the script says so.

The image uses SELinux, which is openSUSE Tumbleweed's default for new installs (since snapshot 20250211, [openSUSE News](https://news.opensuse.org/2025/02/13/tw-plans-to-adopt-selinux-as-default/)): the policy comes from `selinux-policy-targeted`, mkosi relabels the tree at build time (`SELinuxRelabel=yes`) and the kernel command line has `security=selinux selinux=1`. The Rocky builds had a `setfiles ... Operation not permitted` failure in restricted build environments; whether that recurs with openSUSE's policy is untested.

This builds `mkosi.output/ecp_0.1.0.raw`, a bootable **disk image**, not an ISO. `Format=esp` would only make an ESP image and would omit ECP's root and persistent data layout.

The image uses the openSUSE Tumbleweed OSS repository (snapshot of 2026-11-02: kernel 7.3.1, GNOME Shell 51.1, Mesa 26.2.5, systemd 261.4, PipeWire 1.6.11). Every package name in `mkosi.conf` was checked against that repository's metadata, and the compiled ECP RPMs were built in a Tumbleweed container; a full image build has **not** been completed yet, so the first build may need fixes (kernel installation under mkosi, firmware sets and the initrd modules are the likeliest).

## Storage and identity

The writable data volume is btrfs (subvolumes, so `/var` and `/home` share space without bind mounts); repart populates it offline (checked with a scratch image). Btrfs snapshots alone would not make the root update atomic, which is why the root is a separate read-only EROFS image. Btrfs snapshots alone would not make kernel and ESP updates atomic; ZFS adds an out-of-tree kernel dependency without a demonstrated need. The intended GPT layout is:

| Partition | Filesystem | Purpose |
| --- | --- | --- |
| ESP, 1 GiB | FAT | UEFI boot files and one UKI per root slot |
| ECP-RECOVERY, 1 GiB | none (reserved) | recovery environment, not implemented yet |
| root slot A, 8 GiB | EROFS (uncompressed), read-only | the running OS, labeled `ecp-root_<version>` |
| root slot B, 8 GiB | empty, labeled `_empty` | inactive slot that `systemd-sysupdate` writes the next version into |
| ECP-DATA, at least 10 GiB, last | btrfs | one volume with two subvolumes: `var` (mounted at `/var`: logs, Flatpak applications and runtimes, container storage, the `/etc` overlay's upper layer) and `home` (mounted at `/home`: user data). They share free space and can be snapshotted, backed up or reset separately. |

`/var` (the btrfs subvolume `var`) is mounted in the initrd (`systemd.mount-extra=` on the kernel command line, see `mkosi.conf.d/30-data-mounts.conf`; there is no `/etc/fstab` entry), and the `/etc` overlay is built on it there. Without a writable `/etc` early, PID 1 starts with the image's placeholder `/etc/machine-id` (`uninitialized`) and `dbus-broker` refuses to start. `/tmp` is a tmpfs because the root is read-only.

`/etc` is an **overlay**, assembled in the initrd (`mkosi.initrd.extra/usr/lib/systemd/system/sysroot-etc.mount`): the image's own `/etc` is the read-only lower layer, and the persistent upper layer lives on the `/var` partition (`/var/lib/ecp/etc/upper`). New defaults from an OS update therefore appear automatically, and local changes (users, network settings, the machine ID, SSH host keys) stay on top and survive updates. The overlay driver is a kernel module, so `/fs/overlayfs/` is in `KernelInitrdModules`. **Unverified:** the initrd ordering of the overlay unit, SELinux labels of files created in the upper layer, and the machine ID being committed on first boot; test these first on the next build.

mkosi leaves `/etc/machine-id` marked uninitialized unless a machine ID is explicitly set. No password, SSH key, or machine identity is committed to the image. `systemd-firstboot.service` is disabled and masked (its interactive wizard hangs the splash); GNOME Initial Setup creates the first account. `/etc` (its upper layer), `/var` and `/home` survive OS image replacement.

The image includes the `ersa-boot-splash` noarch RPM for the Ersa Plymouth theme. `scripts/build` builds this RPM from `packages/ersa-boot-splash/` with `rpmbuild` and supplies it to mkosi as a local package. Mkosi overlays the theme selection config into both the root image and initrd. The theme source and redistribution limit are recorded in [docs/branding.md](docs/branding.md). The theme's appearance in the initrd has not been visually tested.

The home partition is last and may consume available space during image creation. It is not automatically expanded by `install-raw` when installed on a larger disk. Back up all three persistent partitions before changing their sizes or replacing a device. A built image names its data volume by label; an installed system names it by PARTUUID (rewritten into the kernel image by `foundation deploy`), so attaching the installer medium or a second install does not mix their `/var` and `/home`.

## Defaults

The image's locale is `en_US.UTF-8`, and the keyboard layout `us`. The time zone defaults to UTC (an `Asia/Kolkata` default broke the build because the zone files were missing from the build root); users set it in GNOME Initial Setup, which keeps all its pages. The login banner on the console reads "Welcome to Ersa Compute Platform <image version> - Kernel ..." (the package's own banner is replaced in `mkosi.extra/usr/lib/issue.d/` and `mkosi.extra/etc/issue.net`). Text a person reads says "Ersa Compute Platform"; `ECP` is for code and identifiers.

## Packages and apps

`mkosi.conf` installs, from the Tumbleweed OSS repository (every name checked against the repository metadata, not by a completed build):

- **Kernel and firmware:** `kernel-default` (7.3.1 in the checked snapshot, **not** compiled for x86-64-v3), CPU microcode and the firmware sets for common PC hardware (Intel, AMD GPU, Wi-Fi, Bluetooth, audio).
- **Update tools:** `systemd-experimental` (ships `systemd-sysupdate`), `udev` (ships `systemd-repart`, `bootctl`, `systemd-bless-boot`), `systemd-boot`, `systemd-ukify`.
- **Networking and Bluetooth:** NetworkManager with Bluetooth (PAN/DUN) and WWAN support, ModemManager, firewalld, BlueZ with `bluez-obexd` (PBAP for phonebooks) and `gnome-bluetooth`. PipeWire's Bluetooth plugin is in `pipewire-spa-plugins-0_2`.
- **Files and disks:** Nautilus with Sushi preview, File Roller and gvfs (MTP phones, iPhone via `gvfs-backend-afc`, SMB shares, gphoto, GNOME Online Accounts, FUSE), GNOME Disks with udisks2, Disk Usage Analyzer, and the ext4/XFS/FAT/exFAT/NTFS tools.
- **Apps:** Text Editor, Loupe, Papers, Showtime (video), Calculator, Clocks, Characters, Contacts, Calendar, Weather, Maps, System Monitor, Font Viewer, Extensions, Tweaks, Camera (Snapshot), the Ghostty terminal and Flatpak. There is no app store yet: applications are installed with `foundation apps install`.
- **Developer tools:** git, Python 3 (3.13), gcc, g++, clang, make, cmake, gdb, podman, toolbox and distrobox. Heavy toolchains such as an AOSP build belong in toolbox containers, not the base image.
- **Fonts:** Adwaita fonts, DejaVu, Liberation, Noto sans, serif, mono and color emoji.
- **Branding:** the `-upstream` (neutral) branding packages for GDM and Plymouth, replaced by the Ersa theme and logos.

Zen Browser is not in the Tumbleweed repository (its Firefox is `MozillaFirefox`); it is intended to be a Flatpak (`app.zen_browser.zen`).

### Foundation

`foundation` is the command line of the Ersa Compute Platform and `foundationd` is the management service behind it (Python, GLib/GDBus, polkit). The command line is only a client: every change is a request to the service over D-Bus, which authorizes it with polkit and runs it as a job in a separate worker process. The full guide (architecture, how a request travels, how to run it, test it and add to it) is [foundation/README.md](https://github.com/Ersa-ECPX/foundation/blob/main/README.md); the contract is [foundation/data/dbus/org.ersa.Foundation1.xml](https://github.com/Ersa-ECPX/foundation/blob/main/data/dbus/org.ersa.Foundation1.xml); the design notes are in [docs/foundationd.md](docs/foundationd.md).

```sh
foundation status                       # system, running boot slot, storage, services
foundation slot status                  # which slot you are on, boot assessment, every slot
foundation slot boot 0.2.0-copy --once  # try another slot at the next boot only
foundation slot copy --to empty2        # copy the running slot onto another (with a preview and a confirmation)
foundation apps install app.zen_browser.zen
foundation compat install legacy.rpm    # run an RPM as a system extension, base untouched
foundation ext list                     # sysext and confext images
foundation config set ota.channel testing
```

Implemented: boot state and slots (selection, copy, release), reboot, Flatpak applications (libflatpak), compat RPMs as system extensions, sysext/confext management, settings, and jobs with progress and cancellation. Not implemented: OTA updates (`Updates`), installing to a disk (`Deploy`), diagnostics, the recovery entry. The unit and test results are listed in [foundation/README.md](https://github.com/Ersa-ECPX/foundation/blob/main/README.md#verification-status); read that list before trusting any of it on hardware.

### Calls from the desktop through a paired phone

`ersa-bluetooth` (the `org.openersa.Bluetooth` D-Bus bridge for HFP call control, PBAP contacts and call history, AVRCP media control republished as MPRIS, and ANCS notifications) and `ersa-dialer` (a GNOME dialer that is a client of it) let a desktop answer, decline and place calls through a Bluetooth-paired phone, like macOS does with an iPhone. They are part of the default image. The specs are `ersa-bluetooth/ersa-bluetooth.spec` and `ersa-dialer/ersa-dialer.spec` in the `packages` repository, each pinned to an exact upstream commit. They compile native code, so `scripts/build-rpms` builds them in an openSUSE Tumbleweed container (needs podman) from `~/Dev/ersa-bluetooth` and `~/Dev/dialer` and puts the RPMs in `mkosi.packages/`. `scripts/build` runs it automatically when the RPMs are missing or the pinned commit changed.

The user preset enables `ersa-bluetooth.service` for the desktop user. Open items: `ersa-bluetooth` has no license file (its PKGBUILD says `unknown`), and no pairing, call or phonebook flow has been tested against a real phone.

## Local VM test

After a successful build, locate the `.raw` file and boot a **copy** with UEFI firmware. On x86_64 use a v3 capable virtual CPU (`-cpu host` with KVM on a compatible host, or an emulated Haswell or newer CPU):

```sh
cp mkosi.output/ecp_0.1.0.raw /tmp/ecp-test.raw
qemu-system-x86_64 -machine q35,accel=kvm -cpu host -m 4096 \
  -drive if=pflash,format=raw,readonly=on,file=/usr/share/edk2/x64/OVMF_CODE.4m.fd \
  -drive file=/tmp/ecp-test.raw,format=raw,if=virtio -serial mon:stdio
```

OVMF paths vary by host. For AArch64 use `qemu-system-aarch64 -machine virt -cpu cortex-a72` with AArch64 EDK2 firmware; check firmware path and graphics support locally. Verify bootloader, root mounted read-only, `/etc`/`/var`/`/home` mounted writable, machine ID persistence across reboot, first-boot credential flow, NetworkManager, GDM, and a GNOME Wayland session. These tests have **not** yet run.

## GitHub Actions build

`.github/workflows/build-x86_64.yml` builds the normal labeled x86-64 image on an Ubuntu 24.04 GitHub-hosted VM for pushes, pull requests, or manual dispatch. It runs the same `scripts/build x86-64` entry point with `SELinuxRelabel=yes`; it does not disable SELinux relabeling or produce an unlabeled VM image. A successful run uploads a zstd-compressed raw image, its SHA-256 checksum, and mkosi build metadata for 14 days. The first run will show whether the GitHub runner's filesystem and kernel permit `setfiles` to write labels; if that step fails, the workflow reports the limitation rather than bypassing it. This workflow currently covers x86-64 only.

## Installer image (profile)

A second image, with the same OS in a layout small enough for a USB stick, is built with the `installer` mkosi profile; the normal image stays the one to boot in QEMU for testing:

```sh
scripts/build x86-64                        # the normal image: mkosi.output/ecp_0.1.0.raw, 28 GiB layout (A/B slots)
scripts/build x86-64 --profile installer    # the installer image: mkosi.output/ecp-installer_0.1.0.raw, 6 GiB
```

| Partition | Normal image (`mkosi.repart.system/`) | Installer image (`mkosi.repart.installer/`) |
| --- | --- | --- |
| ESP | 1 GiB | 512 MiB |
| Recovery | 1 GiB (reserved) | none |
| Root | two 8 GiB slots (A/B) | one 5 GiB slot (nothing on the stick is updated) |
| Data (btrfs, `/var` and `/home`) | 10 GiB or more 512 MiB, holds the live session's `/var` and `/home` and the installer sysext |

mkosi merges `mkosi.repart/` with anything a profile adds, so the default layout lives in `mkosi.repart.system/` and is selected by `mkosi.conf.d/20-repart.conf` unless the installer profile is active. The installer app and `foundation deploy` are built (see [docs/installer.md](docs/installer.md)); the installer medium keeps a small writable data volume for the live session. The 5 GiB root fails the build, rather than truncating, if the OS image ever outgrows it.

## Update payload (profile)

The `root` profile builds only what an update needs, so it can be published and written to the inactive slot by `systemd-sysupdate` (and `foundation`, through it):

```sh
scripts/build x86-64 --profile root
```

| File in `mkosi.output/root/` | What it is | Matched by |
| --- | --- | --- |
| `ecp_<version>.root-x86-64.raw.zst` | the root filesystem: read-only EROFS, exactly as large as its contents (`Minimize=best`), compressed with zstd | `sysupdate.d/10-root.transfer` |
| `ecp_<version>.efi` | the kernel image; the transfer installs it as `ecp_<version>+3.efi` for boot counting | `sysupdate.d/20-uki.transfer` |
| `ecp_<version>.SHA256SUMS` | checksums of the files above | see below |

The normal image has the same output name, so the profile writes to its own directory. Its disk (a 260 MiB ESP and the minimal root) is a by-product that is never booted. The layout is `mkosi.repart.root/`.

Publishing and testing: `scripts/publish-update` merges the payloads into one `SHA256SUMS` (and signs it with `--sign`), `scripts/serve-update` serves them over HTTP for a test, and `scripts/ota-test-definitions` writes test transfer files for a QEMU guest. The procedure is in [docs/ota-test.md](docs/ota-test.md) (not run yet), and [docs/ota-hosting.md](docs/ota-hosting.md) compares places to host updates.

Still missing: signing keys and the public key in the image, a real URL (the transfer files point at the placeholder `https://updates.ersa.invalid/ecp/x86-64/`), and `foundation` calling `sysupdated` (`Updates` is only designed).

## Installation

Anaconda's `liveimg` payload can install a tarred root filesystem, but it cannot consume this complete mkosi GPT raw image as its payload. Its normal UEFI installation flow would also install a different boot stack and would not preserve this image's systemd-boot entry and read-only partition flags. ECP therefore has no Anaconda ISO yet. An Anaconda integration must be tested with a custom storage and bootloader handoff that retains systemd-boot, the exact GPT layout, persistent boundaries, and first-boot identity flow. Calamares would need comparable work and has no present advantage. [RHEL 10 `liveimg` specification](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html-single/automatically_installing_rhel/automatically_installing_rhel), [mkosi disk output](https://github.com/systemd/mkosi/blob/v27.1/mkosi/resources/man/mkosi.1.md).

For a disposable VM or dedicated disk, boot a rescue/live environment, inspect the target with `lsblk`, and write the complete image:

```sh
sudo scripts/install-raw --erase mkosi.output/ecp_0.1.0.raw /dev/DISK
```

This **erases the entire device**. The script rejects mounted or undersized targets, but it cannot identify the device for you. It does not preserve existing data or partition around another OS. The first boot needs an interactive console to complete local configuration. The raw writer is untested on real hardware.

## Updates and recovery design

The update mechanism is **A/B root slots plus one UKI per slot**, using tools in openSUSE's systemd 261 (`systemd-sysupdate` is in the `systemd-experimental` package):

- Each UKI is named `ecp_<version>+3.efi` and its command line has `root=PARTLABEL=ecp-root_<version>`, so booting a UKI selects its slot. `/etc/kernel/tries` sets the initial boot counter to 3.
- `systemd-boot` counts tries down; `systemd-bless-boot` marks the entry good after the graphical target is reached with no failed units (`graphical.target.d/10-ecp-boot-assessment.conf`, `systemd-boot-check-no-failures`). After three failed tries the bootloader falls back to the previous UKI.
- `mkosi.extra/usr/lib/sysupdate.d/` has the root and UKI transfers. `SplitArtifacts=uki,partitions` makes mkosi emit their payloads. The source URL is a placeholder (`updates.ersa.invalid`) and the split file names in the `MatchPattern` lines are unverified guesses until a build is inspected.

Status: configured, **not built, not booted, not fault-tested**. Gates before this is called an updater: a build that produces both slots, a boot into slot A, `bootctl list` showing the entry blessed, a second version written to slot B by `systemd-sysupdate`, a forced failed boot falling back to slot A, and an interrupted update leaving the running slot intact. There are no signed update artifacts, no update server, no dm-verity, and no Secure Boot keys. `/etc`, `/var`, and `/home` live outside both slots, so a root rollback cannot undo changes to them, and image updates do not refresh `/etc` defaults yet. The 1 GiB recovery partition is reserved only. [systemd-sysupdate](https://www.freedesktop.org/software/systemd/man/261/systemd-sysupdate.html), [systemd boot blessing](https://www.freedesktop.org/software/systemd/man/260/systemd-bless-boot.service.html).

Secure Boot is the next boot integrity step after a labeled VM boot. mkosi can sign systemd-boot and generated UKIs with `SecureBoot=yes`, `SecureBootKey=`, and `SecureBootCertificate=`; signing keys must be supplied from outside this repository. Test key enrollment in disposable OVMF variables before touching hardware firmware. Every updated bootloader and UKI will need signing, while root integrity still requires a separate verity or equivalent design. No Secure Boot keys are generated or enrolled by this repository yet. [mkosi Secure Boot options](https://github.com/systemd/mkosi/blob/v27.1/mkosi/resources/man/mkosi.1.md).

## Version control and validation limits

The mkosi frontend is pinned to 27.1 and the distribution is the rolling openSUSE Tumbleweed OSS repository, so two builds on different days contain different package versions. The generated JSON manifest and checksum identify one build's inputs and artifact; they do not make independent rebuilds bit identical. openSUSE publishes dated Tumbleweed snapshots, but I have not verified their URL layout or used them here. A production release needs a pinned repository snapshot or mirror, pinned tools-tree content, signed artifact metadata, and tested rollback compatibility.

Validation performed here: `bash -n` passed for the shell scripts; mkosi 27.1 `summary` parsed both architecture configurations; the repart layout was checked with `systemd-repart` on a scratch image; every package in `mkosi.conf` exists in the Tumbleweed OSS metadata; and the ECP RPMs build in a Tumbleweed container. The earlier Rocky image was booted in QEMU to `graphical.target` (that is where the first-boot, `/tmp` and machine-id findings came from). **No Tumbleweed disk image has been built or booted.** A build, partition inspection, VM boot, installation on spare media, Plymouth and logo visual checks, and rollback fault injection are required before calling ECP bootable or deployable.

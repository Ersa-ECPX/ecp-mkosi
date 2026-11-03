# ECP — Ersa Compute Platform

Initial **experimental** mkosi configuration for a Rocky Linux 10.2 KDE Plasma desktop. EAP (Ersa Automotive Platform) is a separate Yocto project.

Device-specific ECP sources will be organized under [devices/](devices/README.md) after the x86-64 foundation boots. No phone target is enabled yet.

## Status

This repository defines a raw UEFI GPT image with systemd-boot and a destructive raw writer. It has **not** yet passed an mkosi build or boot test. Do not treat the installation path as a released installer or the update design as an operational atomic updater. The raw writer is suitable only for a dedicated disk whose data can be erased.

| Target | mkosi identifier | Rocky RPM architecture | CPU requirement |
| --- | --- | --- | --- |
| AMD/Intel | `x86-64` | `x86_64` | x86-64-v3 or newer |
| ARM | `arm64` | `aarch64` | ARMv8-A or newer |

`x86-64-v3` is a Rocky CPU baseline, **not** an mkosi architecture identifier. Rocky 10.2 is the latest supported 10 release as of 2026-11-03. [Rocky releases](https://docs.rockylinux.org/releases/), [mkosi architecture values](https://github.com/systemd/mkosi/blob/v27.1/mkosi/resources/man/mkosi.1.md).

## Prerequisites and build

Use Linux with user namespaces, systemd >= 254, Python, DNF, QEMU/OVMF for VM tests, and enough disk space for a roughly 14 GiB raw image plus build caches. Install **mkosi 27.1** from its upstream Git tag in an isolated environment: `pipx install 'git+https://github.com/systemd/mkosi.git@v27.1'`; `scripts/build` rejects other versions. A foreign architecture build also needs registered `qemu-user-static` binfmt support. The Fedora 45 tools tree downloads the needed image tools and fetches its repository signing key. See [mkosi installation](https://github.com/systemd/mkosi#installation) and [mkosi requirements](https://github.com/systemd/mkosi/blob/v27.1/mkosi/resources/man/mkosi.1.md).

```sh
scripts/build x86-64
scripts/build arm64
```

On an Arch host, install `dnf` first. If your home or `/var/tmp` is restricted, place mkosi's cache and temporary files in writable paths:

```sh
mkdir -p /tmp/ecp-cache /tmp/ecp-tmp
XDG_CACHE_HOME=/tmp/ecp-cache TMPDIR=/tmp/ecp-tmp \
  MKOSI_FORCE_USERNS_FALLBACK=1 scripts/build x86-64
```

### Podman builder and SELinux labels

`Containerfile.build` pins mkosi 27.1 in a Fedora 45 build environment. On a host where SELinux labeling works, run:

```sh
scripts/build-podman x86-64
```

The Podman runner uses a privileged container because mkosi creates filesystems and nested sandboxes. It mounts this repository read/write, so use it only with a trusted ECP checkout. A container shares the **host kernel and backing filesystem**: it cannot make SELinux `security.selinux` xattrs work if the host denies them. The reported `setfiles ... Operation not permitted` means the current `SELinuxRelabel=yes` build did **not** produce a correctly labeled image. For a labeled release, use a Rocky/Fedora SELinux-capable builder, with a filesystem that supports labels, and verify the resulting image's labels and enforcing boot. [mkosi SELinux relabel](https://github.com/systemd/mkosi/blob/v27.1/mkosi/resources/man/mkosi.1.md), [Podman privileges](https://docs.podman.io/en/latest/markdown/podman-run.1.html).

For **disposable VM bringup only**, the runner can skip relabeling and set `selinux=0` in a separately named image:

```sh
scripts/build-podman x86-64 unlabeled-vm
```

This produces an `ecp_0.1.0-vm-unlabeled.raw` variant if the other build steps succeed. Do not install or distribute it as an ECP release. SELinux cannot simply be enabled on that read-only root without an offline relabel and an enforcing boot test; the persistent partitions also need correct labels. [RHEL 10 SELinux relabel guidance](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_selinux/changing-selinux-states-and-modes).

This builds `mkosi.output/ecp_0.1.0.raw`, a bootable **disk image**, not an ISO. `Format=esp` would only make an ESP image and would omit ECP's root and persistent data layout.

The image uses Rocky BaseOS/AppStream plus **CRB and EPEL 10**. Rocky publishes KDE 10 live images for both x86_64 and aarch64, and EPEL 10.2 has Plasma and SDDM packages. Rocky's AppStream carries `systemd-boot-unsigned` for x86_64; an AArch64 10.2 AppStream package is visible in Rocky's kickstart tree, but installation from the normal AArch64 repository still needs a build check. This verifies package availability at the distribution level; it does **not** verify this exact package transaction. A successful build and RPM manifest are required before claiming either architecture works. [Rocky image matrix](https://docs.rockylinux.org/latest/teams/rel_eng/image/), [Rocky KDE guide](https://docs.rockylinux.org/es/guides/desktop/kde_installation/), [Plasma package](https://packages.fedoraproject.org/pkgs/plasma-desktop/plasma-desktop/), [SDDM package](https://packages.fedoraproject.org/pkgs/sddm/sddm/), [Rocky x86_64 AppStream](https://dl.rockylinux.org/pub/rocky/10/AppStream/x86_64/os/Packages/s/), [Rocky aarch64 kickstart AppStream](https://download.rockylinux.org/pub/rocky/10.2/AppStream/aarch64/kickstart/Packages/s/).

## Storage and identity

The image uses ext4 because repart can populate it offline and it works with a read-only root. Btrfs snapshots alone would not make kernel and ESP updates atomic; ZFS adds an out-of-tree kernel dependency without a demonstrated need. The intended GPT layout is:

| Partition | Filesystem | Purpose |
| --- | --- | --- |
| ESP, 768 MiB | FAT | UEFI boot files |
| root, 8 GiB | ext4, read-only GPT flag | OS and packaged configuration |
| ECP-ETC, 512 MiB | ext4 | persistent `/etc`, users, network settings, machine ID |
| ECP-VAR, 2 GiB | ext4 | persistent `/var`, logs and application state |
| ECP-HOME, at least 2 GiB | ext4 | persistent `/home` and user data |

The root image carries a copy of `/etc/fstab` so `/etc` can mount early; the separate `/etc` partition is populated with the same initial configuration. mkosi leaves `/etc/machine-id` marked uninitialized unless a machine ID is explicitly set. No password, SSH key, or machine identity is committed to the image. This is an **unvalidated first-boot design**: confirm that Rocky's `systemd-firstboot.service` prompts for a root password and persists the machine ID to ECP-ETC before relying on it. A normal user must then be created locally (`useradd -m -G wheel NAME; passwd NAME`). Never put that password in `mkosi.conf` or a script argument. `/etc`, `/var`, and `/home` are expected to survive future OS image replacement.

The image includes the locally installed Ersa Plymouth theme and a small `foundation status` command. The latter reads actual release, mount, and service state; it provides no update or recovery commands yet. The theme source and redistribution limit are recorded in [docs/branding.md](docs/branding.md). The theme's appearance in the initrd has not been visually tested.

The home partition is last and may consume available space during image creation. It is not automatically expanded by `install-raw` when installed on a larger disk. Back up all three persistent partitions before changing their sizes or replacing a device. The current fstab uses labels, so do not attach two ECP disks with identical labels at once.

## Local VM test

After a successful build, locate the `.raw` file and boot a **copy** with UEFI firmware. On x86_64 use a v3 capable virtual CPU (`-cpu host` with KVM on a compatible host, or an emulated Haswell or newer CPU):

```sh
cp mkosi.output/ecp_0.1.0.raw /tmp/ecp-test.raw
qemu-system-x86_64 -machine q35,accel=kvm -cpu host -m 4096 \
  -drive if=pflash,format=raw,readonly=on,file=/usr/share/edk2/x64/OVMF_CODE.4m.fd \
  -drive file=/tmp/ecp-test.raw,format=raw,if=virtio -serial mon:stdio
```

OVMF paths vary by host. For AArch64 use `qemu-system-aarch64 -machine virt -cpu cortex-a72` with AArch64 EDK2 firmware; check firmware path and graphics support locally. Verify bootloader, root mounted read-only, `/etc`/`/var`/`/home` mounted writable, machine ID persistence across reboot, first-boot credential flow, NetworkManager, SDDM, and a Plasma Wayland session. These tests have **not** yet run.

## GitHub Actions build

`.github/workflows/build-x86_64.yml` builds the normal labeled x86-64 image on an Ubuntu 24.04 GitHub-hosted VM for pushes, pull requests, or manual dispatch. It runs the same `scripts/build x86-64` entry point with `SELinuxRelabel=yes`; it does not disable SELinux relabeling or produce an unlabeled VM image. A successful run uploads a zstd-compressed raw image, its SHA-256 checksum, and mkosi build metadata for 14 days. The first run will show whether the GitHub runner's filesystem and kernel permit `setfiles` to write labels; if that step fails, the workflow reports the limitation rather than bypassing it. This workflow currently covers x86-64 only.

## Installation

Anaconda's `liveimg` payload can install a tarred root filesystem, but it cannot consume this complete mkosi GPT raw image as its payload. Its normal UEFI installation flow would also install a different boot stack and would not preserve this image's systemd-boot entry and read-only partition flags. ECP therefore has no Anaconda ISO yet. An Anaconda integration must be tested with a custom storage and bootloader handoff that retains systemd-boot, the exact GPT layout, persistent boundaries, and first-boot identity flow. Calamares would need comparable work and has no present advantage. [RHEL 10 `liveimg` specification](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html-single/automatically_installing_rhel/automatically_installing_rhel), [mkosi disk output](https://github.com/systemd/mkosi/blob/v27.1/mkosi/resources/man/mkosi.1.md).

For a disposable VM or dedicated disk, boot a rescue/live environment, inspect the target with `lsblk`, and write the complete image:

```sh
sudo scripts/install-raw --erase mkosi.output/ecp_0.1.0.raw /dev/DISK
```

This **erases the entire device**. The script rejects mounted or undersized targets, but it cannot identify the device for you. It does not preserve existing data or partition around another OS. The first boot needs an interactive console to complete local configuration. The raw writer is untested on real hardware.

## Updates and recovery design

The release mechanism still to be implemented is **A/B root partitions plus matching boot artifacts**. Build and sign a complete new root image and matching kernel/initrd or UKI. Download and verify it, write only the inactive root slot, then install its boot entry with a limited boot count. Boot health must require successful local filesystem mounts, NetworkManager, SDDM, and a functioning graphical login before blessing the new entry. If the count expires, firmware/bootloader must return to the previous known-good entry. Keep `/etc`, `/var`, and `/home` outside both slots and migrate their formats compatibly; a root rollback cannot automatically undo changes to those partitions. A rescue image and backups remain necessary if both slots or shared state are damaged.

`systemd-sysupdate` supports whole-partition A/B transfers and `systemd-boot` supports boot counting. The raw image now selects Rocky's AppStream `systemd-boot-unsigned` package and requests unsigned UKIs. It still **does not enable automatic updates, boot blessing, or automatic rollback**. A second root slot, signed update artifacts, tested boot counting and health checks are release gates. Until then, recovery means restoring the whole raw image to a disposable test disk or reinstalling after backing up and restoring the three persistent partitions. [systemd-sysupdate](https://www.freedesktop.org/software/systemd/man/261/systemd-sysupdate.html), [systemd boot blessing](https://www.freedesktop.org/software/systemd/man/260/systemd-bless-boot.service.html), [Rocky AppStream package listing](https://dl.rockylinux.org/pub/rocky/10/AppStream/x86_64/os/Packages/s/).

Secure Boot is the next boot integrity step after a labeled VM boot. mkosi can sign systemd-boot and generated UKIs with `SecureBoot=yes`, `SecureBootKey=`, and `SecureBootCertificate=`; signing keys must be supplied from outside this repository. Test key enrollment in disposable OVMF variables before touching hardware firmware. Every updated bootloader and UKI will need signing, while root integrity still requires a separate verity or equivalent design. No Secure Boot keys are generated or enrolled by this repository yet. [mkosi Secure Boot options](https://github.com/systemd/mkosi/blob/v27.1/mkosi/resources/man/mkosi.1.md).

## Version control and validation limits

The mkosi frontend is pinned to 27.1, Rocky to 10.2, and EPEL/CRB are explicitly selected. RPMs and repository metadata are **not** pinned: Rocky's `releasever` normally resolves to major version 10, mirrors move, and EPEL 10.2 packages change. The generated JSON manifest and checksum identify one build's inputs and artifact; they do not make independent rebuilds bit identical. A production release needs immutable mirrored RPM metadata and package NEVRAs, pinned tools-tree content, signed artifact metadata, and tested rollback compatibility.

Validation performed here: `bash -n` passed for the shell scripts; mkosi 27.1 `summary` parsed both architecture configurations and the `unlabeled-vm` overrides. The user's x86-64 build completed its package transaction but failed at the explicit SELinux relabel step with `setfiles ... Operation not permitted`. The Podman builder has not run here: Podman cannot initialize its state under `/run/user` in this restricted workspace. No completed disk image has been verified. A labeled build, partition inspection, VM boot, installation on spare media, Plymouth visual check, and rollback fault injection are required before calling ECP bootable or deployable.

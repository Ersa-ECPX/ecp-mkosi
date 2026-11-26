# ECP — Ersa Compute Platform

An immutable, image-based GNOME desktop built with mkosi from openSUSE Tumbleweed (rolling). The root is a read-only,
dm-verity-protected image in one of two A/B slots; updates write the other slot and fall back if it does not boot. The design,
and what has and has not been tested, is in [docs/architecture.md](docs/architecture.md); every feature has a page in
[docs/README.md](docs/README.md). EAP (Ersa Automotive Platform) is outside this repository for now.

**Status:** experimental. On 2026-11-25 the 1.0.0 images passed these QEMU tests: install from the installer medium, dm-verity
(intact and tampered), Secure Boot enforcement, and an update into the spare slot with boot, blessing and rollback. Fallback after a
broken update: three tries, then the old system (passed 2026-11-26). Not tested: real hardware, an interrupted
update, signed archives on a signed boot. Not a released installer yet.

## Repositories

| Repository | Contents |
| --- | --- |
| `ecp-mkosi` (this one) | mkosi configuration (`mkosi.*`), `branding/`, `scripts/`, `docs/`, `AGENTS.md` |
| [`foundation`](https://github.com/Ersa-ECPX/foundation) | `foundation` CLI, `foundationd`, the graphical installer, the D-Bus contract, tests |
| [`packages`](https://github.com/Ersa-ECPX/packages) | RPM specs and small package sources, the RPM repository workflow |

`scripts/build` runs `scripts/prepare-build`, which clones what is missing: `foundation/`, `packages/` (git-ignored checkouts) and
the compiled packages' sources in `.sources/`. Existing checkouts are never touched (`ECP_UPDATE=1` fast-forwards them). The
packages repository's RPMs are not consumed by the image build: packages are compiled locally from the cloned sources.

## Build

Needs Linux with user namespaces, systemd 254 or newer, Python, `rpmbuild`, `rsvg-convert`, podman, QEMU/OVMF for tests, about
28 GiB of sparse disk for an image plus caches, and **mkosi 27.1** from its Git tag (`scripts/build` rejects other versions):
`pipx install 'git+https://github.com/systemd/mkosi.git@v27.1'`. The tools tree is Tumbleweed (`ToolsTreeDistribution=opensuse`);
a Fedora tools tree stalled on `perl` and rejected Tumbleweed signatures. Scratch space is `/tmp/ecp-work` (`ECP_WORK_DIR`, about
15 GiB). `scripts/build-podman` runs mkosi in a privileged Fedora container instead (use only with a trusted checkout).

```sh
scripts/build x86-64                              # the A/B disk, mkosi.output/ecp_<version>.raw
scripts/build x86-64 --profile installer          # the installer medium
scripts/build x86-64 --profile root --image-version 0.2.0    # the payload of an update
scripts/make-update-archive 0.2.0 [--sign]        # dist/update/ecp_0.2.0-x86_64.tar.zst
```

Signing: builds use the public test keys in `build/security/testkeys` unless `ECP_SECURITY_DIR` is set or `build/security/release` exists; a release
refuses the test keys ([build/security/README.md](build/security/README.md)). Updates are signed by default.

Targets: `x86-64` (UEFI x86_64) and `arm64` (ARMv8-A, not tested). The image uses SELinux (targeted policy, relabelled at build)
and the Tumbleweed OSS repository, so two builds on different days differ; the manifest and checksum identify one build but do
not make rebuilds identical. A release needs a pinned snapshot, which does not exist yet. Tumbleweed is not built for
x86-64-v3 (about 0.5% of packages have v3 builds, the kernel does not); any v3 work is a measured experiment.

## Layout

ESP 1 GiB, recovery 1 GiB (reserved), root slot a and b (8 GiB EROFS each, each with a 96 MiB dm-verity hash partition), and
`ECP-DATA` (btrfs, at least 10 GiB, `/var` and `/home` as subvolumes). `/etc` is an overlay on `/var`. Details: [architecture](docs/architecture.md),
[storage](docs/features/storage-and-persistence.md). Profiles: the normal layout is `mkosi.repart.system/`, the installer medium
(about 6 GiB, one verity root, a small data volume) `mkosi.repart.installer/`, the update payload `mkosi.repart.root/`.

## Releases

`scripts/release 0.2.0` builds the installer and the update and uploads them to a GitHub release (`--draft`, `--no-build`,
`--sign`). Images are zstd-compressed and split into `.partNN` files above GitHub's 2 GiB limit. `scripts/publish-feed 0.2.0`
publishes the update feed. A release refuses the development Secure Boot key. Releases are pre-releases; install an unsigned
update with `foundation update install FILE --unsigned` (developer mode must be on). See [updates](docs/features/updates.md), [hosting](docs/ota-hosting.md).

## Tests

```sh
ECP_MKOSI_DIR=$PWD python3 -m unittest discover -s foundation/tests     # about 600 tests; some need this repository
scripts/qemu-smoke [IMAGE]                                               # boots an image, checks it came up healthy
scripts/qemu-verity-test --image DISK                                    # verity root; a changed block refuses to boot
scripts/qemu-secureboot-test --image DISK --uki UKI                      # Secure Boot enforcement (throwaway key)
scripts/qemu-fallback-test --archive NEWER-UPDATE.tar.zst                # install, update, break the new slot, fall back (see its header)
scripts/qemu-try install|boot|reset                                  # try the system by hand in a QEMU window (blank disk, installer medium)
scripts/qemu-datadisk-test --disk DISK.raw                        # /var and /home come from the booted disk even next to another ECP disk
```

The QEMU tests need `qemu-system-x86_64`, OVMF and e2fsprogs; they use KVM when available. The disk tests take an installed
disk (not the installer medium). More: [building and testing](docs/features/building-and-testing.md), [testing an update](docs/ota-test.md).

## Installing

Use the graphical installer or `foundation deploy install DISK` from the installer medium ([installer](docs/features/installer-app.md),
[deploy](docs/features/deploy.md)); both erase the whole disk. `scripts/install-raw --erase IMAGE /dev/DISK` writes a complete
image to a dedicated disk and is untested on real hardware. Anaconda cannot consume this GPT image and would install a different
boot stack, so there is no Anaconda path.

## Foundation and packages

`foundation` is the command line of the platform and `foundationd` the D-Bus service behind it (Python, GLib/GDBus, polkit); see
[foundation/README.md](https://github.com/Ersa-ECPX/foundation/blob/main/README.md) and [docs/foundationd.md](docs/foundationd.md).
The package list is `mkosi.conf` and `mkosi.conf.d/`. Applications are Flatpaks (`foundation apps install`); other RPMs are system
extensions (`foundation compat install`). `ersa-bluetooth` and `ersa-dialer` (calls from the desktop through a paired phone) are
built in a Tumbleweed container by `scripts/build-rpms`; no pairing or call flow has been tested against a real phone and
`ersa-bluetooth` has no license file yet.

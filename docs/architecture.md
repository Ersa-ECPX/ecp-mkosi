# Ersa Compute Platform: architecture

An immutable, image-based GNOME desktop built with mkosi from openSUSE Tumbleweed. The operating system is a read-only
image in one of two root slots; updates write the other slot and fall back by themselves if it does not boot. One
service, `foundationd`, is the only supported way to change the system; `foundation` and the installer are its clients.

## Layers

| Layer | What |
| --- | --- |
| Applications | Flatpak (system-wide), compat extensions (`foundation compat`), toolbox/podman for what compat cannot do |
| Desktop | GNOME Shell, GDM, GNOME Initial Setup |
| Management | `foundationd` (root, polkit, jobs in worker processes) and its clients |
| System | systemd, dbus-broker, NetworkManager, PipeWire, SELinux (targeted) |
| Image | read-only EROFS root with dm-verity, built from Tumbleweed RPMs |
| Boot | UEFI, systemd-boot, signed unified kernel image (UKI), initrd |

## Boot

1. UEFI starts systemd-boot from the ESP. Secure Boot checks it, then every UKI.
2. systemd-boot lists `EFI/Linux/ecp_<version>+<tries>.efi`, boots the default or a one-shot choice, and counts tries down.
3. The UKI carries kernel, initrd and one fixed command line. `roothash=` names the root partition; nothing is edited per
   machine, so the signature stays valid.
4. The initrd opens the root as dm-verity, mounts `/var`, builds the `/etc` overlay, switches root.
5. `systemd-bless-boot` marks the entry good once the graphical target is up with no failed unit. An entry whose tries run
   out is skipped and boot falls back to the previous one.

## Storage

| Partition | Size | Holds |
| --- | --- | --- |
| ESP | 1 GiB | systemd-boot, one UKI per slot |
| Recovery | 1 GiB | reserved, empty |
| Root slot a, hash a | 8 GiB, 96 MiB | EROFS image and its dm-verity hash tree |
| Root slot b, hash b | 8 GiB, 96 MiB | the other version |
| `ECP-DATA` | rest, at least 10 GiB | btrfs: `/var` and `/home` as subvolumes |

`/etc` is an overlay: the image is the lower layer, local changes are the upper layer on `/var`. A slot's partitions are
named by the root hash (their GPT UUIDs); the slot labels are fixed (`ecp-root-a`, `ecp-root-b`).

## Updates

`foundation update install FILE-or-URL` streams one archive (manifest, signature, UKI, root image, hash tree) into the
spare slot, checks every entry, verifies what it wrote, then installs the UKI last. The running slot is never touched.
Older versions and versions already installed are refused unless `--force`. `foundation slot boot` selects a slot (once
or for good). An update feed (`ota.url`) and a timer check for new versions.

## Management

`foundationd` exposes `org.ersa.Foundation1` on D-Bus: it validates a request, asks polkit, takes a lock and runs the work
in a worker process that reports as a job. Interfaces: Manager, Config, Boot, Apps, Compat, Extensions, Updates, Job. The
contract is `foundation/data/dbus/org.ersa.Foundation1.xml`; design rule: prefer D-Bus, a library or reading files over
running a command.

## Software model

| Kind | Where | Changed by |
| --- | --- | --- |
| Base system | read-only image | replacing the image |
| Applications | Flatpak in `/var/lib/flatpak` | `foundation apps` |
| Other RPMs | system extensions in `/var/lib/extensions` | `foundation compat` (no scriptlets, checked against the base) |
| Dev environments | toolbox, podman | the user |
| Local configuration | `/etc` overlay and `/var` | the user, `foundation config` |

## Security

SELinux enforcing; polkit on every change; archives are streamed and checked by name, type and size; update manifests are
GPG-signed; kernel images are Secure Boot signed (mkosi); the root is dm-verity protected. No secret or machine identity is
in the image: the machine ID and SSH host keys are made on first boot. Not built: TPM measurement, disk encryption.

## What is verified

QEMU, 2026-11-25, 1.0.0 images, single machine type, no real hardware:

| Area | Result |
| --- | --- |
| Install from the installer medium onto a blank disk | passed |
| dm-verity: boot, and refusal of a changed root block | passed (`scripts/qemu-verity-test`, 9/9) |
| Secure Boot: enrol, boot; unsigned, wrong-key, tampered images and loader refused | passed (`scripts/qemu-secureboot-test`, 8/8; throwaway key, so mkosi's own signature is not what was tested) |
| Update into the spare slot, boot, bless, manual rollback, same version refused | passed |
| Older version refused | unit-tested |
| Broken new slot falls back after three tries | passed (`scripts/qemu-fallback-test`, 2026-11-26): the update boots the new entry once, boot counting does the rest; a hung initrd still needs a power cycle |
| Interrupted install, signed archive on a real boot, GitHub feed, update timer | not tested here |
| Compat bundle merged under enforcing SELinux on a booted image | not tested |
| Real hardware, TPM, encryption, recovery system, aarch64 | not built or not tested |

## Decisions

| Date | Decision | Why |
| --- | --- | --- |
| 2026-11-03 | GNOME replaces KDE | the KDE image did not reach a login |
| 2026-11-04 | openSUSE Tumbleweed base (was Rocky 10) | rolling base, current GNOME and kernel |
| 2026-11-05 | Two fixed A/B root slots, EROFS | robust: no custom initrd code to find the root |
| 2026-11-05 | `/etc` overlay; one btrfs volume for `/var` and `/home` | updates bring new defaults; shared space, separate lifecycles |
| 2026-11-05 | `foundationd` is the single management plane | one authorized, audited path |
| 2026-11-13 | dm-verity and Secure Boot are the only layout; constant command line | signed images cannot be edited per machine |

Next: signed images by update on a signed boot, real hardware, encryption, compression and
deltas once images are measured. Design notes: [foundationd](foundationd.md), [provisioning](provisioning.md),
[installer](installer.md).

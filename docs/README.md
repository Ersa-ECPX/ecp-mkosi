# Ersa Compute Platform documentation

The Ersa Compute Platform is an immutable, image-based desktop operating system built on openSUSE Tumbleweed
with GNOME. The root is a read-only image, updates replace whole images (A/B slots with automatic fallback), and
one management service, `foundationd`, is the only thing that changes the system. `foundation` (the command
line) and the installer are clients of it.

## Start here

| If you want to... | Read |
| --- | --- |
| Understand how the system is put together | [architecture.md](architecture.md) |
| Install it on a disk | [features/installer-app.md](features/installer-app.md), [features/deploy.md](features/deploy.md) |
| Add software (codecs, packages from openSUSE or Packman) | [features/repositories-and-bundles.md](features/repositories-and-bundles.md) |
| Install a Flatpak application | [features/applications.md](features/applications.md) |
| Manage the system from a terminal | [features/foundation-cli.md](features/foundation-cli.md) |
| Build an image or run the tests | [features/building-and-testing.md](features/building-and-testing.md) |
| Change the code | [../foundation/README.md](../foundation/README.md) (developer guide) |

## Features

Every feature has one page: what it is, how to use it, how it works, and what has and has not been tested.

| Feature | What it does | Status |
| --- | --- | --- |
| [Boot, slots and boot assessment](features/boot-and-slots.md) | A/B root slots, boot counting, automatic fallback, `foundation slot` | Built; tested in QEMU (UEFI) |
| [Storage and persistence](features/storage-and-persistence.md) | Read-only root, writable `/etc` overlay, one data volume, mounts that name their own disk | Built; tested in QEMU |
| [`foundation` command line](features/foundation-cli.md) | One consistent command for the whole system, JSON output, jobs | Built; unit and end-to-end tested |
| [Management service](features/management-service.md) | `foundationd`: D-Bus API, polkit on every change, jobs | Built; tested |
| [Graphical installer](features/installer-app.md) | macOS-style installer: language, time zone, disk, install with progress | Built; tested on a virtual display and booted in QEMU |
| [`foundation deploy`](features/deploy.md) | Erase a disk and install the running system on it, safely | Built; tested on files and in QEMU; not on real disks |
| [Installer medium](features/live-medium.md) | Live USB image that runs the installer | Built; booted in QEMU and installed a disk (a read-only variant was tried and removed) |
| [Repositories and bundles](features/repositories-and-bundles.md) | Signed openSUSE/Packman packages as one system extension per bundle | Built; verified against the live repositories in a container; not yet on a booted image |
| [System extensions](features/system-extensions.md) | `foundation ext`, RPM files as extensions, rechecking after an OS update | Built; tested |
| [Applications](features/applications.md) | System-wide Flatpaks | Built; tested against Flathub |
| [Configuration and provisioning](features/configuration.md) | Settings with a schema, first-boot "provisioned" state | Built; tested |
| [Multimedia and fonts](features/multimedia-and-fonts.md) | Codecs, hardware video decode, fonts for every script of the language list | Built; `@codecs` verified in a container |
| [Branding](features/branding.md) | Logo, boot splash, login screen, wallpapers, installer identity | Partly built; not all of it seen on a running desktop |
| [SELinux](features/selinux.md) | How labels are kept right for overlays, extensions and installed systems | Built; fixes found by booting |
| [Updates (OTA)](features/updates.md) | Signed image updates through `systemd-sysupdate` | **Scaffolding only**: not run end to end, no `foundation update` yet |
| [Building and testing](features/building-and-testing.md) | Build order, scripts, QEMU and container test methods | Built |

**Status words.** *Tested* means a test or a run by hand proved it, and the page says which. *Built* without a
test is stated as such. Nothing here is described as working because it was designed to.

## Known gaps (the honest list)

- **Updates are not usable yet.** The transfer definitions and test scripts exist, but no update has been
  applied end to end, there is no update server, no image signing, and no `foundation update`.
- **No real hardware, Secure Boot or USB stick has been tested.** Everything was run in QEMU or on image files.
- **No recovery system.** The recovery partition is reserved and empty.
- **Per-install kernel command lines are lost on an OTA update** (the new kernel image names the data volume by
  label again); see [storage](features/storage-and-persistence.md#known-gap).
- **Encrypted data volumes and other data filesystems** (ext4, xfs) are not built.
- **A bundle merged on a booted image under enforcing SELinux** has not been run yet.

## Other documents

- [architecture.md](architecture.md): the design and the decisions behind it.
- [foundationd.md](foundationd.md): the service's design and open questions.
- [installer.md](installer.md): the installer design notes and history.
- [provisioning.md](provisioning.md): first boot and device management options.
- [branding.md](branding.md): logo sources and where each appears.
- [ota-hosting.md](ota-hosting.md), [ota-test.md](ota-test.md): hosting updates, and the plan for the first local update test.

# Ersa Compute Platform documentation

The Ersa Compute Platform is an immutable, image-based desktop operating system built on openSUSE Tumbleweed
with GNOME. The root is a read-only image, updates replace whole images (A/B slots with automatic fallback), and
one management service, `foundationd`, is the only thing that changes the system. `foundation` (the command
line) and the installer are clients of it.

**As a wiki.** `scripts/build-wiki` turns all of these pages into one self-contained HTML file with a sidebar, search
and light and dark themes (`docs/wiki/index.html`; open it in a browser, no server needed).

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

## Browse by topic

Every feature has one page: what it is, how to use it, how it works, and what has and has not been tested. The grouping
below is the wiki's sidebar (`docs/wiki.json`).

### Architecture: how the system is put together

| Page | What it covers | Status |
| --- | --- | --- |
| [Architecture](architecture.md) | the layers, the boot chain, the decisions behind them | design |
| [Boot, slots and boot assessment](features/boot-and-slots.md) | A/B root slots, boot counting, automatic fallback, `foundation slot` | Built; tested in QEMU (UEFI) |
| [Storage and persistence](features/storage-and-persistence.md) | read-only root, writable `/etc` overlay, one data volume, mounts that name their own disk | Built; tested in QEMU |
| [SELinux](features/selinux.md) | how labels are kept right for overlays, extensions and installed systems | Built; fixes found by booting |
| [Secure Boot](features/secure-boot.md) | signed boot loader and kernel images, tested in QEMU with enforcement | Written; mkosi signing not yet run on the full image |
| [dm-verity](features/dm-verity.md) | the root filesystem checked block by block, slots named by the root hash | Written; never booted |
| [Updates (OTA)](features/updates.md) | signed image updates written by `foundation update install` | Written and tested on fake disks; not yet run on a booted system |
| [Testing an update locally](ota-test.md), [Hosting updates](ota-hosting.md) | the plan for the first real update test, and where to host | plan |

### Foundation: the management plane and its features

| Page | What it covers | Status |
| --- | --- | --- |
| [Management service](features/management-service.md) | `foundationd`: D-Bus API, polkit on every change, jobs | Built; tested |
| [The `foundation` command line](features/foundation-cli.md) | one consistent command for the whole system, JSON output, jobs | Built; unit and end-to-end tested |
| [Configuration and provisioning](features/configuration.md) | settings with a schema, the first-boot "provisioned" state | Built; tested |
| [`foundation deploy`](features/deploy.md) | erase a disk and install the running system on it, safely | Built; tested on files and in QEMU; not on real disks |
| [System extensions](features/system-extensions.md) | `foundation ext`, RPM files as extensions, rechecking after an OS update | Built; tested |
| [Remote access](features/remote-access.md) | the SSH server on or off with safe defaults, and the keys that may log in | Built; key login tried in a VM |
| [foundationd design](foundationd.md), [Provisioning](provisioning.md) | the service's design and open questions; first boot and device management | design |

### App support: getting software onto an immutable system

| Page | What it covers | Status |
| --- | --- | --- |
| [Applications](features/applications.md) | system-wide Flatpaks | Built; tested against Flathub |
| [Repositories and bundles](features/repositories-and-bundles.md) | signed openSUSE and Packman packages, one system extension per bundle | Built; verified against the live repositories in a container; not yet on a booted image |
| [Multimedia and fonts](features/multimedia-and-fonts.md) | codecs, hardware video decode, fonts for every script of the language list | Built; `@codecs` verified in a container |

### Installer: putting the system on a disk

| Page | What it covers | Status |
| --- | --- | --- |
| [Graphical installer](features/installer-app.md) | macOS-style installer: language, time zone, disk, install with progress | Built; tested on a virtual display and booted in QEMU |
| [Installer medium](features/live-medium.md) | the live USB image that runs the installer | Built; booted in QEMU and installed a disk (a read-only variant was tried and removed) |
| [Installer design notes](installer.md) | the design and its history | design |

### Desktop and branding

| Page | What it covers | Status |
| --- | --- | --- |
| [Branding](features/branding.md) | logo, boot splash, login screen, wallpapers, installer identity | Partly built; not all of it seen on a running desktop |
| [Branding sources](branding.md) | logo sources and where each appears | reference |

### Build and test

| Page | What it covers | Status |
| --- | --- | --- |
| [Building and testing](features/building-and-testing.md) | build order, scripts, QEMU and container test methods, the wiki | Built |

### Developer guide

| Page | What it covers |
| --- | --- |
| [Developer guide](../foundation/README.md) | the request path, how to add a feature, the API, verification status, troubleshooting |

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

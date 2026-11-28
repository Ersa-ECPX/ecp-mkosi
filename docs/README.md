# Ersa Compute Platform documentation

An immutable, image-based desktop built on openSUSE Tumbleweed with GNOME. The root is a read-only, dm-verity-protected image
in one of two A/B slots; updates write the other slot and fall back if it does not boot. One service, `foundationd`, changes
the system; `foundation` (command line) and the installer are its clients. `scripts/build-wiki` turns these pages into one
HTML file (`docs/wiki/index.html`).

Pages say what was tested and how; all runs so far are in QEMU, none on real hardware. Nothing is called working because it was
designed to.

| I want to... | Read |
| --- | --- |
| Understand the design and what is verified | [architecture.md](architecture.md) |
| Install it | [installer-app](features/installer-app.md), [deploy](features/deploy.md) |
| Manage it in a window | [deck](features/deck.md) |
| Add software | [applications](features/applications.md), [repositories and bundles](features/repositories-and-bundles.md) |
| Use the command line | [foundation-cli](features/foundation-cli.md) |
| Build an image, run tests | [building-and-testing](features/building-and-testing.md), [ota-test](ota-test.md) |
| Know what each script does, or serve an update | [scripts](scripts.md) |
| Change the code | [../foundation/README.md](../foundation/README.md) |

## Pages

**Architecture:** [boot and slots](features/boot-and-slots.md) · [storage](features/storage-and-persistence.md) ·
[SELinux](features/selinux.md) · [encryption](features/encryption.md) · [updates](features/updates.md) · [hosting updates](ota-hosting.md) ·
[Secure Boot](features/secure-boot.md) · [dm-verity](features/dm-verity.md)

**Foundation:** [management service](features/management-service.md) · [command line](features/foundation-cli.md) ·
[configuration](features/configuration.md) · [deploy](features/deploy.md) · [system extensions](features/system-extensions.md) ·
[remote access](features/remote-access.md) · designs: [foundationd](foundationd.md), [provisioning](provisioning.md), [devices and diagnostics](devices-and-telemetry.md)

**Identity:** [sign in with Ersa ID](features/identity.md)

**Software:** [applications](features/applications.md) · [repositories and bundles](features/repositories-and-bundles.md) ·
[multimedia and fonts](features/multimedia-and-fonts.md)

**Installer:** [graphical installer](features/installer-app.md) · [installer medium](features/live-medium.md) ·
[design notes](installer.md) · [translations](features/translations.md)

**Desktop:** [branding](features/branding.md) · [branding sources](branding.md)

**Build:** [building and testing](features/building-and-testing.md)

## Known gaps

- Real hardware, TPM measurement, disk encryption and a recovery system are not built or not tested.
- A hung initrd does not reboot by itself; interrupted installs and signed archives on a signed boot are untested: see [updates](features/updates.md).
- A bundle merged on a booted image under enforcing SELinux has not been run.
- The data volume is found by the disk the system started from (two ECP disks no longer compete); written and unit-tested, not yet booted ([storage](features/storage-and-persistence.md)).

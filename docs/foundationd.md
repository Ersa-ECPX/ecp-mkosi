# foundationd design

This page keeps the design points; the user-facing description is [management service](features/management-service.md).

`foundationd` is the one service through which the Ersa Compute Platform is inspected and changed. The `foundation` command
line, Deck, the installer and any future client call its D-Bus API (`org.ersa.Foundation1`) and implement no policy of their
own. The contract is `foundation/data/dbus/org.ersa.Foundation1.xml`; the developer guide is `foundation/README.md`.

## Principles

- **Delegate.** Use systemd, libflatpak, logind and libsolv for the work. `foundationd` authorizes, sequences and reports; it
  does not reimplement package management, update transfer, boot management or hardware control.
- **One API, many clients.** Everything the CLI does is available over D-Bus with the same authorization.
- **Honest state.** A feature that is not implemented is not in `--help` or in the API.
- **Previewable and confirmed.** Substantial changes have a dry run; destructive storage operations need the target typed back.
- **Separate lifecycles.** Image updates, application updates and persistent configuration are different things.

## Process model

```
 foundation   Deck   installer   (any client)
        \       |        /
         +------+--------+   system bus: org.ersa.Foundation1
                 v
   foundationd  (root, D-Bus activated, exits when idle)
     authorizes with polkit, owns jobs, locks, config, state
                 | spawns one worker per job
                 v
   foundationd-worker <kind> ->  bootctl, sfdisk, systemd-repart, systemd-sysext,
                                  libflatpak, logind, rpm tools
```

The service is a Python (PyGObject/GDBus) broker that never blocks its main loop. Long or risky work runs in a worker
process, so a crash cannot take the broker down. Workers inherit the service's unit settings: the unit uses **no
mount-namespace sandboxing**, because an overlay made for `systemd-sysext` in a private namespace would be invisible to the
system. Per-worker transient units are a later option.

Where an API exists it is used: logind (reboot), libflatpak (applications), libsolv (package solving), direct reads of the
partition table, EFI variables, mountinfo and sysfs. Commands remain for `bootctl`, `sfdisk`, `systemd-repart`,
`systemd-sysext`/`systemd-confext`, `rpm`/`rpm2cpio`/`rpmkeys`, `gpg` and `flatpak search`.

## The API

Objects: `/org/ersa/Foundation1` (manager and domain interfaces) and `/org/ersa/Foundation1/Job/<id>`. Changes within
`Foundation1` are additive; a breaking change would be `Foundation2`. Only implemented interfaces are in the XML.

| Interface | Purpose | Polkit action |
|---|---|---|
| `Manager` | status, system facts, base packages, job list, provisioning | none / `provision` |
| `Config` | schema-checked settings | `config` |
| `Job` | state, steps, progress, logs, cancel | `jobs` (cancel) |
| `Boot` | slots, default and next boot, splash, reboot, reboot to firmware | `boot`, `slots`, `splash` |
| `Deploy` | list disks, plan, install, preseeded install | `deploy` |
| `Apps` | Flatpak apps and application sources (remotes) | `apps`, `apps-sources` |
| `Compat` | RPM files and repository bundles as extensions | `compat` |
| `Repos` | list, search, info, enable, add, remove, refresh | `repos` |
| `Updates` | install, check, install latest, restart, pending | `updates` |
| `Extensions` | sysext and confext merge state | `extensions` |
| `Remote` | SSH server and keys | `remote` |

Reads need no authorization. A call that changes the system returns a Job object path after authorization succeeds. Secrets
never travel as D-Bus strings: files and passwords cross as file descriptors or hashes. Errors live under
`org.ersa.Foundation1.Error.*` (`NotAuthorized`, `InvalidArgument`, `Busy`, `NotFound`, `NotCancellable`, `Unavailable`,
`AlreadyProvisioned`, `Failed`, ...).

## Jobs, locks and failure

- A job has steps, a progress fraction (0.0 to 1.0), a log and a result. Cancelling is clean up to a point of no return; after
  it the job reports that it cannot be cancelled.
- Jobs take named locks (`extensions`, `repos`, `flatpak`, a disk, the slots). A conflicting job fails at once with `Busy`.
- Jobs outlive the client. Their history is kept under `/var/lib/foundation/jobs/`.
- Jobs are not resumable, except where the delegate resumes (partial downloads).

## Updates and slots

`foundationd` writes updates itself (`backends/updates.py`); `systemd-sysupdate` was dropped because slots are named by their
dm-verity root hash, not by label. An update is written only to the spare slot with its kernel image, named for boot counting.
The new entry is booted once (one-shot) and is made the default only after a good boot; failed boots fall back to the old slot.
The same root image cannot go into two slots, because the hash names the partitions. Unsigned installs
(`Updates.InstallLatestUnsigned`, `foundation update install --unsigned`) are refused unless `system.developer_mode` is on. See
[updates](features/updates.md) and [boot and slots](features/boot-and-slots.md).

## Configuration and state

| | Configuration | State |
|---|---|---|
| What | declared intent (update source, channel, behaviour) | what the machine did (provisioned, when) |
| Written by | image defaults, administrator, `Config.Set` | `foundationd` only |
| Format and place | `key=value`; `/usr/lib/foundation/defaults.d/`, `/etc/foundation/foundation.conf.d/` | JSON; `/var/lib/foundation/state.json` |

Later files win. `Config.Set` writes one generated drop-in, `90-api.conf`; a preseed writes `50-preseed.conf`, which sorts
earlier, so a later `Config.Set` still wins. Every key is declared in a schema (`foundation/data/config/schema.d/`) with a type,
default and description, so unknown keys and bad values are rejected. Keys today: `ota.url`, `ota.channel`, `ota.auto_check`,
`ota.auto_update`, `system.developer_mode`, `apps.default_apps` (installed once by `foundation-default-apps.service` when the
machine is first online). Secrets are not configuration; they belong in systemd credentials or a root-only file. See
[configuration](features/configuration.md).

## Provisioning

`system.provisioned` is a one-way latch with the marker `/var/lib/foundation/provisioned` for units. It is set by
`Manager.CheckProvisioning` (evidence: an administrator's `gnome-initial-setup-done` file) or by the explicit
`Manager.CompleteProvisioning`. Details and open questions: [provisioning](provisioning.md).

## Not built

- `Diagnostics` (`Doctor`, `CollectBundle`) and `RebootToRecovery` (no recovery entry exists).
- Capability gates beyond developer mode, per-worker sandboxes, factory reset, firmware updates (fwupd), disk encryption, performance
  and power profiles. Non-goals: managing the base RPM set on an immutable system, network configuration, user accounts.

## Testing

Unit tests run the broker on a private bus with fake polkit, logind and bootctl and scripted workers; the CLI and Deck tests
drive a real `foundationd` that way. Real disk, update and boot behaviour is tested only in QEMU on virtual disks; nothing
automated writes to a real disk. Not yet verified on a booted image: SELinux enforcement of the service, `systemd-sysext`
merging of compat extensions end to end, Flatpak installs against the real Flathub, and the polkit prompt in a GNOME session.

## Tested

The service, its jobs and most interfaces exist and are covered by unit tests on a private D-Bus session (fake polkit, logind and boot tools); parts ran in QEMU, none on hardware. `Diagnostics` and a recovery entry are not built.

# foundationd architecture

Status: design proposal with a first implementation (see Implementation status). Claims about other
software are marked **verified** (checked against actual packages in this project) or **unverified**.

`foundationd` is the core of the Ersa Compute Platform: the one service through which the OS is
inspected and changed. `foundation` (the CLI), the graphical installer, the recovery environment and
a future settings application are all clients of its D-Bus API. No client implements OS policy, and
no client shells out to another client.

## Implementation status

The service and the command line are implemented for the interfaces below. The developer guide
(architecture, request lifecycle, how to add a feature, tests) is `foundation/README.md`; the contract
is `foundation/data/dbus/org.ersa.Foundation1.xml`. This file keeps the *design* and the open
questions.

| Interface | State |
| --- | --- |
| `Manager` | implemented: status, system facts, base packages, job list |
| `Config` | implemented: schema, layered files, `Set`/`Reset` with polkit. The `ota.*` keys are stored but nothing reads them yet |
| `Job` objects, locks, workers, queries | implemented: steps, progress, cancellation up to a point of no return, persisted history, restart recovery |
| `Boot` | implemented: state, slots, set default / next boot, release a slot, reboot, reboot to firmware. **No recovery entry exists yet**, so `RebootToRecovery` is not offered |
| `Apps` | implemented: libflatpak for list/install/remove/update/remote; `flatpak` CLI for search and info |
| `Compat` | implemented: inspect, install RPM files, remove, list, revalidate; **bundles** (many repository packages as one extension: install, remove, update, catalog) |
| `Repos` | implemented: list, search, info, enable, refresh; signed metadata, libsolv solving, verified downloads |
| `Extensions` | implemented: list, merge, unmerge, refresh for sysext and confext |
| `Remote` | implemented: SSH on/off with safe defaults, authorized keys; **not yet run on a booted image** |
| `Deploy` | implemented: list disks, plan, install to a disk (used by `foundation deploy` and the graphical installer) |
| `Updates` (OTA), `Diagnostics`, `CompleteProvisioning` front ends | **not implemented** |

Changes from the first design, and why:

- The Python broker with worker processes was built as designed. Workers inherit the service's
  systemd settings instead of running in their own sandboxes, so the unit uses **no mount-namespace
  sandboxing** (an overlay mounted for `systemd-sysext` inside a private namespace would be invisible
  to the system). Per-worker transient units are still the plan.
- Where D-Bus or a library exists it is used (logind for rebooting, libflatpak for applications,
  direct reads for the partition table, EFI variables, mountinfo and sysfs). Commands remain for
  `bootctl`, `sfdisk`, `systemd-sysext`/`systemd-confext`, `rpm`/`rpm2cpio`, and `flatpak search`.
- Files cross the API as descriptors and are staged into a private directory; the RPM payload is
  read by our own strict unpacker, and the RPM's magic bytes are checked before any tool sees it
  (`rpm -qp` treats a text file as a manifest of paths to open).
- A slot copy gets its own label (`ecp-root_<name>-copy`) and its own kernel image with an edited
  `root=PARTLABEL=`, because two partitions with one label are ambiguous and block updates.

Verification results are dated in `foundation/README.md#verification-status`.

## Principles

- **Delegate.** Use systemd (`sysupdate`, `sysext`, `repart`, `bootctl`, logind), Flatpak and fwupd
  for the actual work. `foundationd` orchestrates, authorizes, sequences and reports. It does not
  reimplement package management, update transfer, boot management or hardware control.
- **One API, many clients.** Everything the CLI can do is available over D-Bus, with the same
  authorization.
- **Honest state.** Report what is true. A feature that is not implemented does not appear in
  `--help` or in the API.
- **Reversible and previewable.** Substantial changes have a plan/dry-run form. Destructive storage
  operations require explicit confirmation (see Safety).
- **Image updates are separate from app updates and from persistent machine configuration.**

## Process model

```
 foundation (CLI)  installer app  recovery UI  settings applet
        \              |              |              /
         +-------------+--------------+-------------+
                              | system bus: org.ersa.Foundation1
                              v
                 foundationd  (root, D-Bus activated, idle exit)
                  - authorizes with polkit
                  - owns the job table, locks, config and state
                  - never blocks its main loop
                              | spawns, one per job
                              v
                 foundation-worker <kind>   (short-lived, sandboxed per kind)
                              | delegates to
                              v
        systemd-sysupdated / systemd-sysext / systemd-repart / bootctl / logind
        Flatpak (libflatpak) / fwupd / rpm tools
```

- `foundationd` is a broker: it validates requests, takes locks, starts a worker, and relays the
  worker's progress as D-Bus signals. Long or risky work runs in a separate worker process so a
  crash or hang cannot take the broker down and each worker can have its own systemd sandbox.
- Started on demand by D-Bus activation; exits after a period with no jobs and no clients.
- Implementation language is Python, via GLib/GDBus (PyGObject). **Unverified:** the PyGObject
  package name and its presence in the base image; the recovery environment must carry the same
  runtime.
- The existing pure-standard-library `foundation/` package (Flatpak and system backends, tests with
  fake runners) becomes the backend library used by workers.

## D-Bus API (`org.ersa.Foundation1`)

Names and signatures are provisional until the introspection XML is written. The XML, shipped in
`/usr/share/dbus-1/interfaces/`, is the contract clients are built and mocked against. Additive
changes only within `Foundation1`; breaking changes get `Foundation2`.

Objects: `/org/ersa/Foundation1` (manager and domain interfaces) and
`/org/ersa/Foundation1/Job/<id>`.

| Interface | Purpose | Main members |
|---|---|---|
| `Manager` | Identity and status | `Version`, `GetStatus()`, `ListJobs()` |
| `Config` | Settings and provisioning state | `Get`, `Set`, `Reset`, `List`, `Changed` signal |
| `Deploy` | Write the OS to a disk, create installer media | `ListDisks()`, `PlanDeploy()`, `Deploy()` |
| `Apps` | Flatpak applications | `ListInstalled()`, `Search()`, `Install()`, `Remove()`, `Update()` |
| `Compat` | Legacy RPMs as sysexts | `Inspect()`, `Install()`, `Remove()`, `List()`, `Revalidate()` |
| `Extensions` | sysext and confext management | `List()`, `Merge()`, `Unmerge()`, `Refresh()` |
| `Remote` | SSH server and keys | `GetStatus()`, `ListKeys()`, `SetSsh()`, `AddKey()`, `RemoveKey()` |
| `Updates` | OTA images and slots | `Check()`, `Download()`, `Apply()`, `GetSlots()`, `Rollback()` |
| `Boot` | Boot slots and boot control | `GetBootState()`, `GetSlots()`, `SetDefaultSlot()`, `SetNextBoot()`, `CopySlot()`, `ReleaseSlot()`, `Reboot()`, `RebootToFirmware()` (`RebootToRecovery()` once a recovery entry exists) |
| `Diagnostics` | Health checks and support data | `Doctor()`, `CollectBundle()` |

Long operations return a Job object path. Job members:

- Properties: `Kind`, `State` (`queued`, `running`, `done`, `failed`, `cancelled`), `Step`,
  `Progress` (0.0 to 1.0), `Result` (`a{sv}`), `Error`.
- Signals: `StepChanged(id, title)`, `ProgressChanged(fraction)`, `Finished(ok, message)`.
- Methods: `Cancel()`, `Logs(n)`.

Errors use the namespace `org.ersa.Foundation1.Error.*` (for example `AlreadyProvisioned`,
`NotAuthorized`, `Busy`, `InvalidArgument`, `DiskInUse`, `NeedsReboot`).

### Authorization (polkit)

Every state-changing method checks a polkit action against the calling bus name:

- `org.ersa.foundation.inspect`: read-only status and listing. No authentication.
- `org.ersa.foundation.apps`: install or remove applications.
- `org.ersa.foundation.compat`, `.extensions`: manage sysexts. Admin authentication.
- `org.ersa.foundation.update`: check, download, apply, roll back. Admin authentication.
- `org.ersa.foundation.deploy`: write a disk. Admin authentication, always, even on a live medium
  unless the live-session rule (shipped only in the installer sysext) allows it.
- `org.ersa.foundation.config`: change settings. Admin authentication.

Secrets (passwords, tokens) are never D-Bus string arguments. They travel as file descriptors
(type `h`) or as hashes, so they do not appear in logs, `busctl monitor` output or properties.

## Jobs, locking and failure

- Each job holds named locks. Conflicting jobs fail fast with `Busy` rather than queueing silently.

  | Lock | Held by |
  |---|---|
  | `slots` | Updates apply, rollback, deploy to the running disk (refused anyway) |
  | `extensions` | Compat install/remove, Extensions merge/unmerge, Updates apply |
  | `disk:<dev>` | Deploy to that disk |
  | `flatpak` | Apps install/remove/update |

- Jobs outlive the client that started them. A short job record (kind, state, result, error) is kept
  under `/var/lib/foundation/jobs/` for the last N jobs so a client can report an outcome after a
  restart.
- Jobs are not resumable, except where the delegate resumes (for example partial downloads).
- Irreversible steps write a write-ahead record under `/var/lib/foundation/transactions/` before they
  run. On startup `foundationd` inspects unfinished records and either completes a safe recovery or
  reports the inconsistency; it never guesses.
- `Cancel()` is honoured at safe points. After an update's commit step the job reports that it can
  no longer be cancelled.

## Domain designs

### Deploy and installer media

Covered in `installer.md` and now a section of this API. Key points: `PlanDeploy` is a pure dry run;
`Deploy` refuses any disk that holds the running root or ESP or has anything mounted; it uses
`systemd-repart` with `mkosi.repart/`, writes root slot A and the ESP, seeds `/etc`, applies an
optional `--user`, and reports every step as job signals. Creating installer media is a separate
operation on the same interface; its source is an installer image, because installed systems do not
carry the installer pieces.

### Applications (Flatpak)

- System-wide installation only in the first version. Flatpak's data lives under
  `/var/lib/flatpak`, which is on the writable `/var` partition.
- Use libflatpak through GObject introspection for structured progress, not by parsing CLI output.
  **Verified:** `flatpak` 1.18.4 and `typelib-1_0-Flatpak-1_0` exist in Tumbleweed. The CLI backend
  already in `foundation/flatpak.py` was verified against Flatpak 1.16 (tab-separated output,
  `pkg setup`, `pkg info`) and remains the fallback.
- First boot configures the Flathub remote (`pkg setup`).
- GNOME Software or Bazaar may install apps directly through Flatpak's own authorization; that does
  not conflict with this API, which exists for the CLI, the recovery environment and settings.
- **Open:** per-user installations. They need no privilege and are out of scope for the first
  version.

### Compat RPMs as sysexts

Purpose: run a legacy or third-party RPM without a writable `/usr`.

1. `Inspect(rpm)`: read metadata, list files, check the `Requires` against the base image's rpm
   database and check each ELF's `NEEDED` sonames against the base libraries. Report what is
   missing, and refuse or warn.
2. `Install(rpm)`: extract the payload (no scriptlets are run) into a staging tree, generate
   `usr/lib/extension-release.d/extension-release.<name>` matching the base `ID` and `VERSION_ID`,
   pack as an erofs image or keep as a directory tree, place it under `/var/lib/extensions`, and
   merge.
3. Track what was installed in `/var/lib/foundation/compat/<name>.json` (source NVR, checksums,
   files).
4. `Remove(name)`: unmerge and delete.
5. After an OS update `Revalidate()` re-checks every compat extension against the new base, because
   library versions change. Extensions that no longer validate are not merged and are reported as
   needing attention.

Limits: no scriptlets, and a package's `/etc` files become defaults copied into `/etc` where nothing exists and its `/var` directories become `tmpfiles.d` entries (see [system extensions](features/system-extensions.md)); and
kernel modules built for another kernel do not work. **Unverified** end to end on our image.

### Sysext and confext management

A thin, honest layer over `systemd-sysext`/`systemd-confext`: list images and their merge state,
merge, unmerge, refresh. It also reports why an extension is not merged (release-file mismatch).

### Updates (OTA) and slots

Payloads come from the `root` mkosi profile (`mkosi.output/root/`: `ecp_<version>.root-x86-64.raw.zst`,
`ecp_<version>.efi`, checksums), whose names match the transfer files. Publishing (a merged,
signed `SHA256SUMS` at the source URL) is not built.

- Delegate to `systemd-sysupdate`. **Verified:** Tumbleweed's `systemd-experimental` 261.4 ships
  `systemd-sysupdate` and `systemd-sysupdated`, with D-Bus interfaces
  `org.freedesktop.sysupdate1.Manager`, `.Target` and `.Job`. `foundationd` should call
  `sysupdated` rather than the CLI. **Unverified:** that `sysupdated` exposes the progress and
  target granularity we need, and how it handles our `ecp-root_<version>` / `_empty` slot labels.
- `Check` queries the configured source; `Download` and `Apply` write only the inactive slot and
  the new UKI (named for boot counting, `ecp_<version>+3.efi`). `Apply` never changes the running
  slot. The update takes effect at the next reboot, and boot counting returns to the previous slot
  if the new one fails.
- `GetSlots` reports both slots (version, active, bootable, last boot result). `Rollback` makes the
  other slot the default for the next boot.
- Per AGENTS.md, signed image verification and the tested interrupted-update fallback come before
  OTA is described as working.

### Boot and recovery

- `GetBootState` reads boot-assessment state (`bootctl`, `systemd-bless-boot`, the
  `LoaderEntrySelected` EFI variable).
- `RebootToRecovery` sets a one-shot boot entry for the recovery partition (`bootctl set-oneshot`)
  and reboots through logind (`org.freedesktop.login1.Manager.Reboot`).
- The recovery environment runs the same `foundationd` and API against the main disk, so recovery
  actions (roll back, re-deploy, factory reset) use the same code and the same safety rules.

### Diagnostics

- `Doctor()` validates layout health: partition labels, ESP free space, slot sanity, boot
  assessment, sysext validity, pending updates, time and network reachability of the update source.
- `CollectBundle()` writes a redacted support archive (journal excerpts, `systemd-analyze`, hardware
  summary). No machine identifiers or secrets by default.

## Configuration and state

Two kinds of data, kept apart because they are owned differently:

| | Configuration | State |
|---|---|---|
| What | Declared intent (OTA source, channel, behaviour) | What the machine has done (provisioned, last check) |
| Written by | Image (defaults), admin, or `Config.Set` | `foundationd` only |
| Format | systemd-style `key=value` files | JSON |
| Location | defaults in `/usr/lib/foundation/defaults.d/`, overrides in `/etc/foundation/foundation.conf.d/` | `/var/lib/foundation/state.json` |

Layering (later wins): image defaults, then `/etc` drop-ins in lexical order. `Config.Set` writes one
generated drop-in (`/etc/foundation/foundation.conf.d/90-api.conf`); admins edit their own files and
the API never touches them. Changing a value emits `Config.Changed(key)`.

### Namespaced keys and a schema

Every key is registered in a schema shipped in the image (`/usr/lib/foundation/schema.d/`) with a
type, default, description and mutability, so unknown keys and bad values are rejected. Examples:

```
ota.url=https://updates.ersa.invalid/ecp     # vendor default; placeholder today
ota.channel=stable
ota.auto_check=true
ota.auto_download=false
system.developer_mode=false
```

State keys (not settable through `Config.Set`):

```
system.provisioned=false        # Android's DEVICE_PROVISIONED
system.provisioned_at=
```

Secrets (update-server tokens) are not configuration. They use systemd credentials
(`systemd-creds`, `LoadCredential=`) or a root-only file, never the config files.

### Derived files

Some consumers cannot read our config. `systemd-sysupdate` transfer files are the first: they take
the source URL as text. Changing `ota.url` or `ota.channel` therefore renders override files into
`/etc/sysupdate.d/` from templates shipped in `/usr/lib/foundation/templates/`, validates them, and
only then replaces the old ones atomically. The current transfer files in the image hard-code a
placeholder URL (`updates.ersa.invalid`); the defaults and the template replace that.
**Unverified:** that `/etc/sysupdate.d` overrides take precedence over `/usr/lib/sysupdate.d` by
file name in the shipped systemd version.

### Capability gates ("things tucked away")

Features are visible or usable depending on state and configuration, enforced in the daemon (not
only hidden in clients):

| Gate | Condition | Examples |
|---|---|---|
| Setup-only | `system.provisioned=false` | `CompleteProvisioning()`, first-boot helpers. After provisioning these return `AlreadyProvisioned`. |
| Developer | `system.developer_mode=true` (admin sets it) | The writable-`/usr` overlay, switching to a testing channel |
| Live-only | Booted from installer media | The graphical installer and its polkit rule (shipped only in the installer sysext) |

- Clients ask `GetStatus()` for the capability list and show only what is available. `--help` does
  the same.
- systemd units use a marker file for conditions, because units cannot read JSON. Provisioning also
  creates `/var/lib/foundation/provisioned`, and first-boot units use
  `ConditionPathExists=!/var/lib/foundation/provisioned`.
- The image ships with `system.provisioned=false`, no state file and no machine identity, per the
  AGENTS.md rule that identity is initialized on first boot.

### Provisioning

Implemented (see `docs/provisioning.md` for the design and the open questions):

- `system.provisioned` and `system.provisioned_at` are state, latched one way by the service, with the
  marker file `/var/lib/foundation/provisioned` for units.
- `Manager.CheckProvisioning()` records evidence (an administrator's `gnome-initial-setup-done` file);
  a user unit calls it after login. `Manager.CompleteProvisioning()` is the explicit call for any other
  front end and returns `AlreadyProvisioned` the second time.
- An unattended `deploy --user` will call `CompleteProvisioning` on the target as its last step once
  `Deploy` exists.
- A factory reset wipes `/etc` and `/var`, which returns the machine to unprovisioned.

Not done: the capability gates that depend on it (setup-only calls other than these, "tucked away"
features) and the stage model for MDM enrolment described in `docs/provisioning.md`.

## CLI mapping

The CLI is a thin D-Bus client. It never implements behaviour; `--json` prints the daemon's signals
as JSON lines.

| Command | API |
|---|---|
| `foundation status` | `Manager.GetStatus`, `Updates.GetSlots`, `Boot.GetBootState` |
| `foundation deploy --disk DEV [...]` | `Deploy.PlanDeploy` / `Deploy.Deploy` |
| `foundation apps list\|install\|remove\|update` | `Apps.*` |
| `foundation compat install\|remove\|list\|check` | `Compat.*` |
| `foundation ext list\|merge\|unmerge\|refresh` | `Extensions.*` |
| `foundation update check\|download\|apply` | `Updates.*` |
| `foundation rollback`, `foundation slots` | `Updates.Rollback`, `Updates.GetSlots` |
| `foundation reboot [--recovery\|--firmware]` | `Boot.*` |
| `foundation config get\|set\|list` | `Config.*` |
| `foundation doctor`, `foundation diag` | `Diagnostics.*` |
| `foundation job list\|show\|cancel` | `Manager.ListJobs`, Job objects |

The existing `foundation pkg` commands become `foundation apps`, because "package" is ambiguous in
an RPM-based system.

## Other capabilities to consider

Not part of the first milestone. Listed so the API leaves room for them. Items 1 to 3 are the ones I
would add next because OTA and recovery depend on them.

1. **Doctor and boot health** (`Diagnostics`): needed to trust updates.
2. **Factory reset:** wipe `/etc`, `/var` and optionally `/home` while keeping the OS. Destructive;
   explicit confirmation. Requires a clear definition of what survives.
3. **Update channels and pinning:** stable and testing channels, and the ability to hold an image.
4. **Firmware updates:** a thin wrapper over fwupd (**verified:** 2.1.9 in Tumbleweed) for UEFI and
   device firmware, shown beside OS updates.
5. **Disk encryption:** LUKS with TPM2 enrollment for `/var` and `/home`.
6. **Secure Boot:** key enrollment and UKI signing status.
7. **Performance and power profiles** (AGENTS.md phase 3): measured, explicit rules, user power
   preferences respected.
8. **Developer overlay:** the writable `/usr` overlay, behind the developer gate.
9. **Kernel selection:** choosing between kernels delivered as UKIs, once we ship more than one.

Explicit non-goals: managing the base RPM set on an immutable installation (base changes come from
image builds), network configuration (NetworkManager), user accounts beyond provisioning
(`homed`/GNOME Settings), and per-user Flatpak.

## Code layout

```
foundation/foundation/    existing backend library and CLI (Flatpak, system); reused by workers
foundation/foundationd/   broker: D-Bus glue, polkit, config, state   (exists; job table, locks later)
  workers/                one module per job kind (deploy, apps, compat, update, ...)   (planned)
foundation_cli/           thin D-Bus client   (planned: the current CLI still calls backends directly)
foundation/data/dbus/     org.ersa.Foundation1.xml (the contract), policy and activation file
foundation/data/polkit/   org.ersa.foundation.policy
foundation/data/config/   schema.d (exists); defaults.d and templates (planned)
foundation/data/systemd/  foundationd.service
foundation/data/libexec/  foundationd launcher
```

Packaging: separate RPMs for the daemon and workers, the CLI, and the installer UI, so installed
systems do not receive installer-only pieces.

## Testing

- Backends keep the existing fake-runner unit tests.
- The broker is tested on a private bus (`dbus-run-session`) with polkit stubbed and workers
  replaced by mocks that replay recorded signals. The GUI is built against the same mock.
- Real behaviour (deploy, update, sysext, rollback, interrupted update, failed-boot fallback) is
  tested only in QEMU, on virtual disks and loop devices backed by files. Nothing automated writes
  to a real disk.
- Report what was actually tested; nothing is called working until it has been.

## Verified and unverified

Verified in this project: Tumbleweed `systemd-experimental` 261.4 contains `systemd-sysupdated` and
its D-Bus interface XML; Flatpak 1.18.4 with GObject typelib, fwupd 2.1.9 and polkit 128 exist in
Tumbleweed; the Flatpak CLI backend against Flatpak 1.16.

Unverified (check before relying on them): `sysupdated` behaviour with our slot layout and
progress reporting; `/etc/sysupdate.d` override precedence; the compat-sysext pipeline end to end;
the PyGObject runtime in the image and in recovery; the hook that detects finished first-boot
setup; seeding `/etc` on a deployed target.

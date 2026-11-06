# The management service (`foundationd`)

**Status:** built. Tested through a private D-Bus session with fake polkit, logind and boot tools; the daemon and
workers also ran in QEMU.

## What it is

`foundationd` is a root D-Bus service, `org.ersa.Foundation1`, started on demand. It is the **single management plane**:
the command line, the installer, a future settings applet or an enrolment agent all call it. It authorizes every change
with **polkit** and runs long work as **jobs** in separate worker processes, so a crashing worker never takes the
service down and a job's log and result survive a restart.

## The interfaces

| Interface | What it offers | Polkit action |
| --- | --- | --- |
| `Manager` | status, system facts, base packages, job list, provisioning check | none (read) / `provision` |
| `Config` | settings with a schema, layered files, `Set`/`Reset` | `config` |
| `Job` | state, steps, progress, logs, cancel (one object per job) | `jobs` (cancel) |
| `Boot` | slots, default/next boot, copy and release a slot, reboot | `boot`, `slots` |
| `Deploy` | list disks, plan, install to a disk | `deploy` (always asks) |
| `Apps` | Flatpak list/install/remove/update/remote | `apps` |
| `Compat` | RPM files and bundles of repository packages as extensions | `compat` |
| `Repos` | repository list, search, info, enable, refresh | `repos` |
| `Extensions` | list, merge, unmerge, refresh sysext/confext | `extensions` |

The contract is `foundation/data/dbus/org.ersa.Foundation1.xml`. `Manager.GetStatus` lists which capabilities the
running service has, so a client can adapt to an older or newer service.

## Jobs

An operation that can take time returns a **job object** at once. A job has steps (`prepare`, `download`, `merge`...),
a progress fraction, a log, and a **commit point** before which cancelling is clean. Jobs take **locks**
(`extensions`, `repos`, `disk`...), so two operations that would collide fail fast with `Busy`. History is kept under
`/var/lib/foundation/jobs`. Files (an RPM) cross the API as descriptors and are copied into a private staging
directory first; the daemon never opens a path a client chose.

## Safety rules the service enforces itself

- Every string that reaches a worker or a command line is checked against a narrow pattern and may not start with `-`.
- Polkit is checked before anything starts. A denied call starts nothing.
- Destructive operations repeat their target as a confirmation (a disk path).
- Packages are verified against pinned keys, unpacked by a strict reader, never run scripts, and are labelled for SELinux.

## Delegation

Where a system API exists it is used: logind for rebooting, libflatpak for applications, libsolv for package solving,
direct reads of the partition table, EFI variables, mountinfo and sysfs. Commands remain only where nothing else exists:
`bootctl`, `sfdisk`, `systemd-repart` and mtools (installing), `systemd-sysext`, `rpm`/`rpm2cpio`/`rpmkeys`, `gpg`.

## Where things are

`foundationd/api/` (one module per interface), `foundationd/workers/`, `foundationd/backends/` (the real work),
`foundation/data/` (the XML, polkit policy, units, shipped repositories, keys and catalog).

## Limits

- Not built: `Updates`, `Diagnostics`, `RebootToRecovery`, a first-boot front end that calls `CompleteProvisioning`,
  per-worker sandboxes.
- The unit uses no mount-namespace sandboxing, because an overlay made for `systemd-sysext` inside a private namespace
  would be invisible to the system.

See also: [docs/foundationd.md](../foundationd.md) (design and open questions), the
[developer guide](../../foundation/README.md), [command line](foundation-cli.md).

# The management service (`foundationd`)

## What it is

`foundationd` is a root D-Bus service, `org.ersa.Foundation1`, started on demand. It is the single management plane: the
`foundation` command line ([command line](foundation-cli.md)), Deck ([Deck](deck.md)), the installer and any future client call
it. It authorizes every change with polkit and runs long work as jobs in worker processes, so a crashing worker never takes
the service down and a job's log survives a restart.

## Interfaces

| Interface | Offers | Polkit action |
| --- | --- | --- |
| `Manager` | status, system facts, base packages, job list, provisioning | none / `provision` |
| `Config` | settings with a schema, `Set`/`Reset` | `config` |
| `Job` | state, steps, progress, logs, cancel | `jobs` (cancel) |
| `Boot` | slots, default/next boot, release a slot, splash, reboot | `boot`, `slots`, `splash` |
| `Deploy` | list disks, plan, install, preseeded install | `deploy` (always asks) |
| `Apps` | Flatpak apps and application sources, `InstallFrom` | `apps`, `apps-sources` |
| `Compat` | RPM files and repository bundles as extensions | `compat` |
| `Repos` | list, search, info, enable, add, remove, refresh | `repos` |
| `Updates` | install, check, install latest, restart, pending | `updates` |
| `Extensions` | list, merge, unmerge, refresh sysext/confext | `extensions` |
| `Remote` | SSH server and keys | `remote` |

The contract is `foundation/data/dbus/org.ersa.Foundation1.xml`. `Manager.GetStatus` lists the capabilities of the running
service so a client can adapt. Reads need no authorization.

## Jobs

A change returns a job object at once. A job has steps, a progress fraction, a log and a commit point before which cancelling
is clean. Jobs take locks (`extensions`, `repos`, a disk, ...), so colliding operations fail fast with `Busy`. History is in
`/var/lib/foundation/jobs`. Files cross the API as descriptors and are copied into a private staging directory first.

## Safety rules the service enforces

- Strings that reach a worker or command line are checked against narrow patterns and may not start with `-`.
- Polkit is checked before anything starts; a denied call starts nothing.
- Destructive operations repeat their target as confirmation (a disk path).
- RPMs are verified against pinned keys, unpacked by a strict reader, never run scripts, and are labelled for SELinux.
- Installing an update without checking its signature (`Updates.InstallLatestUnsigned`, `--unsigned`) is refused unless
  `system.developer_mode` is on.

## Where things are

`foundationd/api/` (one module per interface), `foundationd/workers/`, `foundationd/backends/` (the real work) and
`foundation/data/` (XML, polkit policy, units, keys, catalog). The unit uses no mount-namespace sandbox, because an overlay
made for `systemd-sysext` would be invisible to the system.

## Limits

Not built: `Diagnostics`, `RebootToRecovery`, per-worker sandboxes. See [design notes](../foundationd.md) and the
[developer guide](../../foundation/README.md).

## Tested

Tested through a private D-Bus session with fake polkit, logind and boot tools, and parts ran in QEMU. Not tested on hardware, with SELinux enforcing, or with real polkit prompts. `Diagnostics` and a recovery entry do not exist.

# Configuration and provisioning

## Settings

```
foundation config list
foundation config get ota.channel
foundation config describe ota.channel     # meaning, type, default
foundation config set ota.channel testing
foundation config reset ota.channel
```

Settings have a schema, so a wrong type or a value outside the allowed choices is refused. Layers, last wins: the schema's
defaults (`/usr/lib/foundation/schema.d`), image defaults (`/usr/lib/foundation/defaults.d`), then local changes in
`/etc/foundation/foundation.conf.d`, so an OS update can change defaults without losing a local choice. Setting a value needs
the polkit action `org.ersa.foundation.config`; reading does not.

| Key | Default | Effect |
| --- | --- | --- |
| `ota.url` | `https://github.com/Ersa-ECPX/ecp-mkosi/releases/download/updates` | where `Updates.Check` and `Updates.InstallLatest` look: `<ota.url>/latest-<arch>.json` |
| `ota.channel` | `stable` (or `testing`) | none yet |
| `ota.auto_check` | true | `foundation-autoupdate.timer` checks about 10 minutes after start and every 6 hours (random delay); a new version is announced |
| `ota.auto_update` | false | the same service also downloads and installs the newer update (active at the next restart); setting it needs the administrator's password; needs `ota.auto_check` |
| `system.developer_mode` | false | allows an update without a signature (`foundation update install --unsigned`, Deck's "Install without checking the signature"); shown in `foundation status`; needs the administrator's password |
| `apps.default_apps` | `io.github.kolunmi.Bazaar` | Flatpak ids (space-separated) installed once when the computer is first online; empty for none ([applications](applications.md)) |

See [updates](updates.md) for how the update keys are used.

## Provisioning

`system.provisioned` is false until first-boot setup finishes, then true, one way. It is machine state
(`/var/lib/foundation/state.json`), not a setting; latching also creates `/var/lib/foundation/provisioned` for
`ConditionPathExists=` in units. It latches in one of two ways:

- **Evidence** (`Manager.CheckProvisioning`, no authorization): the service looks for an administrator's
  `~/.config/gnome-initial-setup-done`. A user path unit watches for that file; nothing polls.
- **Explicit** (`Manager.CompleteProvisioning`, polkit `provision`): for a front end other than GNOME Initial Setup, such as
  a custom setup app or enrolment agent. A second call fails with `AlreadyProvisioned`.

`foundation status` shows it. Design discussion: [docs/provisioning.md](../provisioning.md).

## Limits

- No first-boot front end of our own: GNOME Initial Setup creates the account.
- No device-management agent, device identity or enrolment; only the state it would use exists.

See also: [management service](management-service.md), [updates](updates.md).

## Tested

Tested through the daemon (schema, layering, set and reset with polkit, persistence across a restart) with unit tests. `ota.channel` is stored but nothing reads it yet. Not tested on a booted image.

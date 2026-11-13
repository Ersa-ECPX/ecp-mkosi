# Configuration and provisioning

**Status:** built; tested through the daemon (schema, layering, set and reset with polkit, restart persistence).
The `ota.*` keys are stored but nothing reads them yet.

## Settings

```
foundation config list                 # every setting and its value
foundation config get ota.channel
foundation config describe ota.channel # what it means, its type, its default
foundation config set ota.channel testing
foundation config reset ota.channel
```

Settings have a **schema**, so a value of the wrong type or outside its allowed values is refused. Defaults come from
`/usr/lib/foundation/defaults.d`, the schema from `/usr/lib/foundation/schema.d`, and local changes are written to
`/etc/foundation/foundation.conf.d`, so an OS update can change defaults without losing a local choice. Changing a
setting needs the polkit `config` action; reading does not.

Current keys: `ota.url`, `ota.channel` (default `stable`), `ota.auto_check` (default true), `ota.auto_download`, and
`system.developer_mode`.

What each does today (the others are stored but nothing reads them yet):

| Key | Default | Effect now |
| --- | --- | --- |
| `ota.url` | `https://github.com/Ersa-ECPX/ecp-mkosi/releases/download/updates` | the update window's "Check for Updates" reads `<ota.url>/latest-<arch>.json` |
| `ota.channel` | `stable` | none yet |
| `ota.auto_check` | true | none yet (there is no background check) |
| `ota.auto_download` | false | none yet |
| `system.developer_mode` | false | shown in `foundation status`; no developer feature exists yet |

## Provisioning

`system.provisioned` is false until first-boot setup finishes, then true, one way. It is machine **state**
(`/var/lib/foundation/state.json`), not a setting, and latching also creates `/var/lib/foundation/provisioned` for
`ConditionPathExists=` in units. It latches in one of two ways:

- **Evidence** (`Manager.CheckProvisioning`, no authorization): the service looks for an administrator's
  `~/.config/gnome-initial-setup-done`. A user path unit watches for that file in the new account's session; nothing polls.
- **Explicit** (`Manager.CompleteProvisioning`, polkit `provision`): for a front end that is not GNOME Initial Setup,
  such as a custom setup app or an enrolment agent. A second call fails with `AlreadyProvisioned`.

`foundation status` shows it. The design discussion (a custom first-boot app, device management) is in
[docs/provisioning.md](../provisioning.md).

## Limits

- No first-boot front end of our own: GNOME Initial Setup creates the account.
- No device-management agent, device identity or enrolment: only the state it would use exists.

See also: [management service](management-service.md), [updates](updates.md).

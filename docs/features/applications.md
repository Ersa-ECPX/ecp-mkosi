# Applications (Flatpak)

**Status:** built. Listing, installing, removing and updating use libflatpak; tested with fake and real Flatpak
installations and by hand against Flathub in QEMU. The Flathub remote works on an installed system since the missing
`/var/lib/ca-certificates` was regenerated on first boot (confirmed on an installed disk).

## What it is

Applications are **system-wide Flatpaks** stored in `/var/lib/flatpak`, separate from the operating system image: an OS
update never touches them, and installing one never touches the OS. Each application brings its own runtime and is
sandboxed. Things that must live inside `/usr` (codecs, plugins) are bundles or extensions instead
([repositories and bundles](repositories-and-bundles.md)).

## Using it

```
foundation apps setup                  # add the Flathub application source (needs network)
foundation apps search firefox
foundation apps info org.mozilla.firefox
foundation apps install org.mozilla.firefox
foundation apps list
foundation apps update                 # update installed applications
foundation apps remove org.mozilla.firefox
```

Applications install from the `stable` branch of Flathub for the system's architecture. Removing an application keeps its
data in `~/.var/app`.

## How it works

`Apps` in [foundationd](management-service.md) uses libflatpak through GObject introspection for list, install, remove,
update and the remote, which gives structured errors and real progress; `search` and `info` run the `flatpak` command
because the library has none. Installs are jobs ([command line](foundation-cli.md)).

## Codecs

Flatpak applications get their codecs from the Flatpak runtime, not from the system, so the system's
[codecs bundle](multimedia-and-fonts.md) does not affect them and they do not need it.

## Limits

- **A default app store (Bazaar) is not installed yet.** The plan is to have first-login provisioning install it from
  Flathub when the machine is online; nothing of that is built.
- Search and info depend on the `flatpak` command; the first search needs network.
- A user-scoped (per-user) Flatpak installation is not managed by `foundation`.

See also: [docs/foundationd.md](../foundationd.md#applications-flatpak).

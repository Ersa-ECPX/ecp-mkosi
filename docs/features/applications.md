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

## Your own application sources

Besides Flathub, a system can have other signed Flatpak sources (a company's own, a mirror):

```
foundation apps remotes                                          # list the sources
foundation apps remote-add acme https://apps.acme.example/acme.flatpakrepo   # shows the signing key's fingerprint first
foundation apps remote-add mirror https://mirror.example/repo --key mirror.asc --title "Mirror"
foundation apps remote-disable acme   /   remote-enable acme   /   remote-remove acme
foundation apps install org.acme.Tool --from acme
```

Every source must be signed: a `.flatpakrepo` file has to carry its key, and a plain repository address needs `--key`. Addresses are
`https://` (or a local `file:///` mirror). `Apps.InspectRemote` shows the title, address and the key's fingerprint and owner before anything
is trusted; `AddRemote`, `SetRemoteEnabled` and `RemoveRemote` need the administrator's password (their own polkit action, apart from
installing applications). A source with applications installed from it cannot be removed. Sources are system-wide, so app stores such as
Bazaar offer their applications too. Deck's **Application Sources** page does the same. Tested against a fake libflatpak (23 tests); **not
yet run against the real library or a real second source**: whether libflatpak accepts the key exactly as the backend hands it over is the
first thing to check. Search and `apps info` still look at Flathub only.

## How it works

`Apps` in [foundationd](management-service.md) uses libflatpak through GObject introspection for list, install, remove,
update and the remote, which gives structured errors and real progress; `search` and `info` run the `flatpak` command
because the library has none. Installs are jobs ([command line](foundation-cli.md)).

## Codecs

Flatpak applications get their codecs from the Flatpak runtime, not from the system, so the system's
[codecs bundle](multimedia-and-fonts.md) does not affect them and they do not need it.

## Limits

- **The app store (Bazaar, `io.github.kolunmi.Bazaar`, GPL-3.0-or-later, from Flathub) is installed by the system once the computer is
  first online:** `foundation-default-apps.service` runs `Apps.SetupRemote` and `Apps.Install` for the ids in `apps.default_apps`
  (a Flatpak id list; empty for none), retries every ten minutes until it works, and records an id once it was offered, so an
  application you remove is not installed again. The logic is unit-tested with a fake `foundationd`; **it has not run on a booted image,
  and Bazaar itself has not been run here.** Bazaar installs through Flatpak directly (polkit asks for the administrator's password),
  not through `foundationd`, so its installs do not appear in Deck's Activity page.
- Search and info depend on the `flatpak` command; the first search needs network.
- A user-scoped (per-user) Flatpak installation is not managed by `foundation`.

See also: [docs/foundationd.md](../foundationd.md#applications-flatpak).

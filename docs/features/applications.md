# Applications (Flatpak)

## What it is

Applications are system-wide Flatpaks in `/var/lib/flatpak`, separate from the OS image: an OS update never touches them and
installing one never touches the OS. Each brings its own runtime and is sandboxed. Things that must live in `/usr` (codecs,
plugins) are [bundles](repositories-and-bundles.md) instead. Flatpaks take their codecs from the Flatpak runtime, so the
[codecs bundle](multimedia-and-fonts.md) does not affect them.

## Using it

```
foundation apps setup                  # add Flathub (needs network)
foundation apps search firefox
foundation apps info org.mozilla.firefox
foundation apps install org.mozilla.firefox [--from SOURCE]
foundation apps list
foundation apps update
foundation apps remove org.mozilla.firefox
```

Installs use the `stable` branch for the system's architecture. Removing an application keeps its data in `~/.var/app`.
Search and `apps info` look at Flathub only.

## Your own application sources

Besides Flathub, a system can have other signed Flatpak sources (a company's, a mirror):

```
foundation apps remotes
foundation apps remote-add acme https://apps.acme.example/acme.flatpakrepo     # shows the key's fingerprint first
foundation apps remote-add mirror https://mirror.example/repo --key mirror.asc --title "Mirror"
foundation apps remote-disable acme | remote-enable acme | remote-remove acme
foundation apps install org.acme.Tool --from acme
```

Every source must be signed: a `.flatpakrepo` file must carry its key, a plain repository address needs `--key`. Addresses
are `https://` or a local `file:///` mirror. `Apps.InspectRemote` shows title, address and the key's fingerprint and owner
before anything is trusted. `AddRemote`, `SetRemoteEnabled` and `RemoveRemote` need the administrator's password (polkit
action `org.ersa.foundation.apps-sources`, separate from installing applications). A source with applications installed from
it cannot be removed. Sources are system-wide, so app stores such as Bazaar offer their applications too; Deck has an
Application Sources page ([Deck](deck.md)).

Not yet run against the real library or a real second source; the first thing to check is whether libflatpak accepts the key
exactly as the backend hands it over.

## How it works

`Apps` in [foundationd](management-service.md) uses libflatpak through GObject introspection for list, install, remove,
update and remotes (structured errors, real progress). `search` and `info` run the `flatpak` command, because the library
has none. Installs are jobs ([command line](foundation-cli.md)).

## App store

The store, Bazaar (`io.github.kolunmi.Bazaar`, from Flathub), is installed once the computer is first online:
`foundation-default-apps.service` runs `Apps.SetupRemote` and `Apps.Install` for the ids in `apps.default_apps` (empty for
none), retries every ten minutes until it works and records each id once offered, so an application you remove is not
reinstalled. The logic is unit-tested with a fake `foundationd`; it has not run on a booted image, and Bazaar itself has not
been run here. Bazaar installs through Flatpak directly (polkit asks for the administrator's password), so its installs do not
appear in Deck's Activity page.

## Limits

- The first search needs network.
- A per-user Flatpak installation is not managed by `foundation`.

See also: [docs/foundationd.md](../foundationd.md#applications-flatpak).

## Tested

Listing, installing, removing and updating use libflatpak; tested with fake and real Flatpak installations and by hand against Flathub in QEMU (the Flathub remote works on an installed disk since `/var/lib/ca-certificates` is regenerated on first boot). Extra application sources are tested only against a fake libflatpak; the app store install has not run on a booted image.

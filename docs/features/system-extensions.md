# System extensions

## What it is

A system extension (sysext) overlays its `/usr` and `/opt` on the running system without changing the read-only root.
Three things use them: the [installer](live-medium.md) (on its medium only), [bundles](repositories-and-bundles.md)
(packages from repositories) and **compat RPMs** (a single RPM file from outside the base, such as a vendor's browser).

## Using it

```
foundation ext list [--kind sysext|confext]     # every extension image; which are merged, which managed
foundation ext merge | unmerge | refresh

foundation compat inspect foo.rpm               # check an RPM against this system, change nothing
foundation compat install foo.rpm
foundation compat list                          # enabled or disabled, merged or not
foundation compat remove foo
foundation compat revalidate                    # after an OS update: recheck every extension
```

## How a compat RPM is installed

`compat install` unpacks the payload into `/var/lib/extensions/<name>` with a strict reader of our own (not `cpio`, so a
crafted RPM cannot write outside its directory), writes an `extension-release` file (`ID` from the base's `os-release`,
`VERSION_ID` or `SYSEXT_LEVEL` if the base has them, `ARCHITECTURE`), labels the files for SELinux and merges. A failed merge
removes the extension again. `compat inspect` checks first:

- every requirement is looked up in the base's RPM database by name (rich dependencies such as `(a or b)` are evaluated;
  version constraints are not compared);
- every ELF file's needed libraries must be in the base's linker cache or in the package (libraries only plugins need are
  notes);
- files that would replace files of the base are conflicts.

Problems stop an install unless forced (`--force`); notes do not.

After an OS update, `compat revalidate` disables an extension that no longer fits (moved to
`/var/lib/foundation/compat/disabled`, never deleted) and re-enables it if a later base fits. Bookkeeping is
`/var/lib/foundation/compat/<name>.json`.

**Labels.** An overlay shows the extension's files with the extension's labels, so files labelled for
`/var/lib/extensions` would make `/usr` unreadable to confined services. Every file is labelled for the path it has once
merged ([SELinux](selinux.md)).

## Files outside `/usr` and `/opt`

- **`/etc` files become defaults.** They are kept in the extension (`usr/share/foundation/etc/<name>/`) and a `tmpfiles.d`
  file makes `systemd-tmpfiles` copy each one into the real `/etc` only where nothing exists yet, after every merge and at
  boot. Files you edited or the image ships are never replaced; a newer package default does not replace an old copy.
  Removing the extension removes the copies nobody changed. A configuration extension (confext) was built first and dropped:
  the kernel refuses an overlay as its upper layer, and our `/etc` is an overlay.
- **`/var` cannot be merged.** The package's `/var` directories become `tmpfiles.d` lines (created at boot, as root); files
  under `/var` are dropped and reported.
- Anything else outside `/usr`, `/opt` and `/etc` is dropped and listed.

## Scriptlets

Install scripts are never run: a package script running as root on an immutable system is what extensions avoid. foundation
reads what each install script does and reports what is missing ("Not done: ..." after install, `ListBundles` `notes`,
recomputed when a bundle is rebuilt). Removal scripts are ignored.

| The script... | Here |
| --- | --- |
| runs `ldconfig` | nothing needed; the cache is refreshed on merge |
| creates system users or groups (`useradd -r`, `groupadd -r`, `usermod -aG`, `sysusers.d`) | done: translated to `usr/lib/sysusers.d/foundation-<name>.conf`, applied after every merge by `ersa-sysusers-extensions.service` |
| creates a home directory (`useradd -m`) | done: a `tmpfiles.d` line |
| creates a user it cannot translate (regular user, shared id, unknown option) | not done; reported |
| presets or enables services | done: the named units are preset after the merge; removing the extension disables them |
| starts services | not done; reported |
| compiles settings schemas | done, see below |
| updates icon caches | done differently: icon directories are dated at build time so GTK and Qt read them instead of the stale cache |
| rebuilds other caches (MIME, fonts, certificates, application list), alternatives, anything else | not done; reported. A new file type or font may not be recognised until the next start |

The descriptions come from the script text, so an unusual script may show as "runs other commands".
`systemd-sysusers.service` itself is not ordered after `systemd-sysext.service`: that made an ordering cycle with the `/boot`
mount and udev that stalled boots for 90 seconds, hence the separate service.

## Generated files after a merge

Some files only work after a tool processed them, and the base image's cache cannot change. After every merge and at boot,
foundation reads the jobs in `/usr/lib/foundation/post-merge.d/*.json` of the image (a bundle cannot add one), gathers the
matching files from every enabled extension and runs the command into `/var/lib/foundation/generated/<name>/`. A failed job
is logged and its output removed.

```json
{"name": "gsettings", "collect": ["usr/share/glib-2.0/schemas/*.gschema.xml"], "run": ["glib-compile-schemas", "{out}"]}
```

The one job compiles settings schemas; without it a GLib or GNOME application from a bundle stops with "schema not
installed". The image sets `GSETTINGS_SCHEMA_DIR=/var/lib/foundation/generated/gsettings`
(`usr/lib/environment.d/60-foundation-generated.conf`). Another processed file type is one more JSON file.

## Tested and not tested

Tested: gathering, replacing, removing, failing safely, and a real `glib-compile-schemas` result read by a second process.
Not tested on a booted image: that the session receives the environment setting, and a real GNOME application from a bundle
using it.

## Limits

- An extension is not a sandbox; it runs with full privileges.
- Compat checks are by name and linker cache; they cannot prove an RPM works.
- Extensions must match the base's `ID` and version fields; a different base needs a rebuild.

See also: [repositories and bundles](repositories-and-bundles.md), [docs/foundationd.md](../foundationd.md).

## Tested

Tested with unit tests using real RPMs and fake base-image queries, and in QEMU for merging. Compat merging on a booted image under enforcing SELinux has not been run.

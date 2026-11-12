# System extensions

**Status:** built; tested with real RPMs and fake base-image queries, and in QEMU for merging. Compat merging on a
booted image under enforcing SELinux has not been run.

## What it is

A **system extension** (sysext) is a directory or image whose `/usr` and `/opt` are overlaid on the running system
without changing the read-only root. Three things in this system use them: the
[installer](live-medium.md) (on its medium only), [bundles](repositories-and-bundles.md) (packages from repositories),
and **compat RPMs** (a single RPM file from outside the base, such as a vendor's browser).

## Using it

```
foundation ext list [--kind sysext|confext]     # every extension image, of any origin; which are merged and which managed
foundation ext merge | unmerge | refresh        # overlay, remove the overlay, or re-merge to pick up changes

foundation compat inspect foo.rpm               # check an RPM against this system, change nothing
foundation compat install foo.rpm               # install it as an extension
foundation compat list                          # installed extensions: enabled or disabled, merged or not
foundation compat remove foo
foundation compat revalidate                    # after an OS update: recheck every extension
```

## Compat RPMs

`compat install foo.rpm` unpacks the payload into `/var/lib/extensions/<name>` (no scriptlets are run), writes an
`extension-release` file (`ID=` from the base's `os-release`, plus `VERSION_ID` or `SYSEXT_LEVEL` if the base has them,
and `ARCHITECTURE=`), labels the files for SELinux, and merges. What is checked first (`compat inspect`):

- every requirement is looked up in the base's RPM database **by name** (rich dependencies such as `(a or b)` are
  evaluated; version constraints are not compared);
- every ELF file's needed libraries must be in the base's linker cache or in the package; libraries that only plugins
  need are notes, not problems;
- files that would replace files of the base are conflicts.

**Problems stop an install** (unless forced); **notes do not**: files outside `/usr`, `/opt` and `/etc` are dropped and listed (`/etc` and `/var` are handled as described below), and
scriptlets are never run. The payload is read by a strict reader of our own, not by `cpio`, so a crafted RPM cannot write
outside its directory. A failed merge removes the extension again.

## After an OS update

`compat revalidate` rechecks every extension against the new base. One that no longer fits is **disabled** (moved to
`/var/lib/foundation/compat/disabled`, never deleted) and re-enabled if a later base fits again. Bookkeeping for each
extension is `/var/lib/foundation/compat/<name>.json`.

## Why labels matter

An overlay shows the extension's directories on top of the real ones, with the extension's labels. Files created under
`/var/lib/extensions` would carry that path's label and make `/usr` unreadable to confined services, so every file is
labelled for the path it will have once merged ([SELinux](selinux.md)).

## Limits

- An extension is not a sandbox; it runs with full privileges.
- Compat checks are by name and linker cache; they cannot prove an RPM works.
- Extensions must match the base's `ID` and version fields; a different base needs a rebuild.

See also: [repositories and bundles](repositories-and-bundles.md), [docs/foundationd.md](../foundationd.md).

## Files in `/etc` and `/var`

A sysext can only overlay `/usr` and `/opt`, so a package's other files need their own handling (compat RPMs and
[bundles](repositories-and-bundles.md) alike):

- **`/etc` files become defaults.** They are kept inside the extension (`usr/share/foundation/etc/<name>/`) and a `tmpfiles.d`
  file makes `systemd-tmpfiles` copy each one into the real, writable `/etc` **only where nothing exists yet**. So a file you
  edited, or one the image ships, is never replaced, and what you change afterwards stays yours. The copy happens after every
  merge and at boot. Removing the extension removes the copies nobody changed and leaves the ones you edited.
- **Why not a configuration extension (confext)?** A confext would merge the files over `/etc`, but it needs the layer below
  it writable, and the kernel refuses an overlay as that layer (`filesystem on ... not supported as upperdir`; our `/etc` is
  an overlay). It was built first and replaced after that test.
- **A newer default does not replace an old copy.** If a package updates its default file, the copy already in `/etc`
  keeps the old content (the same trade-off as `.rpmnew` files, without the `.rpmnew`).
- **`/var` cannot be merged.** There is no "varext", and `/var` is machine state. The package's `/var` directories become
  `tmpfiles.d` lines in the extension (created at boot, as root); files under `/var` are dropped and reported.

## Scriptlets

An RPM's install scripts (scriptlets) are **never run**: a script from a package, running as root on an immutable system,
is what an extension is there to avoid. Instead foundation reads what each script does and tells you what is missing:

| The script... | What happens here |
| --- | --- |
| runs `ldconfig` | nothing needed; the library cache is refreshed when the extension is merged |
| creates users or groups (`useradd -r`, `groupadd -r`, `usermod -aG`, or a `sysusers.d` file) | **done**: translated into `usr/lib/sysusers.d/foundation-<name>.conf` inside the extension, and `systemd-sysusers` applies it after every merge and at boot |
| creates a home directory (`useradd -m`) | **done**: a `tmpfiles.d` line creates it, owned by that user, since `sysusers.d` cannot |
| creates a user it cannot translate (a regular user, a shared id, an option foundation does not know) | **not done**; you are told |
| presets or enables services (`systemctl preset foo.service`, `%service_add_post`) | **done**: after the merge foundation presets the named units, so the preset files decide, exactly as the script did. Removing the extension disables them again; a rebuild for a new image keeps them enabled |
| starts services, or anything else about them | **not done**; you are told (start one with `systemctl`) |
| compiles settings schemas (`glib-compile-schemas`) | **done**, see "After a merge" below |
| updates icon caches | **done** differently: the extension's icon directories are dated at build time, so GTK and Qt ignore the base image's out-of-date cache and read the directories |
| rebuilds other caches (application list, MIME types, fonts, certificates) | **not done**; you are told. A new file type or font may not be recognised until the next start; applications still run |
| sets up alternatives, or anything else | **not done**; you are told |

Only the scripts that run when a package is **installed** are read; the ones that run on removal are ignored. Only system
accounts are translated (`sysusers.d` cannot make a regular user), and only commands whose every option is understood.

The notes show after `foundation compat install` and bundle installs ("Not done: ..."), are kept with the bundle (`ListBundles` `notes`),
and are recomputed whenever the bundle is rebuilt for a new image. They come from the text of the scripts, so an unusual
script may be described as "runs other commands". `systemd-tmpfiles --create` also runs after a merge, for the `/var`
directories above.

Not yet checked on a booted image: ordering `systemd-sysusers.service` after the extension merge (a drop-in does it),
and whether a real GNOME application from a bundle works without its schema cache rebuilt.

## After a merge: generated files

Some files only work once a tool has processed them, and the cache in the base image cannot be changed. After every merge
(and at boot) foundation reads the jobs in `/usr/lib/foundation/post-merge.d/*.json` of the **image** (a bundle cannot add
one), gathers the matching files from every enabled extension, and runs the job's command into
`/var/lib/foundation/generated/<name>/`. A job that fails is logged and its output removed, never left half made.

```json
{"name": "gsettings", "collect": ["usr/share/glib-2.0/schemas/*.gschema.xml"], "run": ["glib-compile-schemas", "{out}"]}
```

The first job compiles settings schemas. Without it an application from a bundle that has its own settings (GNOME, GTK and
any GLib application) stops at launch with "schema not installed". The image sets
`GSETTINGS_SCHEMA_DIR=/var/lib/foundation/generated/gsettings` (`usr/lib/environment.d/60-foundation-generated.conf`);
GLib reads that directory in addition to the system schemas. Nothing is specific to one desktop: supporting another
(for KDE, or anything else that needs a processed file) is one more JSON file plus, if it needs one, an environment line.

Tested: gathering, replacing, removing, failing safely, and a real `glib-compile-schemas` result found next to the system's
own schemas by a second process. Not tested on a booted image: whether the session actually receives the environment
setting (it relies on systemd's `environment.d`), and a real bundle application using it.

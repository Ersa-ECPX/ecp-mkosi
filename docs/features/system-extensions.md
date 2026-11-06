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

**Problems stop an install** (unless forced); **notes do not**: files outside `/usr` and `/opt` are dropped and listed, and
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

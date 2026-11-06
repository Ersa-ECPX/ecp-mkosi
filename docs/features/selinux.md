# SELinux

**Status:** built. The labelling code is unit-tested, and each problem below was found by booting an image and fixed.
The effect of a *bundle's* labels on a running system under enforcing SELinux has not been run yet.

## Why this needs care here

The system runs the `targeted` policy in enforcing mode, with a read-only root, a writable overlay for `/etc`, and system
extensions merged over `/usr`. Each of those shows files from a place whose labels are not automatically the right ones.
Real failures taught the rules:

| Symptom | Cause | Fix |
| --- | --- | --- |
| `useradd` and `dbus-broker` denied; `/etc` was `var_lib_t` | the `/etc` overlay takes the label of its **upper directory** | the upper layer is labelled `etc_t`: at image build (`file_contexts.subs`, `mkosi.finalize.chroot`) and in the installer's data-volume seed |
| Black screen: `dbus-broker`, `logind`, GDM denied reading `/usr` (`avc ... path="/usr/lib" dev="overlay" ... unlabeled_t`) | an overlay shows the **extension's** directories on top of the real ones, and an extension image had no labels | every extension file is labelled for its merged path |
| An extension under `/var/lib/extensions` would be `var_lib_t` | labels follow the path where files are stored | labels are computed for the path after merging, not the storage path |

## How extensions get their labels

`foundationd/backends/sellabel.py` reads the policy's own `file_contexts` (the file is sorted from least to most specific,
so the last matching entry wins; POSIX bracket classes are translated; unreadable entries are skipped and counted) and
sets `security.selinux` on every file, directory and symlink of the tree **for the path it will have when merged onto `/`**.
A path with no label in the policy is an error. It is used by:

- bundles ([repositories and bundles](repositories-and-bundles.md)) and compat RPM installs ([system extensions](system-extensions.md)),
- the installer sysext, whose script (`scripts/build-installer-sysext`) writes the same labels into the EROFS image.

## Labels written without SELinux

`foundation deploy` and the image build write labels as extended attributes offline, so no policy needs to be loaded:
the data volume's `/etc` overlay directories and the locale and time-zone files. A `home-relabel` unit on the image
relabels `/home`.

## Limits

- The kernel command line cannot carry a context (it contains colons, which `systemd.mount-extra=` uses as separators), so
  a mount that needs one would have to be a unit file; a RAM-disk mount of that kind failed in the initrd and was removed
  ([installer medium](live-medium.md)).
- Only the `targeted` policy of the image is read; another policy type is not handled.

See also: [storage](storage-and-persistence.md), [installer medium](live-medium.md).

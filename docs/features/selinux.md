# SELinux

## What it is

The system runs the `targeted` policy in enforcing mode, with a read-only root, a writable overlay for `/etc` and system
extensions merged over `/usr`. Each shows files from a place whose labels are not automatically right. Failures that set the
rules:

| Symptom | Cause | Fix |
| --- | --- | --- |
| `useradd` and `dbus-broker` denied; `/etc` was `var_lib_t` | the `/etc` overlay takes the label of its upper directory | the upper layer is labelled `etc_t` at image build (`file_contexts.subs`, `mkosi.finalize.chroot`) and in the installer's data-volume seed |
| Black screen: `dbus-broker`, `logind`, GDM denied reading `/usr` (`unlabeled_t` on the overlay) | an overlay shows the extension's directories, and the extension image had no labels | every extension file is labelled |
| An extension under `/var/lib/extensions` would be `var_lib_t` | labels follow the storage path | labels are computed for the path after merging |

## How extensions get labels

`foundationd/backends/sellabel.py` reads the policy's own `file_contexts` (sorted least to most specific, last match wins;
POSIX bracket classes translated; unreadable entries skipped and counted) and sets `security.selinux` on every file,
directory and symlink for the path it will have when merged onto `/`. A path with no label in the policy is an error. Used by:

- bundles ([repositories and bundles](repositories-and-bundles.md)) and compat RPMs ([system extensions](system-extensions.md));
- the installer sysext (`scripts/build-installer-sysext` writes the labels into the EROFS image).

`foundation deploy` and the image build write labels as extended attributes offline, so no policy needs to be loaded: the data
volume's `/etc` overlay directories and the locale and time-zone files. A `home-relabel` unit relabels `/home`.

## Limits

- The kernel command line cannot carry a context (colons clash with `systemd.mount-extra=`); a mount needing one must be a
  unit file. A RAM-disk mount of that kind failed in the initrd and was removed ([installer medium](live-medium.md)).
- Only the image's `targeted` policy is read.

See also: [storage](storage-and-persistence.md), [installer medium](live-medium.md).

## Tested

The labelling code is unit-tested, and each problem below was found by booting an image and fixed. The effect of a bundle's labels on a running system under enforcing SELinux has not been run.

# Testing an update locally (QEMU)

Goal: prove, on a real booted installation, that `foundation update install` writes the next version to the spare slot,
that the machine boots it, blesses it, rolls back when it should, and that your bundles come along.

Status: **not run yet.** The code is tested against fake disks and images; the steps below are the first real test.

## 0. What you need

- An **installed** disk (not the live installer medium: the update needs the A/B layout and a spare slot), built from a
  foundation package that already contains the update code. The updater lives in the image, so rebuild first.
- An **update archive** of a *different, higher* version built from the same sources.

Never update from a `-copy` slot (made by `foundation slot copy`); test from the original slot.

## 1. Build the baseline (host) and install it

```sh
scripts/build-foundation-package                       # the foundation RPM, with the update code
scripts/build x86-64 --profile installer               # the installer medium, version 0.1.0
```

Boot it in QEMU with a blank target disk, install with the installer, remove the installer disk, boot the installed system.
Optionally install a bundle first, so you can see it follow the update: `foundation compat install @codecs`.

## 2. Build and serve the update (host)

```sh
scripts/build x86-64 --profile root --image-version 0.1.1
scripts/make-update-archive 0.1.1                      # dist/update/ecp_0.1.1-x86_64.tar.zst  (add --sign for a signed one)
scripts/serve-update dist/update 8000                  # leave it running; it prints the guest command
```

## 3. Install it (guest)

```sh
foundation slot status                                 # note the running slot (not a -copy)
foundation update install http://10.0.2.2:8000/ecp_0.1.1-x86_64.tar.zst --unsigned
foundation update status                               # an update is waiting for a restart
foundation slot status                                 # 0.1.1 present as a standby slot, tries 3, the default entry
foundation compat bundles                              # (if you installed one) rebuilt for the new system: see the job log
```

The running slot is untouched. The new kernel image is installed last, so it never points at a missing root.

## 4. Boot it, bless it, roll back

1. `foundation reboot`. systemd-boot should pick `0.1.1` (3 tries). The "Update ready" notification should have appeared.
2. `foundation status`: running `0.1.1`, assessment pending, then good (blessed). `foundation update status`: nothing waiting.
3. `foundation compat bundles`: the bundle is `current` and merged, with no network needed.
4. Roll back: `foundation slot boot a --once --yes` (slot names are a and b; `foundation slot status` shows which holds the old version), reboot: you are on the old version and the old bundle is back
   (`journalctl -b -u foundation-bundles-select`).

## 5. Failures that must be safe

| Do | Expect |
| --- | --- |
| Ctrl-C the server during the download | the command fails; the running slot and the ESP are unchanged |
| Kill the `foundation update install` job (or the VM) while it writes | the spare slot stays empty or unbootable, never half-booted; run it again and it completes |
| Edit one byte of the archive's `root.erofs` | refused before anything is written ("does not match the checksum") |
| Install the same version again | refused unless `--force` |
| Install with a signature check on (no `--unsigned`) and no key installed | refused: "no image signing key is installed" |
| A `0.1.2` whose kernel image is broken | three failed tries, then boot falls back to the previous entry |

## Not covered

An update server and automatic checks, the shipped signing key, delta updates, and the `-copy` slot case (section 0).

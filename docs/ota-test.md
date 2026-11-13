# Testing an update locally (QEMU)

Goal: prove, on a real booted installation, that `foundation update install` writes the next version to the spare slot,
that the machine boots it, blesses it, rolls back when it should, and that your bundles come along.

Status: **not run yet.** The code is tested against fake disks and images; the steps below are the first real test.

## 0. What you need

- An **installed** disk (not the live installer medium: the update needs the A/B layout and a spare slot), built from a
  foundation package that already contains the update code. The updater lives in the image, so rebuild first.
- An **update archive** of a *different, higher* version built from the same sources.

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

To try the window instead of the command line: in the guest run `foundation config set ota.url http://10.0.2.2:8000`, open
"System Update" (`ersa-updater`), press **Check for Updates**, then **Update to 0.1.1**. A build made with `--sign` is installed
without the signature check being skipped; an unsigned one can only be installed from the menu ("Install from a File..."), where the
check can be switched off.

### Signing it with the development key

The image-update key lives in `~/Dev/signing-keys/image-update/` (its public half is in `foundation/data/keys/`, so images built after
that commit trust it). Sign the archive with:

```sh
GNUPGHOME=~/Dev/signing-keys/image-update/gnupg ECP_SIGN_PASSPHRASE_FILE=~/Dev/signing-keys/image-update/passphrase.txt \
  ECP_SIGN_KEY=3E08941D813471EA2EA509A2245AECF0314C00D5 scripts/make-update-archive 0.1.1 --sign
```

An installation built **before** that commit does not know the key. Give it the public key once (this machine's `/etc` is writable):
copy `~/Dev/signing-keys/public/ersa-image-245AECF0314C00D5.asc` into `dist/update/`, then in the guest
`curl -O http://10.0.2.2:8000/ersa-image-245AECF0314C00D5.asc`, check `gpg --show-keys ersa-image-245AECF0314C00D5.asc` prints the
fingerprint above, and `sudo install -Dm644 ersa-image-245AECF0314C00D5.asc /etc/foundation/keys/`. Then install without `--unsigned`.

## 3. Install it (guest)

```sh
foundation slot status                                 # note the running slot and its version
foundation update install http://10.0.2.2:8000/ecp_0.1.1-x86_64.tar.zst --unsigned
foundation update status                               # an update is waiting for a restart
foundation slot status                                 # the other slot (a or b) now shows version 0.1.1, tries 3, the default entry
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

An update server and automatic checks, the shipped signing key, delta updates, and syncing the older slot to the newer version after a good boot.

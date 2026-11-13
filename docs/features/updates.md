# Updates (OTA)

> **Note:** the `systemd-sysupdate` transfer definitions (`/usr/lib/sysupdate.d`) were removed: they named slots by `ecp-root_<version>` labels, which dm-verity replaced, and `foundation update install` is the update path. The sysupdate-based text below, `scripts/publish-update`, `scripts/serve-update` and `docs/ota-hosting.md` describe the older design and are not current.


**Status:** partly built. Installing an update **from a file** is built and unit-tested against fake disks and images. It has **not been
run on a booted system**, and there is no update server, no shipped signing key and no automatic checking. Do not rely
on it yet.

## What it does

`foundation update install FILE.tar.zst` takes one update archive and, in order:

1. **Checks it.** The archive is streamed and nothing is written until every entry is checked by name, type and size. It
   holds exactly `manifest.json` (first), `manifest.json.sig`, `root.erofs` and `uki.efi`; anything else is refused. The
   manifest names the product, architecture, version and the size and SHA-256 of the two big files. It must carry a valid
   GPG signature from an image signing key, unless you pass `--unsigned` for an image you built yourself.
2. **Writes the spare slot.** The running slot is never touched, so a power cut at any point leaves the system as it was.
   The target slot is emptied first (it cannot be booted half-written), the image is written, read back from the disk and
   compared with the manifest, and only then labelled and given its kernel image with three boot tries.
3. **Keeps your settings.** The new kernel image names the new slot, and takes this installation's data-volume mounts from
   the running one (so the per-install command line is no longer lost across an update).
4. **Rebuilds your bundles for the new system** before the restart: each bundle is solved against the new image's own
   package database, its libraries are checked against that image, and the result is stored for that image. A bundle that
   cannot be built (no network, a package gone, a library missing) is reported and rebuilt after the restart.
5. **Makes the new system the next boot** (`bootctl set-default`) and records that a restart is waiting.

Then it says so: the command prints "Restart to start using it: foundation reboot", `foundation update status` shows a
waiting update, and a desktop notification ("Update ready") is shown by `foundation-update-notify` in the user session.
If the new system does not start properly, boot counting returns to the one that worked.

## Bundles follow the system image

A bundle is built for exactly one system image (`built_for`, for example `ecp_0.2.0_20251101`). Its extension-release file
names that base, so systemd never merges it against another. At every boot `foundation-bundles select` (before
`systemd-sysext`) matches bundles to the running image: a bundle built for this image is put back (from the store of
images that is kept while another image ran), and one that exists only for another image is switched off. So after an
update the new image starts with its bundles ready, and after a rollback the old image gets its own bundles back, with no
download either way. If something is still missing, `foundation-bundles-refresh.service` rebuilds it once the network is
up (its failure never fails the boot), and `foundation compat refresh` does it on demand; bundles that already fit are
left alone.

## Making an update archive

```
scripts/build x86-64 --profile root                    # the root and kernel images
scripts/make-update-archive 0.2.0 [--sign]             # dist/update/ecp_0.2.0-x86_64.tar.zst
foundation update install dist/update/ecp_0.2.0-x86_64.tar.zst --unsigned
```

### Signing

Make the image signing key once, on a machine you trust (this is not the package repository key):

```
scripts/make-image-signing-key ~/ersa-image-key          # keep it outside the repository; prints the next steps
cp ~/ersa-image-key/ersa-image-*.asc foundation/data/keys/ && git add foundation/data/keys   # every image then trusts it
export GNUPGHOME=~/ersa-image-key/gnupg ECP_SIGN_PASSPHRASE_FILE=~/ersa-image-key/passphrase.txt ECP_SIGN_KEY=<fingerprint>
scripts/make-update-archive 0.2.0 --sign                 # or: scripts/release 0.2.0 --sign
foundation update install dist/update/ecp_0.2.0-x86_64.tar.zst        # no --unsigned
```

The system trusts a key only if its public half is installed as `/usr/share/foundation/keys/ersa-image-*.asc` or
`/etc/foundation/keys/ersa-image-*.asc`, and **no such key is committed yet**: until one is, an installed system refuses a signed
update with "no image signing key is installed". The check (`repos.verify_detached`) accepts only a good signature by a key
that is neither expired nor revoked; gpg's own exit status says "ok" for those, so it reads the status lines. The key does not
expire. To replace it, ship the new public key in an update first (any trusted `ersa-image-*.asc` is enough), sign with the new
key afterwards, and revoke the old one last.

Tested: a good signature; a manifest changed after signing; a signature by a key that is not trusted; garbage and empty
signatures; two trusted keys (key change); an expired key; a revoked key; the key script and the archive script together,
with the passphrase from a file. All of these refuse before the spare slot is touched. Not tested: an installed system
verifying with the key shipped in the image (that needs a rebuilt image with a committed key), and a downgrade to an older,
validly signed version (not refused today).

### Trying it in QEMU

Start the test server on the host, then install from inside the guest (QEMU's default network shows the host's loopback
as `10.0.2.2`):

```
scripts/serve-update dist/update 8000                                    # host; prints the guest command
foundation update install http://10.0.2.2:8000/ecp_0.2.0-x86_64.tar.zst --unsigned    # guest
foundation reboot
```

The command downloads the file itself (https from anywhere, plain http only from this computer or a private network),
installs it, and deletes the download. The guest needs a free standby slot and the installed disk layout, so test on an
installed disk, not the live installer medium.

## What is tested, and what is not

Tested: archive checks (tampering, extra or misplaced entries, wrong product or architecture, oversize, signatures), the slot
writing with fake partitions, the command line, bundle store/select/refresh/prebuild with fake repositories, and that the
archive script's output is accepted by the installer. **Not tested:** a real booted update in QEMU or on hardware (the
first one is described in [ota-test.md](../ota-test.md)), mounting the new image to prepare bundles, the desktop
notification, `bootctl` against real firmware.

## Still missing

- An update server, update discovery, and `ota.*` settings that something reads ([configuration](configuration.md);
  [ota-hosting.md](../ota-hosting.md)). Updates are installed from a file you have.
- A shipped image signing key and a release process for it.
- Delta updates, and a recovery system for an update that breaks both slots.
- The `systemd-sysupdate` transfer definitions and `scripts/publish-update` were removed; `foundation update install` does not use them.

See also: [boot and slots](boot-and-slots.md), [repositories and bundles](repositories-and-bundles.md),
[architecture.md](../architecture.md#updates-and-rollback-ab).

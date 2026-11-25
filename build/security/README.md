# Signing keys

Like AOSP's test keys and release keys: a build always has keys to sign with, and a production build points at its own.

| Directory | What | Used |
| --- | --- | --- |
| `testkeys/` | **public** test keys, secret halves included, in this repository | by default; anyone can build and sign test images and updates |
| `release/` (git-ignored) | your own keys: a directory, or a symlink to it | automatically, when it exists |
| `$ECP_SECURITY_DIR` | any directory laid out like these | instead of both, when set |

`scripts/security-dir` prints the one in use (`--kind` says `test` or `release`). Layout of a keys directory:

```
secure-boot/ersa-secure-boot.{key,crt,cer}     UEFI Secure Boot: signs systemd-boot and the kernel images (scripts/make-secure-boot-key)
image-update/ersa-image-<id>.asc               the PUBLIC update key
image-update/secret-key.asc + passphrase.txt   the secret update key, or instead a gnupg/ keyring with passphrase.txt
                                               (scripts/make-image-signing-key makes all of it)
```

What the keys do:

- `scripts/build` links `secure-boot/` to the chosen directory, so mkosi signs with it. A test-keys build also puts the test update
  key into the image (`/etc/foundation/keys/ersa-image-test.asc`), so it trusts updates signed with the test key. A release-keys
  build leaves it out and trusts only the keys the foundation package ships.
- `scripts/make-update-archive` signs the update with the chosen update key by default (`--no-sign` skips it). So an update built
  here installs on a system built here without `--unsigned` or developer mode.
- Test-signed images boot with Secure Boot off, or after enrolling `testkeys/secure-boot/ersa-secure-boot.cer`.
- `scripts/release` refuses the test keys. Never trust them on a real machine.

To build with your own keys: `ln -s ~/Dev/signing-keys build/security/release` (it needs `secure-boot/` and `image-update/`), or
`ECP_SECURITY_DIR=~/Dev/signing-keys scripts/build ...`. A production key must also be shipped in the image's trusted list
(`foundation/data/keys/ersa-image-*.asc`).

# Disk encryption

## What it is

The data volume (`ECP-DATA`: `/var` and `/home`, everything a person keeps) is a LUKS2 container. The operating system is not
encrypted: it is public, read-only and protected by [dm-verity](dm-verity.md). With a TPM the volume unlocks by itself at start;
without one it asks for a passphrase. A recovery key always works.

## How the unlock works

- The volume has up to three key slots: the **TPM**, a **passphrase** (a computer with no TPM, or one the person added), and a **recovery
  key** (always). The one-time key the volume is created with is wiped at the end of the install.
- The TPM slot is not bound to the measurements of one boot, which an update would change. It is bound to a **signed policy**: the
  kernel image carries a signature over what its own measurement (PCR 11) will be (`.pcrsig`) and the signer's public key (`.pcrpkey`),
  and the slot trusts that key. Every update brings a kernel image signed the same way, so the disk still unlocks; an image nobody
  signed, or a different computer's TPM, does not.
- The initrd unlocks the volume (`/usr/libexec/ecp/data-link`, run by `ecp-data-device.service` after the TPM is ready and the
  measurement barrier `systemd-pcrphase-initrd`) with `systemd-cryptsetup`. If the TPM cannot release the key, it asks on the screen
  or the serial console for the passphrase or the recovery key.
- The signer is the Secure Boot key of the build ([signing keys](../../build/security/README.md)), set with `SignExpectedPcr=` in
  `mkosi.conf`.

## Using it

```
foundation deploy install /dev/sdb --encrypt                          # asks for a passphrase only when there is no TPM
foundation deploy install /dev/sdb --encrypt --passphrase-file FILE --recovery-key-file KEY   # unattended
```

The **recovery key** (eight blocks of eight letters) is made by the client, passed to the service as a file descriptor and shown once
at the end: write it down and keep it away from the computer. The service never sends a secret back, and secrets are never arguments of
a job: they are staged in a private file that the worker reads once and deletes, and tools get them through their environment.
D-Bus: `Deploy.DeployEncrypted`.

## Limits

- Only the signed PCR 11 policy is bound. Not bound: PCR 7 (the Secure Boot state), so a computer whose Secure Boot is turned off
  still unlocks with a kernel image signed by our key. Binding PCR 7 as well is the stronger option, but a firmware or key-database
  update then needs the recovery key; it should follow a real-hardware Secure Boot test.
- The test keys are public (`build/security/testkeys`): an image signed with them is not protected by this. A release uses its own key.
- The unlock needs a TPM 2.0 that Linux exposes (`/dev/tpmrm0`). Without Secure Boot, the TPM policy does not stop someone who can
  change the boot loader, which is what Secure Boot is for.
- Not built: encryption of a system installed without `--encrypt` (a reinstall is needed), changing keys in Deck, per-user encrypted
  homes (`systemd-homed`), a FIDO2 key slot, an installer page (the command line only so far).

## Tested

Unit tests: the commands and their order, that no secret is on a command line, the refusals (no TPM and no passphrase, an image
without a signed policy), the initrd helper against fake devices. **Not yet run in QEMU with a TPM** (`scripts/qemu-encryption-test`,
needs an image built with this support); not run on hardware.

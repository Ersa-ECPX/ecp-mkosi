# Secure Boot

The platform can sign systemd-boot and every kernel image (UKI) with its own key, so a computer that trusts that key boots
them and refuses anything else.

## Build signed images

```
scripts/make-secure-boot-key                       # once: secure-boot/ersa-secure-boot.{key,crt,cer} (git ignores the folder; back it up)
scripts/build x86-64 --profile root --profile secureboot
scripts/build x86-64 --profile installer --profile secureboot
```

The `secureboot` profile (`mkosi.profiles/secureboot/mkosi.conf`) sets `SecureBoot=yes` with that key and certificate and
`SecureBootAutoEnroll=yes`. This is a different key from the update signing key (`scripts/make-image-signing-key`) and the
package repository key. **The profile itself has not been built yet** (mkosi cannot run in the sessions that wrote it); the
test below signs an existing image's boot files instead.

## Trusting the key

- **QEMU:** with firmware in Setup Mode (a fresh OVMF variable store), systemd-boot enrols the certificate from
  `loader/keys/auto/` by itself (`secure-boot-enroll force`), resets, and boots signed. Shown by the test.
- **A real computer:** put the firmware in Setup Mode (its menu clears the platform key) and either let systemd-boot enrol
  (`secure-boot-enroll force` in `loader/loader.conf`) or add `ersa-secure-boot.cer` as a `db` entry in the firmware menu.
  Firmware that only trusts Microsoft's keys refuses these images. Switching Secure Boot off in the menu always restores
  booting. **Not tested on any real computer.**

## Tested: `scripts/qemu-secureboot-test`

OVMF with Secure Boot enforcement in QEMU, a throwaway key, signing with sbsign. Passed on the 0.1.0 image:

| Case | Result |
| --- | --- |
| Setup Mode: systemd-boot enrols the key, resets, the signed kernel image boots to a login prompt, the kernel says it is locked down from EFI Secure Boot mode | passed |
| unsigned kernel image | refused (Access Denied) |
| kernel image signed by another certificate | refused |
| signed kernel image with one byte changed | refused |
| unsigned systemd-boot | refused |

Not tested: mkosi's own signing, real firmware, TPM measurements, the installer and `foundation update install` with signed
images.

## Known blocker for signed updates

A signed kernel image cannot be edited. `foundation update install` rewrites its command line (the slot's `root=` label and
this machine's data volume tokens), and `pe.rewrite_cmdline` refuses signed images, so **signed images cannot be installed by
update yet**. The command line has to be the same on every machine (root found from the booted entry or a dm-verity hash,
the data volume found on the same disk), and per-machine choices such as the splash theme already live in UEFI variables.
That change, with dm-verity, comes next (`docs/architecture.md`).

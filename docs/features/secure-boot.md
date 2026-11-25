# Secure Boot

Every boot component is signed, so the firmware can refuse anything that was not signed with the platform's key.

## Signing

systemd-boot and every kernel image (UKI) are signed with `secure-boot/ersa-secure-boot.{key,crt}` (git-ignored; back it up).
A computer without Secure Boot ignores the signatures. This key is separate from the update-signing and package keys.

```
scripts/make-secure-boot-key         # the real key, once; scripts/build makes a DEVELOPMENT key if none exists
scripts/release 0.2.0                # refuses the development key
```

## Trusting the key

- **QEMU / Setup Mode:** systemd-boot enrols the certificate from `loader/keys/auto/` itself (`secure-boot-enroll force`).
- **A real computer:** put the firmware in Setup Mode (clear the platform key) and let systemd-boot enrol, or add
  `ersa-secure-boot.cer` as a `db` entry in the firmware menu. Firmware that only trusts Microsoft's keys refuses these
  images; switching Secure Boot off restores booting. A shim or documented enrolment steps are still needed for ordinary users.

## The test

`scripts/qemu-secureboot-test --image DISK --uki UKI`: enrol and boot to a login prompt with the kernel locked down; then an
unsigned UKI, a UKI signed by another key, a signed UKI with one changed byte, and an unsigned systemd-boot must all be
refused (each: "Access Denied"). QEMU's firmware is emulated, so this is not proof for any real firmware.

## Tested

Every image is signed by mkosi (`SecureBoot=yes`); the mkosi-built 1.0.0 kernel image carries a signature by "Ersa Compute Platform Secure Boot". Firmware enforcement was tested in QEMU (`scripts/qemu-secureboot-test`, 8/8, 2026-11-25), but with a throwaway key that re-signs the same kernel image, so the test covers the firmware chain, not mkosi's own signature or key handling. Not tested on any real computer, with TPM measurement, or with `foundation update install` of a signed image on a signed boot.

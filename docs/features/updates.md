# Updates (OTA)

**Status: scaffolding only.** The pieces that apply and confirm an update exist and the offline parts are tested, but
**no update has been applied end to end**, there is no update server, no image signing, and no `foundation update`
command. Do not rely on updates.

## What it is meant to be

A new system version is downloaded as a whole root image and kernel image, written to the slot that is not running, tried
with a boot counter, confirmed by `systemd-bless-boot`, and rolled back automatically if it does not start. Applications
(Flatpak) and settings are unaffected by an OS update.

## What exists

- **Transfer definitions** for `systemd-sysupdate` (`mkosi.extra/usr/lib/sysupdate.d/10-root.transfer`, `20-uki.transfer`):
  write `ecp_<v>.root-x86-64.raw[.zst]` into the free root slot, install `ecp_<v>.efi` with `TriesLeft=3`. The URL is a
  placeholder (`updates.ersa.invalid`). `ProtectVersion=%A` keeps the running version safe.
- **The `root` image profile**, which produces those payloads (`scripts/build x86-64 --profile root`).
- **Scripts:** `scripts/publish-update` (assemble what a server serves, optionally signed), `scripts/serve-update`
  (a local HTTP server), `scripts/ota-test-definitions` (transfer files pointing at it, with signature checks off). Tested
  offline with fake payloads.
- **The other half, already working:** boot counting, blessing and fallback ([boot and slots](boot-and-slots.md)), and the A/B
  layout ([storage](storage-and-persistence.md)).
- **Settings:** `ota.url`, `ota.channel`, `ota.auto_check`, `ota.auto_download` exist ([configuration](configuration.md))
  but nothing reads them.

## What is missing

- Applying an update on a booted image in QEMU: the first real test is described in [ota-test.md](../ota-test.md).
- A `foundation update` command and an `Updates` interface (through `sysupdated`), including refusing an update from a
  `-copy` slot.
- Signing and verification of images (the test definitions use `Verify=no`), and a place to host them
  ([ota-hosting.md](../ota-hosting.md)).
- Per-install kernel command line: an updated kernel image is built with the label form of the data mounts, so on a
  system with another ECP disk attached the old confusion can return ([storage](storage-and-persistence.md#known-gap)).
- Bundles after an update: `foundation compat revalidate` exists; its behaviour across a real update is untested.
- A recovery system for an update that breaks both slots.

See also: [boot and slots](boot-and-slots.md), [architecture.md](../architecture.md#updates-and-rollback-ab).

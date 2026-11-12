# dm-verity and the machine-independent kernel image (plan)

**Status: the `foundation` side is written and unit-tested; the mkosi side is written but has never been built, and nothing has
been booted with dm-verity.**

Done and tested (unit tests with synthetic disks, no image): `foundationd/backends/verity.py` (root hash to partition UUIDs,
`roothash=` from a kernel image); slot discovery by root hash for slots labelled `ecp-root-a`/`ecp-root-b` (older
`ecp-root_<version>` slots still work); `foundation update install` of a **format 2** archive (the image, its hash tree, the
root hash in the signed manifest) into the standby slot's two partitions, which are then named by the hash, with the signed
kernel image copied byte for byte and the label never changed; a failed write leaves the slot unbootable; the older format 1
archive is refused for a dm-verity slot; `scripts/make-update-archive` makes format 2 archives from the verity build outputs.

Written, not built: the `verity` mkosi profile (`mkosi.repart.root-verity/`, `mkosi.repart.system-verity/`, `roothash` split
artifact, `/drivers/md/` initrd modules). Its configuration resolves as intended (`mkosi --json summary`).

Not done: the initrd generator that finds the data volume on the booted disk; the installer's disk layout and its copy of the
partitions; a first boot with dm-verity; the tampered-block and failed-update tests in QEMU; the slot tools that still read
versions from labels (`copy`, `release`, the installer) for verity slots. Existing 0.1.x installations have no hash
partitions: they need one reinstall to get the new layout (an update cannot add partitions).

## Why

Secure Boot (docs/features/secure-boot.md) signs the kernel image, but the root filesystem is a plain partition: someone with
disk access can change it and the signed kernel image still boots it. dm-verity puts the root's hash in the signed kernel
image's command line, so any changed block is refused. It also detects bit rot.

A signed kernel image cannot be edited (`pe.rewrite_cmdline` refuses it), but `foundation update install` edits the command
line today (the slot's `root=PARTLABEL=ecp-root_<version>` and this machine's data volume `systemd.mount-extra=PARTUUID=...`).
So signed updates need a command line that is **identical on every machine and slot**. dm-verity gives that too: the root is
found by the hash, not by a label.

## Checked

- The openSUSE kernel-default config (kernel-source, config/x86_64/default): `CONFIG_DM_VERITY=m`,
  `CONFIG_DM_VERITY_VERIFY_ROOTHASH_SIG=y`, `CONFIG_DM_VERITY_FEC=y`. It is a module, so `dm-verity` must be in
  `KernelInitrdModules` (today `default /fs/erofs/ /fs/btrfs/ /fs/overlayfs/`).
- systemd in the build tools tree has `systemd-veritysetup` and `systemd-veritysetup-generator`. Whether the **image's** initrd
  has them (the initrd is built from the image's packages) is not checked.
- Not checked: that EROFS on top of dm-verity behaves as expected with the A/B partitions in QEMU; nothing here has been booted.

## Design

1. **Separate hash partitions per slot** (root-a, verity-a, root-b, verity-b), with repart's `Verity=data` / `Verity=hash`. Hash
   tree for an 8 GiB slot at 4 KiB blocks with SHA-256 is about 65 MiB, so each hash partition is a fixed 96 MiB. Systemd finds
   such a pair by itself: the root partition's UUID is the first 128 bits of the root hash and the hash partition's the last
   128 bits, and it only looks on the disk the boot loader came from. No `root=` token and no `ecp-root_<version>` label.
   (Appending the hash tree inside the same partition was considered: systemd's discovery does not do it.)
2. **The command line becomes constant:** `roothash=<hash>` (put there by mkosi) plus the same flags everywhere. Per-machine and
   per-user choices live in UEFI variables (like the splash theme) or files; anything the machine *must* have to boot is
   found on the booted disk, never in NVRAM (a firmware reset or a moved disk would lose it).
3. **Data volume found on the booted disk:** an initrd generator reads `LoaderDevicePartUUID` (the ESP's PARTUUID), finds the
   `ECP-DATA` partition on the same disk and writes the `/var` and `/home` mounts. This replaces the per-install
   `systemd.mount-extra=PARTUUID=` tokens.
4. **Slots without labels:** `foundation` today reads the version in a slot from `ecp-root_<version>`. It would read it from the
   UKI name in the ESP, or from a small marker, and identify slots by partition number or the hash UUID. The slot code
   (`slots.py`, `layout.py`, `updates.py`, `gpt.py`) changes; the update archive carries the hash partition image and the root
   hash in the manifest, covered by the update signature.
5. **Update install** writes the data and hash partitions of the standby slot, sets both partition UUIDs from the root hash, and
   copies the signed UKI as is: no command line editing.

## Order of work

1. mkosi: repart files with `Verity=`, `roothash=` in the UKI, `dm-verity` and veritysetup in the initrd; boot it in
   QEMU and `veritysetup status` it. Needs an mkosi build.
2. The initrd data-volume generator; test in QEMU.
3. `foundation`: slots without labels, update format with the hash partition, no command line rewriting. Unit tests first
   (they do not need an image), then the OTA test in QEMU with a signed image under Secure Boot.
4. Optional later: sign the root hash (`SIGNED roothash`, kernel option above) so a hash in an unsigned place is also checked.

## Risks

- Every change to the root layout touches the installer and the A/B update. A bug in the initrd path stops every boot, the
  same reason the image-file pool was set aside: keep the A/B fallback and test a tampered image and a failed update.
- EROFS compression plus verity costs some read speed; measure before claiming anything.
- A 96 MiB hash partition per slot assumes slots stay 8 GiB.

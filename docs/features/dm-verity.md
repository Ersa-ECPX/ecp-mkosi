# dm-verity (root integrity)

**Status:** built; the only layout. An installed 1.0.0 disk boots with a verity root, and a copy with one changed root block does not
start; the log names dm-verity (`scripts/qemu-verity-test`, 9/9, QEMU, 2026-11-25). Not tested on real hardware.

## What it is

Each root slot is an EROFS partition plus a hash-tree partition (96 MiB, SHA-256, 4 KiB blocks). The kernel image's command
line carries `roothash=`, so the signed image decides what the root must contain: a changed block is refused, and bit rot is
detected. The root partition's UUID is the first half of the hash and the hash partition's the second; systemd finds the pair
on the disk the boot loader came from, so there is no `root=` and no per-version label. The command line is identical on every
machine and slot, which is what lets signed kernel images be copied unchanged by an update.

## Consequences

- `foundation` identifies a slot by its root hash, and a slot's labels (`ecp-root-a`, `ecp-verity-a`) never change.
- `update install` writes both partitions, names them from the root hash, copies the signed kernel image unchanged.
- No `slot copy` (a copy would have the same UUIDs). `slot release` gives a standby slot new random UUIDs.
- 0.1.x installs have no hash partitions and need one reinstall.

## Known gaps

- `/var` and `/home` are found by `LABEL=ECP-DATA` on the command line. Two ECP disks attached at once can confuse that; the
  planned fix is an initrd generator that finds the data volume on the booted disk.
- The older slot is not synced to the newer system after a good boot (each slot has its own root hash).
- EROFS compression plus verity costs read speed: measure before claiming anything. A 96 MiB hash partition assumes 8 GiB slots.

## Try it

```
scripts/build x86-64                 # the A/B disk
scripts/qemu-verity-test             # intact: verity root, labels, UUIDs = hash halves; tampered copy: refuses to boot
```

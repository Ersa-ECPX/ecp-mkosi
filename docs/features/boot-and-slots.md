# Boot, A/B slots and boot assessment

**Status:** boot counting and blessing were tested in QEMU through UEFI on an installed disk (the older label-based layout),
including a re-armed counter. The dm-verity slots described below are covered by unit and end-to-end tests with synthetic
disks and **have never been booted** ([dm-verity](dm-verity.md)). Not tested on real hardware.

## What it is

The disk holds two equal root slots, **slot A and slot B**: each is a read-only EROFS root partition (8 GiB) with its
dm-verity hash partition (96 MiB). Each slot has its
own **unified kernel image** (UKI) on the EFI system partition. A new system version is written to the slot that is
not running, tried with a **boot counter**, and kept only if it starts properly; otherwise the machine falls back to
the previous slot by itself.

## How a boot goes

1. UEFI starts **systemd-boot** (`EFI/BOOT/BOOTX64.EFI`, the removable-media path; no firmware boot entry is written).
2. systemd-boot lists the kernel images in `EFI/Linux/`. A name such as `ecp_0.2.0+3.efi` means "version 0.2.0, three
   tries left". Each boot attempt renames it (`+2-1`, `+1-2`, ...).
3. The kernel image carries its own command line, the same on every machine, with the root hash (`roothash=`) that names
   the root partition to mount. The command line is never rewritten: the image is signed for Secure Boot.
4. The initrd opens the root as a dm-verity device, mounts `/var`, builds the `/etc` overlay, and switches root.
5. When the graphical target is reached and no unit has failed, `systemd-bless-boot` removes the counter
   (`ecp_0.2.0.efi`): the entry is **blessed**. If three tries fail, systemd-boot skips it and boots the next entry.

## Using it

```
foundation status                        # boot slot, assessment and storage at a glance
foundation slot status                   # which slot is running, the boot assessment, every slot
foundation slot boot b [--once]          # choose what boots (once, or from now on)
foundation slot release b                # empty a standby slot (its kernel image and names) so an update can use it
foundation reboot [--firmware]
```

The assessment shows **good** after a blessed boot and **pending** while a counter is still on the entry. The slot
table says `blessed` or `N left`.

## How it works

- A slot is a pair of GPT partitions: a root partition (label `ecp-root-a` or `ecp-root-b`) and its hash partition
  (`ecp-verity-a`/`-b`). The labels never change, and neither do the slot names `a` and `b`; the version a slot holds is shown next to the name. What a slot holds is identified by its root hash: the root partition's UUID is
  the first half of the hash and the hash partition's the second half, and the kernel image `ecp_<version>[+tries].efi` whose
  command line has that `roothash=` is the one that boots it. A slot whose UUID no kernel image names is empty.
  `foundationd` reads the partition table, the ESP and the EFI loader variables directly (`backends/slots.py`).
- `slot boot` calls `bootctl set-default` with the entry id *without* its counter, so the choice survives the
  rename at blessing.
- `slot release` removes a standby slot's kernel image and gives its partitions new random UUIDs. There is no `slot copy`
  any more: a copy has the same root hash, so the same UUIDs, and nothing could tell the two slots apart.
- Blessing is skipped on a boot that has no counter: `systemd-bless-boot.service.d/10-ecp.conf` makes it conditional on
  systemd-boot having set `LoaderBootCountPath`, and orders it after the graphical target. The "no failed units" check
  also waits for the graphical target.

## What was found by testing

- Every boot after the first used to show `systemd-bless-boot` as **failed** ("Not booted with boot counting in
  effect"), because our own drop-in pulled it in on every boot. It is now skipped when no counter is active.
  Verified through UEFI on an installed disk: a blessed boot has zero failed units; with the counter renamed back to
  `+3` the next boot blesses the entry again.
- With the installer medium and an installed disk both attached, the boot counted on one disk's ESP while the system
  blessed the other's, so "good" never appeared. See [storage and persistence](storage-and-persistence.md).

## Limits

- No recovery entry exists yet; the recovery partition is reserved and empty.
- Updates that write new slots are scaffolding only ([updates](updates.md)).

See also: [architecture.md](../architecture.md#boot-chain), [storage and persistence](storage-and-persistence.md).

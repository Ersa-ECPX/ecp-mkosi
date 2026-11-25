# Boot, A/B slots and boot assessment

The system lives in one of two root slots. An update is written to the other slot and tried at the next restart; a boot only counts as good once the desktop is up, and a slot that does not start is given up on.

## Model

Two equal slots, a and b, each a root partition plus its hash partition. Each has its own kernel image (UKI) on the ESP. A
new version is written to the slot that is not running, tried with a boot counter, and kept only if it starts properly.

1. UEFI starts systemd-boot (`EFI/BOOT/BOOTX64.EFI`; no firmware entry is written).
2. It lists `EFI/Linux/`. `ecp_0.2.0+3.efi` means "three tries left"; each attempt renames it (`+2-1`, `+1-2`, ...).
3. The kernel image carries a constant command line with `roothash=`, which names the slot to mount.
4. The initrd opens the root as dm-verity, mounts `/var`, builds the `/etc` overlay, and switches root.
5. When the graphical target is reached with no failed unit, `systemd-bless-boot` removes the counter. An entry whose tries
   run out is skipped and the next one boots.

## Commands

```
foundation status                  # boot slot, assessment and storage
foundation slot status             # running slot, assessment, every slot
foundation slot boot b [--once]    # choose what boots, once or from now on
foundation slot release b          # empty a standby slot so an update can use it
foundation reboot [--firmware]
```

Assessment is **good** after a blessed boot and **pending** while a counter is on the entry.

## Details

- A slot is identified by its root hash (partition UUIDs); the kernel image whose `roothash=` matches boots it. A slot whose
  UUID no kernel image names is empty. `foundationd` reads the partition table, the ESP and the EFI loader variables directly.
- `slot boot` uses `bootctl set-default` with the entry id without its counter, so the choice survives blessing.
- Blessing is skipped on a boot without a counter (`systemd-bless-boot.service.d/10-ecp.conf`) and waits for the graphical target.
- With the installer medium and an installed disk both attached, the counter can run on one disk's ESP while the other is blessed
  ([storage](storage-and-persistence.md)).

## Limits

No recovery entry (the recovery partition is reserved and empty). Nothing in the image reboots a hung initrd; a power cycle or
hardware watchdog is needed.

## Tested

Tested in QEMU on 1.0.0 images (2026-11-25): installed boot, blessing, an update booted from the spare slot and marked good, a one-shot rollback with `slot boot a --once`; and, on 2026-11-26, the fall-back after a broken update (three failed tries, then the old system, marked good; [updates](updates.md)). Not tested on real hardware.

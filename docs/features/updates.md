# Updates (OTA)

An update replaces the whole system image. `foundation update install` writes it into the spare slot, the next restart tries it, and if it does not start properly the old system comes back. Applications and your data are not touched.

## Install

```
foundation update install FILE-or-URL [--unsigned] [--force] [--slot b]
foundation update status                 # an update is waiting for a restart
foundation reboot                        # the next boot tries the new slot
foundation slot boot a --once            # go back
```

In order, and nothing is written until every entry has been checked (name, type, size):

1. The archive holds exactly `manifest.json` (first), `manifest.json.sig`, `uki.efi`, `root.erofs`, `verity.img`. The signed
   manifest names the version, architecture, root hash and the size and SHA-256 of the big files. A valid GPG signature by a
   trusted image key is required. `--unsigned` (your own images) skips it, and is refused unless developer mode is on (`foundation config set system.developer_mode true`); a locally built update can instead be signed with your own key (`scripts/make-update-archive --sign`) and then needs neither.
2. The spare slot is emptied, the image and hash tree are written and read back against the manifest, the partitions are named
   by the root hash, and the signed kernel image is copied unchanged, last, with three boot tries.
3. Bundles (compat extensions) are rebuilt for the new image; failures are reported and retried after the restart.
4. The new entry is chosen for the next start only (`bootctl set-oneshot`). After a good boot `ecp-pin-default.service` makes it the default. A bad update therefore never becomes the default: after its three tries the old system boots again (tested, below).

Refused without `--force`: the version already installed or running, a version **older than the running one**, and a spare
slot that is the default boot entry. The running slot is never written.

## Checking and automatic updates

`ota.url` (default: the GitHub release `updates`) holds `latest-<arch>.json`, a small unsigned feed: version, file, size,
sha256. It only says what exists; the archive's signed manifest is what is verified. `foundationd` does the work:

| Method | Does | Authorization |
| --- | --- | --- |
| `Updates.Check` / `Available` | ask the source, report what was found | none |
| `Updates.InstallLatest`, `Install(fd)` | download and install (jobs) | `org.ersa.foundation.updates` |
| `Updates.Restart` | restart into a waiting update | same |

`foundation-autoupdate.timer` checks about ten minutes after boot and every six hours; it installs only if the administrator
turned on `ota.auto_update`. Nothing restarts the computer by itself. The Updates page of [Deck](deck.md) is a client
of the same methods.

## Bundles follow the image

A bundle is built for one image. At boot `foundation-bundles select` puts back the bundles built for the running image, so an
update starts with its bundles ready and a rollback gets the old ones back, with no download. `foundation compat refresh`
rebuilds what is missing.

## Making and signing an archive

```
scripts/build x86-64 --profile root --image-version 0.2.0
scripts/make-update-archive 0.2.0 [--sign]       # dist/update/ecp_0.2.0-x86_64.tar.zst and latest-x86_64.json
scripts/serve-update dist/update 8000            # QEMU guest: http://10.0.2.2:8000/
```

`scripts/make-image-signing-key DIR` makes the update key (keep it outside the repository; it is not the Secure Boot or package
key). The system trusts a key whose public half is in `/usr/share/foundation/keys/` or `/etc/foundation/keys/`; one is
committed (`foundation/data/keys/ersa-image-*.asc`). Tested: good signature, changed manifest, untrusted, expired and revoked keys,
garbage signatures, key change. Not tested: verification by an installed system using the key shipped in its image.

## Fallback test: passed (2026-11-26)

`scripts/qemu-fallback-test --archive FILE` installs from the installer medium, installs an update, overwrites the start of the new
slot's root, then boots until a boot reports; a boot that does not report in 200 s is killed, as a power cycle would. With a
newer update whose root image differs from the installed one it passed: the new entry was tried three times and did not start,
the fourth boot was the original system, marked good, and the broken entry was not the default (ESP: `ecp_1.0.2+0-3.efi`).
The first version of the update code pinned the new entry as the default, so systemd-boot kept booting it with no tries left and
the machine never came back; the update now chooses the new entry for one start (`bootctl set-oneshot`), and
`ecp-pin-default.service` pins a system as the default only after a good boot. `slot boot X` without `--once` still pins.
Not covered: a hung initrd does not reboot by itself (nothing in the image resets the machine; the test kills the VM, as a power cut
would), and a system installed before this change may have an old entry pinned.

Same image twice: a slot is named by its root hash, so the same image cannot also go into a second slot (the two would have one
name and the boot entries could not be told apart). The update refuses that and says where the image is; reinstalling into the
slot that holds it, to repair it, is allowed.

## Restarting by itself

A start that fails or hangs must not wait for someone to power-cycle the computer. Built (the files are checked by unit tests; **not
yet run in QEMU**: it needs an image built after this change, then `scripts/qemu-fallback-test --self-reboot`):

- the kernel command line has `panic=10`: a kernel panic (for example no root) restarts the computer after 10 s;
- on a boot that is on trial (the entry still carries its try counter), reaching the emergency target restarts the computer, in the
  initrd and in the system (`ecp-trial-reboot.service`); a blessed system that drops to emergency stays there;
- a boot that is on trial and not confirmed good within 15 minutes is restarted (`ecp-trial-boot-timeout.service`), which covers a unit
  that hangs without failing;
- systemd feeds the hardware watchdog every 30 s if the computer has one (`RuntimeWatchdogSec=30s`, system and initrd).

Each restart uses one of the three tries; after the last, the loader starts the system that worked. Not covered: a hang inside the
kernel or the firmware before systemd starts (only a watchdog the firmware already armed helps), and a system that was already
blessed and later hangs (the watchdog restarts it, nothing rolls it back).

## Not built

Delta updates, `ota.channel`, a recovery system for both slots failing, GNOME Software integration. Hosting: [ota-hosting](../ota-hosting.md).

## Tested

`foundation update install` was run on a booted system in QEMU (2026-11-25): written into the spare slot, booted, marked good, rolled back by hand, same version refused. Older versions are refused (unit-tested). Not tested: an interrupted install, a signed archive on a real boot, the GitHub feed, the update timer. The fallback after a broken update was tested in QEMU (see the end of this page).

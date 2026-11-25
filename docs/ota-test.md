# Testing an update locally (QEMU)

Run `scripts/qemu-fallback-test --archive a NEWER update (a real build with a different root image, see the script header):` it installs the installer medium onto a blank disk, installs an update into the spare slot,
breaks that slot and checks the fallback ([updates](features/updates.md)). It needs the installer image
(`scripts/build x86-64 --profile installer`) and an archive in `dist/update/` (`scripts/make-update-archive`). About 20 minutes.

To try the happy path by hand on an installed disk:

```
scripts/serve-update dist/update 8000                                       # host
foundation update install http://10.0.2.2:8000/ecp_0.2.0-x86_64.tar.zst --unsigned   # guest; first: foundation config set system.developer_mode true
foundation reboot
foundation status                                    # running 0.2.0, boot assessment good
foundation slot boot a --once --yes                  # roll back for one boot
```

Failures that must stay safe (all but the first are unit-tested; none was run on a booted system):

| Do | Expect |
| --- | --- |
| Kill the VM while `update install` writes | the spare slot stays empty or unbootable; run it again |
| Change one byte of `root.erofs` in the archive | refused before anything is written |
| Install the same or an older version | refused unless `--force` |
| Signature check on and no key installed | refused |
| A new slot that cannot mount its root | three failed tries, then the old slot boots (`qemu-fallback-test`: passed 2026-11-26) |

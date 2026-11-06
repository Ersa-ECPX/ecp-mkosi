# Testing an update locally (QEMU)

Goal: prove, with the payloads from the `root` profile, that `systemd-sysupdate` writes the next
version to the inactive slot, installs its kernel image with a boot counter, and that the machine
boots it, blesses it, and can roll back. Nothing here touches a real disk or a real server.

Status: **not run yet.** The scripts are tested offline with fake payloads; the steps below are the
first real test.

## 0. A rule before you start

**Never run `systemd-sysupdate` while booted from a `-copy` slot** (made by `foundation slot copy`).
The transfer files protect `ProtectVersion=%A`, the version in the running OS's `os-release`. A copy
runs version `0.1.0` from a partition labelled `ecp-root_0.1.0-copy`, so the protected label
(`ecp-root_0.1.0`) is the *other* slot, and the running partition is the one `sysupdate` may overwrite.
Test from the original slot (label equals the running version). The `Updates` interface in
`foundationd` will have to refuse this case; until it exists, this is on you.

## 1. Build two versions (host)

The running image is the normal build (`0.1.0`). The payload to update to is a higher version:

```sh
scripts/build x86-64 --profile root --image-version 0.1.1
scripts/publish-update            # mkosi.output/root -> dist/x86-64 (+ one SHA256SUMS)
scripts/serve-update              # http://127.0.0.1:8000/  (leave it running)
scripts/ota-test-definitions      # writes ./ota-test.d (source http://10.0.2.2:8000/x86-64/, Verify=no)
```

Copy `ota-test.d/` into the VM (`scp -r`, or a shared folder).

## 2. Update (guest)

```sh
foundation slot status                                   # note the running slot (must not be a -copy)
sudo systemd-sysupdate --definitions=$HOME/ota-test.d list
sudo systemd-sysupdate --definitions=$HOME/ota-test.d update
lsblk -o NAME,PARTLABEL,SIZE                             # the other root now labelled ecp-root_0.1.1
ls /boot/EFI/Linux                                       # ecp_0.1.1+3.efi next to the old kernel image
foundation slot status                                   # 0.1.1 present as a standby slot, tries 3
```

The running slot must be untouched. `sysupdate` writes only the inactive slot, then the kernel
image last, so a kernel image never points at a missing root.

## 3. Boot it, bless it, roll back

1. Reboot. systemd-boot should pick `0.1.1` (the higher version) with 3 tries.
2. `foundation slot status`: running `0.1.1`, assessment `pending` then `good` (blessed).
3. `foundation slot boot 0.1.0 --once --yes`, reboot: back on the old version for one boot.

## 4. Failures that must be safe

| Do | Expect |
| --- | --- |
| Stop `serve-update` (Ctrl-C) during the download | the update fails; the running slot and its kernel image are unchanged; no half-written `ecp_0.1.1+3.efi` |
| Kill `systemd-sysupdate` during `update` | same; run it again and it completes |
| Publish a `0.1.2` whose kernel image is broken (for example truncated) | three failed tries, then boot falls back to the previous entry |
| Fill the data volume | `/var` is not involved; the update still succeeds |

## Not covered

Signature checking (`Verify=no` here; production needs `SHA256SUMS.gpg` and the public key in
`/usr/lib/systemd/import-pubring.pgp`), HTTPS, resuming a partial download, the `Updates`
interface, and a `-copy` slot (see section 0).

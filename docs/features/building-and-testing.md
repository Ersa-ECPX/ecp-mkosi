# Building the images and testing

**Status:** the build scripts and unit tests are used daily. Images are built by hand (mkosi needs root or user namespaces
and about 15 GiB); the tests below run without them.

## Images

mkosi 27.1 builds three kinds of image from one tree: the **normal** image (A/B layout, for QEMU and development), the
**root** profile (the payloads an update would carry) and the **installer** profile ([installer medium](live-medium.md)).

```
scripts/build x86-64                        # the normal image
scripts/build x86-64 --profile root         # update payloads
scripts/build x86-64 --profile installer    # the installer medium (also builds its sysext)
```

`scripts/build` calls `scripts/prepare-build` first. Output goes to `mkosi.output/`. Set `ECP_WORK_DIR` to move the scratch
space (default `/tmp/ecp-work`, which needs about 15 GiB free).

| Script | What it does |
| --- | --- |
| `build` | check mkosi's version, prepare, run mkosi; builds the installer sysext for the installer profile |
| `build-podman` | run mkosi inside a pinned container instead of on the host |
| `prepare-build` | everything that must exist before mkosi: logos, wallpapers, the RPMs |
| `build-branding` | render the logos and wallpapers from `branding/*.svg` into `mkosi.branding/` |
| `build-foundation-package` | build the `foundation`, `foundationd` and `foundation-installer` RPMs into `mkosi.packages/` (runs the tests) |
| `build-rpms` | build the compiled RPMs inside a Tumbleweed container |
| `build-splash-package` | render the boot splash package's logo PNGs |
| `build-installer-sysext` | build the installer as a labelled system extension |
| `publish-update`, `serve-update`, `ota-test-definitions` | the update test scaffolding ([updates](updates.md)) |
| `qemu-smoke` | a boot smoke test in QEMU |
| `qemu-uefi-test` | the boot splash's UEFI variable on OVMF: write, reboot, find, remove (two boots; retries a boot that stops in the initrd) |
| `test-bundles-container` | run the repository and bundle code against the live repositories in a container |
| `build-wiki` | turn the documentation into one self-contained HTML wiki page (`docs/wiki/index.html`) with search and light/dark themes |
| `clean` | remove outputs and scratch space |

**Order for the installer:** `scripts/build-foundation-package`, then `scripts/build x86-64 --profile installer`. Packages
for the image come from the local repository `mkosi.packages/` plus openSUSE.

## Unit and end-to-end tests

```
cd foundation
python3 -m unittest discover -s tests          # about 380 tests, a minute and a half
env -u WAYLAND_DISPLAY GDK_BACKEND=x11 GSK_RENDERER=cairo ERSA_GUI_TESTS=1 \
    xvfb-run -a python3 -m unittest discover -s tests    # also the installer window tests
```

Tests start a real `foundationd` on a private D-Bus session with fake polkit, logind and boot tools, use real
`systemd-repart`, mtools, `rpmbuild` and `gpg` on image files and temporary repositories, and skip what a machine lacks
(several are skipped in the RPM build, which has only the `foundation/` directory; the tests that read the image
configuration then skip). `scripts/build-foundation-package` runs them as the package's check.

## Testing an image without a screen or a login

These methods found most of the boot bugs, and need only QEMU and KVM:

- **A shell in a headless guest.** Extract the kernel and initrd from the UKI (`objcopy -O binary --only-section=.linux`
  and `.initrd`), boot them with `-kernel`/`-initrd`, `-append "... systemd.debug_shell=/dev/ttyS1"` and a second serial
  port on a unix socket (`-serial unix:sh.sock,server,nowait`). That gives a root shell without credentials; type commands
  into the socket and read the output.
- **A real UEFI boot of an installed disk,** with fresh firmware variables, to see boot counting and blessing.
- **A screenshot of the live desktop:** `-monitor unix:mon.sock,server,nowait` and `screendump file.ppm`.
- **A debug shell on an installed disk** without changing its kernel image: an addon `loader/addons/dbg.addon.efi` built
  from systemd's addon stub with `objcopy --add-section .cmdline=...`.
- **Containers.** Package solving needs libsolv and an RPM database of the target; `scripts/test-bundles-container` runs it
  in a Tumbleweed container against the live repositories.

## Limits

- mkosi cannot run in unprivileged environments without user namespaces, so image builds are done on a build machine.
- There is no CI: the old workflow file is stale and does not build the current images.
- Nothing has been tested on real hardware.
- **An intermittent boot stop in QEMU (cause not found).** Roughly half of the boots of the root image stop in the initrd: the kernel reports the virtio disk (`vda`), then nothing happens for 90 seconds, and then systemd gives up waiting for `/dev/disk/by-label/ECP-DATA`, `/dev/disk/by-partlabel/ecp-root_*` and `/dev/disk/by-diskseq/N-part1` (the same `by-diskseq/9-part1` wait seen as a slow start job on a VM) and drops into emergency mode. What is known: with systemd and udev debug logging, udev handles events for the loop devices and serial ports but **never receives one for the virtio disk**, and the kernel log shows no output at all for the 90 seconds. It is *not* Plymouth (`plymouth.enable=0` changes nothing), the number of vCPUs, the CPU model, the host's storage (the image copied into RAM fails as often), the empty default CD-ROM, nor KVM (it also happens under pure software emulation, TCG). Legacy interrupts (`vectors=0`) were worse (0 of 4). It happens with one or two virtio disks. Not yet tried: other disk buses (NVMe, virtio-scsi) and a build with more kernel logging (`loglevel=7`, to see whether the partition scan of `vda` ever finishes). `scripts/qemu-uefi-test` retries around it (`--attempts`)

See also: [installer medium](live-medium.md), [developer guide](../../foundation/README.md#testing).

## Publishing the wiki (GitHub Pages)

The wiki is published at **https://pkgs-compute.ersa.dev/wiki/**, as part of the packages site: the workflow in the `packages`
repository clones this repository (and `foundation`, for its README) when a release is published and runs
`scripts/build-wiki`. To look at it yourself, `scripts/build-wiki docs/wiki/index.html` writes `index.html`, `assets/` (the
screenshots) and `.nojekyll`, and nothing in the page needs a server. `--cname DOMAIN` adds a `CNAME` for publishing it as a site of
its own.

Screenshots in `docs/images/` are real: installer windows are captured from the application in a headless session,
and terminal images are output captured from a booted installer image in QEMU, rendered as pictures.

# Building the images and testing

## Images

mkosi 27.1 (checked by `scripts/build`) builds three kinds of image from one tree: the **normal** image (A/B layout, for QEMU
and development), the **root** profile (the payloads an update carries) and the **installer** profile
([installer medium](live-medium.md)).

```
scripts/build x86-64                        # the normal image
scripts/build x86-64 --profile root         # update payloads
scripts/build x86-64 --profile installer    # installer medium (also builds its sysext)
```

`scripts/build` runs `scripts/prepare-build` first (logos, wallpapers, RPMs). Output goes to `mkosi.output/`; `ECP_WORK_DIR`
moves the scratch space (default `/tmp/ecp-work`, about 15 GiB). For the installer, run `scripts/build-foundation-package`
first; image packages come from the local repository `mkosi.packages/` plus openSUSE.

| Script | What it does |
| --- | --- |
| `build-podman` | run mkosi inside a pinned container instead of on the host |
| `build-branding`, `build-splash-package`, `make-wordmark` | render logos, wallpapers, splash PNGs and the wordmark ([branding sources](../branding.md)) |
| `build-foundation-package` | build the foundation RPMs into `mkosi.packages/` (runs the tests) |
| `build-rpms` | build the compiled RPMs in a Tumbleweed container |
| `build-installer-sysext` | build the installer as a labelled system extension |
| `release`, `publish-feed`, `make-update-archive` | publish images and the update feed ([updates](updates.md)) |
| `serve-update` | local HTTP server to test an update from a QEMU guest |
| `qemu-smoke`, `qemu-try` | boot smoke test; run the installer or installed disk by hand |
| `qemu-uefi-test`, `qemu-fallback-test`, `qemu-secureboot-test`, `qemu-verity-test` | boot-splash variable, failed-update fallback, Secure Boot and dm-verity checks in QEMU with OVMF |
| `test-bundles-container` | run the repository and bundle code against live repositories in a container |
| `build-wiki` | build the documentation as one self-contained HTML page |
| `clean` | remove outputs and scratch space |

## Unit and end-to-end tests

```
cd foundation
python3 -m unittest discover -s tests          # 648 tests, about two minutes, 53 skipped on this machine
env -u WAYLAND_DISPLAY GDK_BACKEND=x11 GSK_RENDERER=cairo ERSA_GUI_TESTS=1 \
    xvfb-run -a python3 -m unittest discover -s tests    # also the window tests
```

Tests start a real `foundationd` on a private D-Bus session with fake polkit, logind and boot tools, and use real
`systemd-repart`, mtools, `rpmbuild` and `gpg` on image files and temporary repositories. Tests skip what a machine lacks
(several in the RPM build, which has only `foundation/`). `scripts/build-foundation-package` runs them as the package check.

## Testing an image without a screen

Only QEMU and KVM are needed:

- **Headless root shell:** extract kernel and initrd from the UKI (`objcopy -O binary --only-section=.linux` / `.initrd`),
  boot with `-kernel`/`-initrd`, `-append "... systemd.debug_shell=/dev/ttyS1"` and a second serial port on a unix socket.
- **Real UEFI boot of an installed disk** with fresh firmware variables, to see boot counting and blessing.
- **Screenshot:** `-monitor unix:mon.sock,server,nowait` and `screendump file.ppm`.
- **Debug shell on an installed disk** without changing its kernel image: an addon `loader/addons/dbg.addon.efi` built from
  systemd's addon stub with `objcopy --add-section .cmdline=...`.
- **Containers** for package solving (`scripts/test-bundles-container`).

Screenshots in `docs/images/` are real: installer windows captured headless, terminal images captured in QEMU and rendered as
pictures.

## Publishing the wiki (GitHub Pages)

`docs/` is built by `scripts/build-wiki` (`scripts/build-wiki [OUTPUT]`, default `docs/wiki/index.html`; it writes
`index.html`, `assets/` and `.nojekyll`, needs no server, `--cname DOMAIN` adds a `CNAME`). `.github/workflows/docs.yml`
publishes it with GitHub Pages (`https://ersa-ecpx.github.io/ecp-mkosi/`) when `docs/` changes on `main`. **It has not run:** it
needs Settings > Pages > Source "GitHub Actions" once, and the workflow pushed. A second copy is on the packages site
(`https://pkgs-compute.ersa.dev/wiki/`), built by the workflow in the `packages` repository when a release is published, so the
two can differ.

## Limits

- Image builds need a machine with root or user namespaces.
- `.github/workflows/build-x86_64.yml` is manual-only and stale (written for the Rocky base); it does not build current images.
- **Intermittent 90-second boot stop in QEMU (cause found; fix not confirmed by repeated boots).** About half of boots stalled
  for 90 seconds into emergency mode. A drop-in ordering `systemd-sysusers.service` after `systemd-sysext.service` caused an
  ordering cycle that systemd broke by deleting a job at random. Found with
  `scripts/qemu-uefi-test --cmdline "systemd.log_level=debug systemd.log_target=kmsg systemd.log_ratelimit_kmsg=0"` (look for
  `Found ordering cycle`). The drop-in is gone and `ersa-sysusers-extensions.service` replaces it; boot a rebuilt image several
  times to confirm.

See also: [installer medium](live-medium.md), [developer guide](../../foundation/README.md#testing).

## Tested

The build scripts and unit tests are used daily; images are built by hand (mkosi needs root or user namespaces and about 15 GiB), and the unit tests run without them. Nothing has been tested on real hardware, and there is no CI for images.

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

See also: [installer medium](live-medium.md), [developer guide](../../foundation/README.md#testing).

## Publishing the wiki (GitHub Pages)

`scripts/build-wiki docs/wiki/index.html --cname compute.ersa.dev` writes a folder holding `index.html`, `assets/`
(the screenshots), `.nojekyll` and `CNAME`. Publish the contents of `docs/wiki/` as the root of a Pages site (a
`gh-pages` branch or a Pages workflow) and point the domain's DNS at GitHub Pages. Nothing in the page needs a server.

Screenshots in `docs/images/` are real: installer windows are captured from the application in a headless session,
and terminal images are output captured from a booted installer image in QEMU, rendered as pictures.

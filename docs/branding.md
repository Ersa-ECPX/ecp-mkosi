# Ersa branding sources

## Logo sources

`branding/ersa-mark-light.svg` (dark mark, for light backgrounds) and `branding/ersa-mark-dark.svg` (light mark, for dark
backgrounds) are the only logo sources. `scripts/build-branding` renders them, and the wallpapers
(`branding/wallpaper-{light,dark}.svg`, 3840x2160), with `rsvg-convert` into `mkosi.branding/` (not committed), which mkosi
overlays on the image. `scripts/build-splash-package` renders the Plymouth PNGs (1x and 2x), because Plymouth cannot load SVG.

## What GNOME reads

| Where | How GNOME picks it | What we provide |
| --- | --- | --- |
| Settings, About: logo | `LOGO=` in `os-release`, looked up as `<LOGO>-text-dark`, `<LOGO>-text`, `<LOGO>-dark`, `<LOGO>` (dark names first in the dark style) | `LOGO="ersa-logo"`; icons `ersa-logo` and `ersa-logo-dark` in `hicolor` (SVG and PNG) |
| Settings, About: name | `PRETTY_NAME`; `IMAGE_VERSION` separately | `PRETTY_NAME="Ersa Compute Platform"`; mkosi adds `IMAGE_VERSION` |
| Initial Setup welcome | `PRETTY_NAME` | the same |
| GDM logo | setting `org.gnome.login-screen logo` | `10_ersa-branding.gschema.override` pointing at `/usr/share/pixmaps/ersa-gdm-logo.png` (light mark), compiled by `mkosi.postinst.chroot` |
| Wallpaper | gschema default and `gnome-background-properties` | `90_ersa-wallpaper.gschema.override`; files in `/usr/share/backgrounds/ersa/`, listed by `ersa.xml` |

Read from `gnome-control-center` and `gnome-initial-setup` upstream sources, not from a running system.

`mkosi.extra/usr/lib/os-release` overrides openSUSE's. It keeps `ID="opensuse-tumbleweed"` and `ID_LIKE="opensuse suse"` for
tool compatibility, has no `VERSION_ID` (Tumbleweed has no stable one), and records the base in `ECP_BASE`, which
`foundation status` prints. No Ersa home or bug-report URL exists yet. The `-upstream` (neutral) openSUSE branding packages
are installed for GDM and Plymouth.

## Boot splash

The theme is packaged by `packages/ersa-boot-splash/ersa-boot-splash.spec`, installed in the root image and the initrd, and
selected with `Theme=ersa` in both. It was copied on 2026-11-03 from an earlier installed `ersa` theme; its original packaging
declared the licence `unknown`, so confirm it before publishing images or RPMs. The Plymouth program and plugin are
distribution packages.

The package carries two themes: `ersa` (light, default) and `ersa-dark` (dark background, light mark, `fill-dark.png` and
`track-dark.png`). The two scripts differ only between `# THEME VARIABLES` and `# END THEME VARIABLES`;
`foundation/tests/test_splash.py` fails if they drift.

The choice is not in the kernel image (a signed image cannot be edited, and Secure Boot needs the signature):

- **UEFI variable `ErsaSplashDark-<GUID>`:** present means dark. The initrd unit `ersa-splash-dark.service`
  (`mkosi.initrd.extra/`) runs before `plymouth-start.service`, only when the variable exists, and links the initrd's
  `/etc/plymouth/plymouthd.conf` to `plymouthd-dark.conf` through one tmpfiles line.
- **`/etc/plymouth/plymouthd.conf`** of the installed system: foundation edits its `Theme=` line, for the shutdown and restart
  splash when the initrd is gone.

`foundation splash light|dark` (D-Bus `Boot.SetSplash`, polkit action `org.ersa.foundation.splash`, allowed without a password
for the active local user) creates or removes the variable and edits the line. On a computer that did not start with UEFI it
writes the file and says the boot splash cannot follow. `foundation-splash-sync.service` (user service, enabled by preset)
follows `org.gnome.desktop.interface color-scheme` (`prefer-dark` is dark) at login and when it changes. An update changes
nothing: the variable and `/etc` belong to the machine, not to a slot. The splash changes at the next boot.

On OVMF in QEMU (two boots sharing one variable store) the variable was created (attributes `0x7`, value `01`), survived a
power cycle, and a `ConditionPathExists` unit was skipped when it was absent and ran when present; the tmpfiles line linked
the dark configuration and `light` removed the variable (efivarfs marks it immutable; the code clears that first). Not tested:
the initrd unit itself (the tested image predates it), `systemd-tmpfiles` in the initrd, the dark theme showing, real firmware,
the user service against a real GNOME session, polkit with the real policy, and a visible splash (Plymouth's log alone does not
prove one rendered).

## Fonts

Inter (`inter-fonts`) is the interface, document and title-bar font, set by `20_ersa-fonts.gschema.override`. Monospace is IBM
Plex Mono (`ibm-plex-mono-fonts`; JetBrains Mono is not in Tumbleweed's main repository) with Adwaita Mono as fallback.
Applications that name a font and scripts Inter lacks use Noto through fontconfig ([multimedia and fonts](features/multimedia-and-fonts.md)).

The "ersa.compute" wordmark is Inter Tight ("ersa" bold, ".compute" light), converted to outlines by `scripts/make-wordmark`
into `foundation/foundation_deck/ersa-wordmark-{light,dark}.svg`, since Inter Tight is not packaged and a logo should not
depend on installed fonts. Inter Tight is under the SIL OFL 1.1 (`docs/licenses/INTER-TIGHT-OFL.txt`).

## Tested

The assets, scripts and package exist, and the unit tests check the splash configuration and backend, the daemon and the CLI on a private bus. The UEFI variable behind the dark splash was tested on OVMF in QEMU (`scripts/qemu-uefi-test`). Nothing here has been seen on a booted desktop, and the initrd unit has not been booted. For the overview see [branding](features/branding.md).

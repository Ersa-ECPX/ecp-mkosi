# Ersa branding sources

The Plymouth theme sources are packaged by `packages/ersa-boot-splash/ersa-boot-splash.spec`; the corresponding source is in `~/Dev/eap/ersa-packages/ersa-boot-splash/` (local version 0.1.0). The theme was copied on 2026-11-03 from this user's installed `/usr/share/plymouth/themes/ersa/`. EAP uses a separate Yocto PSplash asset; no Yocto recipe or binary package is imported here.

The theme has a script, progress-bar images, and an Ersa mark. Its Arch PKGBUILD declares `license=('unknown')`, and EAP's boot splash document says the artwork license has not been declared. The user authorized reuse for this ECP workspace. Confirm the artwork license before publishing redistributed ECP images or RPMs. The Plymouth program and plugin remain distribution packages.

The `ersa-boot-splash` RPM is included in the root image and default initrd. Mkosi supplies the `Theme=ersa` Plymouth selection in both images. A VM graphical boot is still needed to confirm the theme appears, scales correctly, and handles a failed boot or password prompt. Plymouth's service-start log alone does not prove that a visible splash was rendered; if the guest display stays blank, investigate the virtual display/console and graphical target separately.

## Light and dark boot splash

The package carries two Plymouth themes: `ersa` (light, the default) and `ersa-dark` (dark background, the light mark from `branding/ersa-mark-dark.svg`, `fill-dark.png`/`track-dark.png`). `ersa.script` and `ersa-dark.script` are identical except for the block between `# THEME VARIABLES` and `# END THEME VARIABLES`; `foundation/tests/test_splash.py` fails if they drift. Plymouth selects the dark theme from the kernel command line token `plymouth.splash=ersa-dark` (light has no token), so no initrd change is needed: the initrd already installs the whole package.

- `foundation splash light|dark` (D-Bus `Boot.SetSplash`, polkit action `org.ersa.foundation.splash`, allowed without a password for the active local user) adds or removes that token in every unified kernel image on the ESP, in place and atomically, keeping file names (and so boot counters). It prints which files changed or were skipped.
- `foundation-splash-sync.service` (a systemd user service, enabled by preset) follows the desktop: it reads `org.gnome.desktop.interface color-scheme` (`prefer-dark` is dark, anything else light), calls `SetSplash` once at login and again only when the value changes.
- An OTA-installed slot carries the running system's `plymouth.splash=` token over (`updates.update_cmdline`), so an update does not reset the choice.

Limits: the splash changes at the next boot, because the kernel command line is read at boot. A signed UKI cannot be edited without breaking its signature; it is reported as skipped. If the command line section has no room for the token the call fails and names the file. What has been tested: unit tests with fake ESP/UKI files, and the daemon and the CLI on a private bus. Not tested: the user service against a real GNOME session and system bus, polkit with the real policy, and a boot in which Plymouth actually shows the dark theme (neither theme script was run by Plymouth for this change, and the dark logo PNGs were not rendered because the RPM was not built).

## Logo variants and where they appear

`branding/ersa-mark-light.svg` (dark mark, for light backgrounds) and `branding/ersa-mark-dark.svg` (light mark, for dark backgrounds) are the only logo sources. `scripts/build-branding` renders them with `rsvg-convert` into `mkosi.branding/` (not committed), which mkosi overlays on the image. `scripts/build-splash-package` renders the Plymouth logo PNGs (1x and 2x) from the light SVG (the `ersa-dark` ones from the dark SVG), because Plymouth cannot load SVG.

GNOME's own code (read from `GNOME/gnome-control-center` and `GNOME/gnome-initial-setup` on GitHub, which is what Tumbleweed ships) decides what shows where:

| Where | How GNOME picks it | What ECP provides |
| --- | --- | --- |
| Settings, About: logo | `LOGO=` in `os-release`, looked up in the icon theme as `<LOGO>-text-dark`, `<LOGO>-text`, `<LOGO>-dark`, then `<LOGO>` (the dark names first in the dark style) | `LOGO="ersa-logo"`; icons `ersa-logo` (light mark) and `ersa-logo-dark` (dark mark) in `hicolor`, as SVG and PNG |
| Settings, About: OS name | `PRETTY_NAME`; `IMAGE_VERSION` is shown separately | `PRETTY_NAME="Ersa Compute Platform"`; mkosi adds `IMAGE_VERSION` |
| Initial Setup welcome page | `PRETTY_NAME` | the same |
| GDM login screen logo | the `org.gnome.login-screen logo` setting | `10_ersa-branding.gschema.override` pointing at `/usr/share/pixmaps/ersa-gdm-logo.png` (light mark, for the dark GDM style), compiled by `mkosi.postinst.chroot` |

`mkosi.extra/usr/lib/os-release` overrides openSUSE's file. It keeps `ID="opensuse-tumbleweed"` and `ID_LIKE="opensuse suse"` for tool compatibility, has no `VERSION_ID` (Tumbleweed has none that is stable), and records the base in `ECP_BASE`, which `foundation status` prints. No Ersa home or bug-report URL exists yet, so none is set. The `-upstream` (neutral) openSUSE branding packages are installed for GDM and Plymouth instead of the openSUSE-branded ones.

Wallpaper: `branding/wallpaper-light.svg` and `wallpaper-dark.svg` (the mark on a soft gradient) are rendered at 3840x2160 by `scripts/build-branding` into `/usr/share/backgrounds/ersa/`, listed for Settings through `gnome-background-properties/ersa.xml`, and set as the default (light, dark and lock screen) by `90_ersa-wallpaper.gschema.override`. Not yet seen in a booted image.

GNOME's own wallpapers (`gnome-backgrounds`) are installed beside them.

Not yet replaced: other openSUSE logo files, and `/etc/os-release` compatibility files. None of this has been visually verified in a booted image. This replaces the earlier Rocky scheme, which patched `fedora-logo*` file names.

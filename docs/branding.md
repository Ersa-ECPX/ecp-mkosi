# Ersa branding sources

The Plymouth theme sources are packaged by `packages/ersa-boot-splash/ersa-boot-splash.spec`; the corresponding source is in `~/Dev/eap/ersa-packages/ersa-boot-splash/` (local version 0.1.0). The theme was copied on 2026-11-03 from this user's installed `/usr/share/plymouth/themes/ersa/`. EAP uses a separate Yocto PSplash asset; no Yocto recipe or binary package is imported here.

The theme has a script, progress-bar images, and an Ersa mark. Its Arch PKGBUILD declares `license=('unknown')`, and EAP's boot splash document says the artwork license has not been declared. The user authorized reuse for this ECP workspace. Confirm the artwork license before publishing redistributed ECP images or RPMs. The Plymouth program and plugin remain distribution packages.

The `ersa-boot-splash` RPM is included in the root image and default initrd. Mkosi supplies the `Theme=ersa` Plymouth selection in both images. A VM graphical boot is still needed to confirm the theme appears, scales correctly, and handles a failed boot or password prompt. Plymouth's service-start log alone does not prove that a visible splash was rendered; if the guest display stays blank, investigate the virtual display/console and graphical target separately.

## Logo variants and where they appear

`branding/ersa-mark-light.svg` (dark mark, for light backgrounds) and `branding/ersa-mark-dark.svg` (light mark, for dark backgrounds) are the only logo sources. `scripts/build-branding` renders them with `rsvg-convert` into `mkosi.branding/` (not committed), which mkosi overlays on the image. `scripts/build-splash-package` renders the Plymouth logo PNGs (1x and 2x) from the light SVG, because Plymouth cannot load SVG.

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

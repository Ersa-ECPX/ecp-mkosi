# Branding

**Status:** partly built. The logos, os-release identity, boot splash, GDM login logo setting, wallpapers and the
installer's identity are in the repository; the GDM logo and the wallpapers have **not yet been seen on a running
desktop**. The Plymouth artwork's licence is undeclared.

## Where the Ersa identity appears

| Where | What | Source |
| --- | --- | --- |
| Settings, About | the logo, the name "Ersa Compute Platform", `IMAGE_VERSION` | `LOGO=ersa-logo` in `os-release`; icons `ersa-logo` (dark mark) and `ersa-logo-dark` |
| Boot | the Plymouth theme `ersa` | package `ersa-boot-splash` (logo PNGs rendered from the SVG) |
| Login screen (GDM) | the light mark | `10_ersa-branding.gschema.override` -> `/usr/share/pixmaps/ersa-gdm-logo.png` |
| Desktop and lock screen | light and dark wallpapers | `branding/wallpaper-*.svg`, rendered at 3840x2160; `90_ersa-wallpaper.gschema.override` |
| Installer | the mark on the welcome page, the app icon | `foundation_installer/*.svg`, `org.ersa.Installer.svg` |
| Initial Setup | the name | `PRETTY_NAME` |

The two logo sources are `branding/ersa-mark-light.svg` (dark mark, for light backgrounds) and `ersa-mark-dark.svg` (light
mark, for dark ones). `scripts/build-branding` renders everything from them into `mkosi.branding/` (not committed), which
mkosi overlays on the image; `scripts/build-splash-package` renders the Plymouth PNGs.

## Naming rule

`ECP` is for code only (variables, labels, `IMAGE_ID`). Anything a person reads (CLI output, prompts, banners, installer
text, package descriptions) says "Ersa Compute Platform", never "ECP OS". A test keeps the acronym out of the installer's text.

## Wallpapers

Two generated wallpapers, light and dark, are the default for the light style, the dark style and the lock screen, and
are listed in Settings under "Ersa". They are made of soft colour fields (radial gradients: deep green with teal and
a blue corner in the dark one; mint with a peach hint and sky blue in the light one), faint rings echoing the circle of
the mark, and a small mark in the middle. GNOME's own wallpapers (the `gnome-backgrounds` package, about 25 of them) are
installed too and appear in Settings beside them, so there is plenty to choose from. An earlier version, a flat gradient
with a large mark, looked dull and is gone.

Photographic wallpapers from Unsplash were considered and not used: Unsplash's licence allows free use and modification
but says nothing about bundling photos in an operating system, and picking good photos is a matter of taste. They could be
added later, with credit to each photographer, once someone has chosen them.

## Limits and open items

- The Plymouth artwork was copied from an earlier install; its licence has to be confirmed before publishing images.
- The default accent colour is libadwaita's blue, not an Ersa colour.
- Not replaced: other openSUSE logo files and compatibility `os-release` files.
- A booted check of Settings, the login screen and the wallpapers is still needed.

See also: [docs/branding.md](../branding.md) (sources and details).

## In Settings

![Settings, About](../images/desktop-about.webp)

*GNOME Settings, About page, captured from the installer image booted in QEMU.*

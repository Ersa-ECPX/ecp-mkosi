# Ersa branding sources

The Plymouth theme in `mkosi.extra/usr/share/plymouth/themes/ersa/` was copied on 2026-11-03 from this user's installed `/usr/share/plymouth/themes/ersa/`. The corresponding source is in `~/Dev/eap/ersa-packages/ersa-boot-splash/` (local version 0.1.0). EAP uses a separate Yocto PSplash asset; no Yocto recipe or binary package is imported here.

The theme has a script, progress-bar images, and an Ersa mark. Its Arch PKGBUILD declares `license=('unknown')`, and EAP's boot splash document says the artwork license has not been declared. The user authorized reuse for this ECP workspace. Confirm the artwork license before publishing redistributed ECP images or RPMs. The Plymouth program and plugin remain distribution packages.

`mkosi.initrd.conf` copies the theme and Plymouth selection into the default initrd. A VM graphical boot is still needed to confirm the theme appears, scales correctly, and handles a failed boot or password prompt.

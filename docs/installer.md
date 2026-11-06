# Installer and `foundation deploy`

Status: design proposal. Nothing here is implemented or tested yet. Items marked **unverified** must be
checked (in QEMU, on virtual disks only) before they are described as working.

## Goals

- Put a complete ECP system on a disk: the A/B layout from `mkosi.repart/`, a bootable ESP, and fresh
  persistent partitions.
- Work from a live session (graphical installer) and from an installed system (provisioning,
  re-deploying, recovery), using one backend.
- Keep policy in `foundation`, exposed through a D-Bus API served by the `foundationd` system
  service. The CLI, the graphical installer, the recovery environment and a future settings applet
  are all clients of that API. No client implements deployment logic or shells out to another
  client.
- Installed systems carry no live-session configuration; the installer is a sysext on the installer medium only.

## Installer media layout (decided 2026-11-05)

The installer is an mkosi **profile** of the same OS, not a separate distribution: `scripts/build x86-64
--profile installer`. It uses a minimal layout, defined in `mkosi.repart.installer/`: a 512 MiB ESP, **one**
5 GiB root slot (nothing on the stick is updated, so there is no slot B and no recovery partition), and a
512 MiB btrfs data volume for the live session's `/var` (including the `/etc` overlay's upper layer) and
`/home`. About 6 GiB in total, so it fits an 8 GB stick; a 28 GiB image would not, and a sparse file is
still written out in full by `dd`. The normal image (A/B slots) remains the one for QEMU testing. The
profile is the layout only so far: the installer app and `Deploy` are not built.

## Architecture

The daemon and its full API are designed in `foundationd.md`; the deploy API below is one interface
of it. This section is the installer's view.

```
foundation (CLI) ─┐
installer app ────┼──► system bus ──► foundationd (root, polkit-gated) ──► systemd-repart, ...
recovery UI ──────┘     org.ersa.Foundation1
```

- `foundationd` is a system service, started on demand by D-Bus activation. It runs as root and
  authorizes every method call with polkit before acting.
- The CLI is a thin client: it parses arguments, calls the D-Bus API and renders the result. With
  `--json` it prints the daemon's signals as JSON lines for scripting.
- The interface is defined by an introspection XML file
  (`/usr/share/dbus-1/interfaces/org.ersa.Foundation1.xml`). The XML is the contract that the CLI
  and the GUI are built against, and what the GUI mocks in tests.
- Implementation uses GLib's GDBus through PyGObject (already in the GNOME image), so no extra
  D-Bus library is needed. **Unverified:** how this affects the current pure-standard-library
  packaging of `foundation`; the CLI and the daemon may need separate dependency sets.

## Commands (implemented, 2026-11-05)

```
foundation deploy list [--json]
foundation deploy install DISK [--language LOCALE] [--timezone ZONE] [--dry-run] [--yes] [--no-wait] [--json]
```

- `deploy list` shows each whole disk and whether it can be installed to (and why not).
- `deploy install` prints the plan (partitions, sizes, existing partitions that will be erased), then
  needs the disk path typed back (`--yes` skips the prompt). `--dry-run` changes nothing.
- It is called `deploy`, not `install`, because installing packages is `foundation apps` / `compat`.
- Not implemented yet: `--image FILE`, `--user`/password options (see below).

### Unattended user

- `--user NAME` creates the account on the target (member of `wheel`, home on the new `/home`).
- The password is never accepted as a plain `--password VALUE` argument: it would be visible in
  `ps`, `/proc/*/cmdline` and shell history. Use `--password-file`, `--password-stdin` or
  `--password-hash` (a crypt(3) hash). Over D-Bus the password is never a string argument: the
  client passes a file descriptor (D-Bus type `h`) or a hash, so it cannot appear in logs, bus
  monitoring output or object properties.
- With `--user` and a password, first-boot GNOME Initial Setup is skipped for that account.
  Without `--user`, Initial Setup creates the account at first boot (current behaviour).
- **Unverified:** the exact way to skip Initial Setup, and SELinux labelling of files written
  from outside the target (a relabel on first boot is the fallback).

### Installer media (open decision)

Writing an *installer USB* is a different operation from installing an OS: `--user`, `--password*`
and the first-boot behaviour do not apply. Two shapes were discussed:

- `foundation deploy --installer --disk /dev/X` (one verb, mutually exclusive flags), or
- `foundation media create --installer --disk /dev/X` (separate noun).

Either way the installer payload (live configuration and installer sysext) is not present on
installed systems, so the source must be `--image installer.img` or a download. That depends on the
OTA work and is out of scope for the first milestone.

## Safety rules (enforced in `backends/deploy.py`, tested)

- The target must be an absolute path to a whole block device the kernel knows. Partitions, plain
  files and paths with `..` are refused. A plain file is accepted only by a daemon started with
  `--allow-file-targets` on the session bus (tests); the system bus refuses that flag.
- Refused: the disk that holds the running root, any disk with a mounted partition, a read-only
  disk, a disk smaller than the layout needs (about 28 GiB), a running slot with no kernel image,
  a version too long for a partition name.
- The caller must pass the disk path again as `confirm`; the CLI also makes people type it.
- Polkit action `org.ersa.foundation.deploy` always asks for an administrator (no remembered answer).
- One deployment at a time (job lock `disk`).
- Nothing is changed before the job's commit point; staging lives under the daemon's state directory
  and is removed afterwards.

## Deployment steps (implemented)

1. Validate, then stage: copy the running ESP, with the running slot's kernel image renamed
   `ecp_<v>+3.efi.partial` (systemd-boot ignores it), and build the data-volume seed (the `/etc`
   overlay directories labelled `etc_t`, an empty `/home`). Read the root image's real length.
2. Commit (point of no return), then `systemd-repart --empty=force --offline=yes` with definitions
   generated from `backends/layout.py`: ESP 1 GiB, recovery 1 GiB (reserved, empty), root A 8 GiB
   (`ecp-root_<v>`), root B 8 GiB (`_empty`), `ECP-DATA` btrfs (`var`, `home`) taking the rest.
3. Copy the root image into slot A, read both back from the disk and compare SHA-256.
4. Rename the kernel image to `ecp_<v>+3.efi` with `mren` (mtools, no mount). Only now can the disk boot, so an
   interrupted or failed install never looks bootable.
5. `fsync`; the result lists the partitions.

`layout.py` is the single source for sizes and labels; a test compares it with `mkosi.repart.system/`.
The ESP also carries the fallback boot path from the running ESP; no firmware boot entry is written
(v1), so firmware must pick the disk through the fallback path.

**Tested:** real `systemd-repart` and mtools on sparse image files (layout, root bytes identical,
kernel image in place, corrupt copy leaves only the `.partial` file, staging removed), refusal rules,
and the daemon plus CLI end to end on a file target. **Tested by hand (2026-11-06, QEMU):** booted the installer image, ran `foundation deploy install
/dev/vdb` onto a 40 GiB virtual disk, then booted that disk alone with fresh firmware variables. It
booted and the slots were as expected; the user reported everything worked. The individual checks
(SELinux label of `/etc`, data-volume mounts, whether `liveuser` leaked into the installed system)
were not itemised, so they are not recorded as verified.
**Not tested:** a real physical disk or USB stick, a disk with non-512-byte sectors, Secure Boot, and
an automated install-then-boot check (planned for `scripts/qemu-smoke`).

## D-Bus API (`org.ersa.Foundation1.Deploy`, in the contract XML)

- `ListDisks() -> aa{sv}`: path, name, size, model, removable, boot_medium, mounted, usable, reason.
- `PlanDeploy(s disk) -> a{sv}`: disk, size, version, partitions, will_erase, problems, possible.
- `Deploy(s disk, s confirm, s language, s timezone) -> o`: a job; result disk, version, root_partition, bytes,
  partitions. `language` (a locale such as `de_DE.UTF-8`) and `timezone` (such as `Europe/Berlin`) are written into
  the installed system's `/etc` (the overlay's upper layer on the data volume) as `locale.conf` and the `localtime`
  link, labelled `locale_t`; empty means unset. An unknown time zone is refused before anything is written.

Still planned: unattended user setup and the live-medium polkit rule that lets the live user run
deployments without a password.

## Graphical installer (`ersa-installer`, package `foundation-installer`)

A GTK 4 and libadwaita window in the style of the macOS *operating system* installer ("Install macOS"),
not the `.pkg` installer: no sidebar of steps, one centred page at a time, Go Back and Continue at the
bottom right. Pages: a welcome page; "Select your language" and "Select your time zone" (two separate screens,
each a searchable list; the zone defaults to the live system's); "Select the disk where you want to install" with large selectable
disk icons (disks that cannot be used are dimmed with the reason); Install, which asks the service for a
plan and then shows an erase confirmation dialog (a disk the service refuses shows the reasons instead);
a progress page with a thin bar, the current step and an estimated time remaining; and a last page with
Restart. A failed install shows the reason and a "Show Details" log.

It is only a client of foundationd's `Deploy` interface (`ListDisks`, `PlanDeploy`, `Deploy`) and
holds no policy: whether a disk is allowed comes from the service. It uses Gio's asynchronous calls
and job signals so the window never blocks. The window cannot be closed while a write is running.

On the installer medium the live session starts it automatically
(`ersa-live-session.service` writes the autostart), signed in as `liveuser`, and a polkit
rule lets `liveuser` run deploy, jobs and reboot without a password.

**Tested:** the text helpers, and the real window driven against a real foundationd on a private bus
(welcome, the disk list marking the running disk unusable, review, a full install onto a file target,
a refused disk showing the failure page, no sidebar, closing blocked during install). The window tests
open real windows, so they run only when asked, on a virtual display:
`env -u WAYLAND_DISPLAY GDK_BACKEND=x11 GSK_RENDERER=cairo ERSA_GUI_TESTS=1 xvfb-run -a python3 -m unittest discover -s tests -p test_installer.py`.
**Not tested:** that the installed system shows the chosen language and zone after boot (the seed files are tested,
not a boot), that the first-boot setup does not ask again, translation of the installer's own text, and the look on the real live session (fonts, dark/light, HiDPI), the autostart and the
polkit rule in the booted installer image, and keyboard navigation and screen readers.

## Live session

- One root image: the installer medium and installed systems use the same root (the installer
  profile only changes the partition layout and output name). The installer itself is a **system
  extension (sysext)**: `scripts/build-installer-sysext` turns the `foundation-installer` RPM into
  `ersa-installer.raw` (EROFS, with `extension-release.ersa-installer` for `ID=opensuse-tumbleweed`),
  and the installer profile puts it on the medium's **data volume** at `/var/lib/extensions/`
  (`ExtraTrees=mkosi.installer.extra`), where `systemd-sysext.service` (enabled in the preset) merges it
  into `/usr` at boot. It carries the app, the hidden launcher and `ersa-live-session.service`, which
  creates `liveuser` (password `liveuser`), passwordless sudo (a one-line drop-in for `systemd-sysext.service` in the root image starts it right after the merge, because a unit merged mid-boot is not in the boot's job list; checked on four boots in a row), the polkit rule, GDM automatic login (SUSE's gdm reads `DISPLAYMANAGER_AUTOLOGIN` from `/etc/sysconfig/displaymanager`, empty in the image, ahead of `custom.conf`, so both are set) and
  the installer autostart under `/etc` and `/home` (the medium's overlay and data volume, never the root image).
- Why not the ESP: systemd-stub's `/loader/extensions/` and `/.extra/sysext` are for extending the
  *initrd* (merged there, unsigned images rejected by the stricter policy, gone after switch-root), and
  the main system's `systemd-sysext.service` only looks in `/etc`, `/run` and `/var/lib/extensions`.
  (Found 2026-11-06 by booting: "no extensions found".)
- SELinux: a sysext keeps the labels stored in its image, and the overlay shows the sysext's directories
  on top of the real ones, so an unlabelled sysext turned `/usr` into `unlabeled_t` and dbus-broker, logind
  and GDM were denied (black screen, found 2026-11-06 by booting the image in QEMU with the kernel and
  initrd taken from the UKI). `scripts/build-installer-sysext` therefore writes every path's label, from
  the policy's `file_contexts` (the selinux-policy-targeted package in mkosi's cache), into the EROFS as
  tar xattrs. The labels were checked in the image's bytes, not yet in a boot.
- Installed systems never get any of it: the root is the production root and Deploy writes a fresh data
  volume. The base image needs `python3-gobject` for the app (added).
- Build order: `scripts/build-foundation-package`, then `scripts/build x86-64 --profile installer`
  (which runs `scripts/build-installer-sysext` first).
- **History:** the first installer image built `liveuser` into the root, so installs from it carried
  it; the sysext (2026-11-06) replaces that. **Not yet verified in a booted image:** that
  `systemd-sysext` merges it (the first attempt, on the ESP, did not), that the `extension-release`
  matches, that the live session starts before GDM, and `getent passwd liveuser` being empty after
  installing. The script was checked by extracting the sysext only.
- Initial Setup handles the user on the installed system.
- Ventoy compatibility is a separate spike; a `dd`-able image comes first.

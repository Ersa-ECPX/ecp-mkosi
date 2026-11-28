# Scripts

Everything in `scripts/`, what it does, how to run it and what it needs. Run all of them from the repository root. This page
is written from the scripts' own header comments; where a script says it was not run, so does this page. The first section is
the answer to the question people ask most.

## Serving an update to a test machine

An update is one archive (`ecp_<version>-x86_64.tar.zst`) plus a small feed file (`latest-x86_64.json`) that says it exists.

1. **Build the root image with a version higher than the installed one** (an older or equal version is refused without `--force`):

   ```
   scripts/build x86-64 --profile root --image-version 1.0.3
   ```

2. **Pack it:**

   ```
   scripts/make-update-archive 1.0.3             # signed with the keys of scripts/security-dir
   scripts/make-update-archive 1.0.3 --no-sign   # unsigned
   ```

   This writes `dist/update/ecp_1.0.3-x86_64.tar.zst` and `dist/update/latest-x86_64.json`.

3. **Serve the folder** on the host's loopback (a QEMU guest on the default network sees it as `10.0.2.2`):

   ```
   scripts/serve-update dist/update 8000
   ```

4. **In the guest**, either install the archive directly:

   ```
   foundation update install http://10.0.2.2:8000/ecp_1.0.3-x86_64.tar.zst
   foundation reboot
   ```

   or point the updater at the feed and let it find the update (the update window then offers it):

   ```
   foundation config set ota.url http://10.0.2.2:8000
   foundation update check
   ```

**Signed or not.** A signed archive installs only if the system trusts the signing key. A build made with the public test keys
(the default, see `security-dir`) trusts the public test update key, so an archive signed with those installs without any flag. An
unsigned archive (`--no-sign`) needs `foundation config set system.developer_mode true` and `--unsigned`. Plain `http` is
accepted only from this machine or a private network; anything on the internet must be `https`.

**For real use** the archive goes to a GitHub release and the feed is published with `scripts/publish-feed` (see below, and
[ota-hosting](ota-hosting.md)). `serve-update` has no TLS, no authentication and no range requests: it is for tests.

## Building

| Script | What it does |
| --- | --- |
| `build ARCH [mkosi args]` | The entry point: `scripts/build x86-64 [--profile root\|installer\|...]`. Needs mkosi 27.1 exactly. Runs `prepare-build`, picks the signing keys (`use-security-keys`) and calls mkosi. Scratch space is `ECP_WORK_DIR` (default `/tmp/ecp-work`, tmpfs); a build needs about 15 GiB free there. For the `installer` profile it also builds the installer extension and stages the operating system the medium installs. `ARCH` is `x86-64` or `arm64`. |
| `build-podman ARCH` | The same build inside a pinned Fedora 45 container (`Containerfile.build`), for a machine without mkosi. The preparation steps still run on the host. Needs podman. |
| `prepare-build` | Everything that must exist before mkosi runs: clones missing sources, renders logos, builds the splash and foundation RPMs, and builds the compiled RPMs when missing or when a spec's pinned commit changed. Called by `build` and `build-podman`. |
| `fetch-sources` | Clones `foundation/`, `packages/` and the sources of the compiled packages (`.sources/`), so a fresh clone can build with no other setup. Existing checkouts are left alone unless `ECP_UPDATE=1`. `ECP_GIT_BASE` changes where it clones from. |
| `build-branding` | Renders the logos from `branding/*.svg` into `mkosi.branding/`, which mkosi overlays on the image (nothing in it is committed). Needs `rsvg-convert`. If an earlier sudo build left the folder owned by root it tells you to remove it. |
| `build-splash-package` | Builds the Ersa Plymouth boot-splash RPM into `mkosi.packages/`, rendering the logo PNGs from the branding SVG. Needs `rpmbuild` and `rsvg-convert`. |
| `build-foundation-package` | Builds the pure-Python `foundation`, `foundationd`, `foundation-installer` and `foundation-deck` RPMs from the working tree's `foundation/` (not from git: uncommitted changes are included). Runs the spec's tests; set `ECP_NOCHECK=1` to skip them. Needs `rpmbuild`. |
| `build-rpms [PACKAGE]` | Builds the compiled RPMs (`ersa-bluetooth`, `ersa-dialer`) inside an openSUSE Tumbleweed container, never on the host (a host-linked binary would not run in the image). Each is built from the commit pinned in its spec, with `git archive`, so uncommitted changes are ignored. Source checkouts: `ERSA_BLUETOOTH_SRC`, `ERSA_DIALER_SRC` (default `~/Dev/...`, else `.sources/`). x86-64 only. |
| `build-installer-sysext [ARCH]` | Builds the installer as a system extension from the `foundation-installer` RPM and stages it for the installer profile. Every file is labelled for SELinux for its merged path. Needs the RPM (`build-foundation-package`), `rpm2cpio`, `cpio`, `mkfs.erofs` and `python3`. Run by `build`. |
| `stage-payload VERSION` | Stages the operating system for the installer medium: the update archive of the normal image, copied to `mkosi.installer.extra/usr/share/ersa/payload/`. Needs the `root` profile built first. Signs with `ECP_SIGN_KEY` when it is set. Run by `build` for the installer profile. |
| `clean [--all] [-n] [-y] [--force]` | Removes build outputs (`mkosi.output`), generated branding and the scratch and cache folders. `--all` also removes built RPMs, the tools tree and package caches (the next build is much slower). `-n` lists what would go and removes nothing. Only fixed paths are ever removed. Refuses while a build seems to be running unless `--force`. |
| `install-raw --erase IMAGE.raw /dev/DISK` | Writes an image to a disk with `dd`. Run as root. **Erases the disk.** Refuses a mounted target or one smaller than the image. First-boot setup is then done on the target's console. |

## Signing keys

There are three separate keys: Secure Boot (signs the boot loader and kernel image), the image-update key (signs update
archives) and the package repository key. Compromise of one must not give another. The public **test** keys in
`build/security/testkeys` let anyone sign, so anything made with them is for testing only and a release refuses them.

| Script | What it does |
| --- | --- |
| `security-dir [--kind]` | Prints which key folder this build uses, or with `--kind` whether it is `test` or `release`. Order: `ECP_SECURITY_DIR`, then `build/security/release` if it exists, then the public test keys. |
| `use-security-keys` | Points the build at those keys: links `secure-boot/` to them, and for a test-keys build installs the public test update key in the image so test updates verify. Refuses the test keys when `ECP_REQUIRE_SB_KEY=1` (a release). Run by `build`. |
| `make-secure-boot-key [DIR]` | Makes the Secure Boot key and certificate (default `./secure-boot`): `ersa-secure-boot.key` (secret), `.crt` and `.cer` (what firmware menus ask for). Valid 10 years. Refuses to replace an existing key. Back the key up off this machine. |
| `make-image-signing-key [DIR]` | Makes the GPG key that signs update archives (default `./image-signing-key`, keep it out of git): the public key, the secret key, its passphrase and a ready keyring. Refuses an existing folder. The key does not expire, because an expired signer would strand every installed system; replace it by shipping the new public key in an update first. Nothing is uploaded. |

## Updates and releases

| Script | What it does |
| --- | --- |
| `make-update-archive VERSION [--no-sign] [--base-url URL]` | Packs the `root` profile's output into the archive `foundation update install` reads, and writes the feed beside it. Signs with the keys of `security-dir`, or with `ECP_SIGN_KEY` for a key in your own GPG keyring. `--base-url` writes the feed's file as an absolute address instead of a name next to the feed. Reads `mkosi.output/root/`; writes `dist/update/`. |
| `serve-update [DIR] [PORT]` | Serves a folder over plain HTTP on the loopback for a test (defaults `dist`, `8000`). Prints the command to run in the guest. See the first section. |
| `release VERSION [--draft] [--no-build] [--sign]` | Builds both images and publishes a GitHub release of this repository: the installer medium (split into `.partNN` files over the part limit, since GitHub allows 2 GiB per file), the update archive and `SHA256SUMS`, in `dist/release/<version>/`. Marked as a pre-release. Needs mkosi, `zstd` and an authenticated `gh`. Refuses the public test keys. `--no-build` reuses images already built. |
| `publish-feed VERSION [--repo OWNER/NAME] [--dry-run]` | Publishes the update feed (`latest-x86_64.json`) to the fixed GitHub release named `updates`, replacing the previous one. Run it after the release `v<version>` is published, not while it is a draft. The default `ota.url` points at it. Needs an authenticated `gh`; `--dry-run` changes nothing. |
| `release-manifest --version V --base-url URL FILE... --output releases.json` | Adds an image release, with sizes and SHA-256 sums, to the download page's list. Uploads nothing. (The website now reads GitHub releases directly, so this is for hosting somewhere else.) |

## Trying and testing in QEMU

All of these use KVM when `/dev/kvm` is usable, and all are emulation: none is proof for real hardware. Tests that attach an
image do so read-only (`snapshot=on`) unless they say otherwise.

| Script | What it does |
| --- | --- |
| `qemu-try install\|boot\|reset` | Try the system by hand in a window: `install` boots the newest installer medium with a blank 40 GiB disk, `boot` boots the installed disk, `reset` erases the disk and firmware variables. Files live in `mkosi.output/qemu-try/`; if that folder is owned by root (a sudo build) the script cannot create it. The guest reaches the host at `10.0.2.2`. No TPM. |
| `qemu-smoke [IMAGE]` | Boots an image with no display and no login and checks it came up healthy, through a diagnostics service injected as a systemd credential. Exit status 0 means every check passed. |
| `qemu-uefi-test [IMAGE]` | Tries the boot splash's UEFI variable on OVMF: write it, reboot, find it, remove it. Does not test the initrd. |
| `qemu-verity-test [--image IMG]` | Boots a dm-verity image, checks the root really is verity-protected, then corrupts a copy and checks it refuses to boot. Needs the `verity` profile build. |
| `qemu-datadisk-test --disk DISK [--installer MEDIUM]` | Checks that `/var` and `/home` come from the disk the system started from even when another disk has the same labels. About 10 minutes. **Not run yet.** |
| `qemu-fallback-test --archive FILE [--installer IMG]` | Installs from the installer medium, installs an update, breaks it, and checks boot counting falls back to the system that worked. The archive must be newer than the installer's and differ in its root image. About 20 minutes. |
| `qemu-secureboot-test [--image IMG] [--uki UKI]` | Boots under enforced Secure Boot: must boot what our key signed and refuse an unsigned image, another key's, a tampered one and an unsigned loader. Uses a throwaway key, never the real one. Needs efitools, mtools and sbsign. |
| `qemu-encryption-test [--installer IMG] [--archive FILE] [--probe]` | Installs an encrypted system with a software TPM and checks that the TPM unlocks it, that another TPM does not, that the recovery key and a passphrase work, and (with `--archive`) that the TPM still unlocks after an update. `--probe` first reports what the guest sees. About 25 minutes for the first three. Needs swtpm. |
| `test-bundles-container [catalog]` | Runs the repository layer and bundle builder against the live openSUSE and Packman repositories inside a Tumbleweed container. `catalog` builds the real codecs bundle (a large download). Needs podman and the network. |

## Documentation and branding

| Script | What it does |
| --- | --- |
| `build-wiki [OUTPUT] [--cname DOMAIN]` | Builds one self-contained HTML wiki (default `docs/wiki/index.html`) from the Markdown under `docs/` and `foundation/README.md`: sidebar, search, light and dark themes, no external fonts or scripts. What is included and how it is grouped comes from `docs/wiki.json`; any page it does not mention goes under "Other" so nothing is silently dropped. It is generated: edit the Markdown and run it again. |
| `make-wordmark FONT.ttf [OUTDIR]` | Draws the "ersa.compute" wordmark as outlines in Inter Tight (an SVG that needs no font), light and dark variants. Needs `fonttools` and `uharfbuzz`. Default output folder `foundation/foundation_deck`. |

## Environment variables

| Variable | Used by | Meaning |
| --- | --- | --- |
| `ECP_WORK_DIR` | `build`, `build-installer-sysext`, `clean` | Scratch and cache folder (default `/tmp/ecp-work`). |
| `ECP_SECURITY_DIR` | `security-dir` and so `build`, `make-update-archive` | A folder of production signing keys. |
| `ECP_REQUIRE_SB_KEY=1` | `use-security-keys` (set by `release`) | Refuse the public test keys. |
| `ECP_SIGN_KEY` | `make-update-archive`, `stage-payload`, `release` | A GPG key in your own keyring to sign updates with. |
| `ECP_NOCHECK=1` | `build-foundation-package` | Skip the package's tests. |
| `ECP_UPDATE=1`, `ECP_GIT_BASE` | `fetch-sources` | Fast-forward existing checkouts; clone from another base address. |
| `ERSA_BLUETOOTH_SRC`, `ERSA_DIALER_SRC` | `build-rpms` | Source checkouts of the compiled packages. |

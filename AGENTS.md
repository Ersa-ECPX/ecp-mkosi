Ersa Compute Platform (ECP)

## Repositories
- As of 2026-11-07 the work is split into `Ersa-ECPX/ecp-mkosi` (this repository: mkosi configuration, branding, scripts, docs), `Ersa-ECPX/foundation` (foundation, foundationd, installer, tests) and `Ersa-ECPX/packages` (RPM specs, small package sources, the RPM repository workflow). The build scripts expect `foundation/` and `packages/` as git-ignored checkouts inside this repository; paths in this document and in `docs/` such as `foundation/README.md` or `packages/foundation/foundation.spec` refer to that layout. See README.md, "Repositories".

## Direction
- Build a polished, fast, dependable OS with mkosi, based on openSUSE Tumbleweed (rolling) as of 2026-11-04. The base moved from Rocky Linux 10 because the user wants a rolling base with current GNOME and kernel; Rocky 10.2 has GNOME Shell 49 and kernel 6.12. Rocky, CentOS Stream, ELN, Alma and Debian were evaluated at length and set aside; the user confirmed Tumbleweed on 2026-11-05. Do not reopen the base question unless the user does; if they do, propose a time-boxed side-by-side mkosi profile instead of switching.
- Storage: two fixed, equal root partitions (A/B, 8 GiB each, read-only EROFS) with one kernel image per slot and boot counting, decided 2026-11-05. A shared pool of EROFS image files was prototyped offline and set aside for robustness (it needs custom initrd code, so a bug would stop every entry from booting, and all images share one filesystem). Revisit only with fault-injection evidence from QEMU (kill mid-update, corrupt the images filesystem, delete an image). `/etc` is an overlay (image lower layer, persistent upper layer on `/var`) instead of a partition. `/var` and `/home` are subvolumes of one btrfs data volume (`ECP-DATA`, at least 10 GiB, last partition): shared space, separate lifecycles, real mounts so SELinux labels `/home` correctly. Delta updates and EROFS compression (lz4 or zstd) are later options.
- GNOME (Tumbleweed packages, GDM) is the selected desktop, replacing KDE Plasma as of 2026-11-03 because the KDE/EPEL image did not reach a graphical login. Revisit KDE only after a GNOME image boots reliably.
- Initial target: UEFI x86_64 tested in QEMU and on a PC. Tumbleweed is not built with an x86-64-v3 baseline (only about 0.5% of its packages have optional v3 builds through glibc-hwcaps, and its kernel is not compiled for v3). x86-64-v3 optimization is an optional, measured experiment, not a requirement. AArch64 and phones (ARMv8-A SoCs) come later as separate device-kernel projects; the RK3326 handheld is dropped.
- EAP (Ersa Automotive Platform) is outside this repository's scope for now. It was planned on a separate Yocto foundation; as of 2026-11-06 the user is open to building it from this mkosi platform later instead. Undecided: do not build for it yet, and do not claim either base publicly.

## Foundation
- `foundation` is the Ersa Compute Platform's primary OS management CLI and consistent supported interface for OS lifecycle and workstation configuration. It is only a client of `foundationd`, the D-Bus service (`org.ersa.Foundation1`) that authorizes with polkit and runs changes as jobs in worker processes. The contract is `foundation/data/dbus/org.ersa.Foundation1.xml`; the guide is `foundation/README.md`. No client implements system policy or runs another client's command line. Prefer D-Bus, a library or reading files over running a command; run a command only where nothing else exists.
- Naming: `ECP` is only for code (variables, labels, `IMAGE_ID`). Anything a person reads (CLI output and help, prompts, package descriptions, banners, GUI text) says "Ersa Compute Platform", never "ECP OS".
- Plan commands for inspection, status, updates, release selection, recovery, profiles, diagnostics, and supported performance/power configuration.
- Implement commands as their backing capabilities become available. Do not provide misleading success responses or placeholders advertised as working features.
- Delegate to established system services and tools rather than reimplementing package management, update transfer, boot management, or hardware controls.
- Keep system policy and orchestration reusable by future graphical clients. GNOME Software or KDE Discover integration is optional and must be verified against actual packaged versions; neither is a prerequisite for OTA.
- Separate OS image updates from application updates and persistent machine configuration. On immutable installations, base package and kernel changes belong in image builds.
- Make changes inspectable, repeatable, and reversible where possible. Provide previews for substantial configuration changes and require explicit confirmation for destructive storage operations.

## Delivery order
1. openSUSE Tumbleweed/mkosi base, GNOME, Ersa branding, boot correctness, responsiveness, and quality.
2. Signed image OTA exposed through foundation, with staged installation, boot health checks, and tested fallback.
3. Workstation profiles and measured performance/power tuning through foundation.
4. Optional adaptive performance service; consider ML only after measurements establish an advantage over explicit rules.

Design the disk layout and persistence boundaries for future OTA now. Do not assume arbitrary installer defaults will support the update design. Prefer Anaconda if its image deployment path is verified; document gaps before choosing alternatives.

## Packages and releases
- Maintain custom packages as RPM specs, patches, configurations, and pinned source references, built in isolated Tumbleweed build environments (containers), never on the host.
- Publish signed RPMs and source RPMs through an ECP DNF repository; mkosi consumes that repository to build OS releases.
- Keep RPM repository publication distinct from signed OS image publication.
- Port selected postmarketOS sources/patches into native RPM recipes rather than installing APKs into the base distribution. Treat device kernels as separately maintained variants.
- Keep the standard openSUSE `kernel-default` initially. CachyOS-inspired or v3-compiled kernels/configurations are experiments requiring workload benchmarks and recovery validation; the A/B boot-counting fallback is the recovery mechanism to validate them against.

## Engineering boundaries
- Verify current capabilities using authoritative documentation and actual package/build inspection. Distinguish proposals from tested functionality.
- Measure boot time, responsiveness under load, workload throughput, idle power, and stability. Do not claim universal performance improvements.
- Respect user power preferences and hardware thermal/safety limits. Do not add opaque automatic boosts or ML infrastructure to the initial release.
- Keep secrets and machine identity out of release images; initialize identity on first boot.
- Validate boot, persistence, update, interrupted update, and failed-boot recovery as applicable. Report what was actually tested.
- Prefer completing the current phase over expanding scope. Update this document when the user changes direction.

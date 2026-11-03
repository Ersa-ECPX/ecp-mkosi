Ersa Compute Platform (ECP)

## Direction
- Build a polished, fast, dependable Rocky Linux 10 based OS with mkosi.
- Keep KDE Plasma as the selected desktop. Do not switch to GNOME solely for update integration.
- Initial target: UEFI x86_64 with an x86-64-v3 CPU baseline, tested in QEMU and on a PC. AArch64 and device-specific Qualcomm support come later.
- EAP (Ersa Automotive Platform) uses a separate Yocto foundation and is outside this repository's scope.

## Foundation
- `foundation` is ECP's primary OS management CLI and consistent supported interface for OS lifecycle and workstation configuration.
- Plan commands for inspection, status, updates, release selection, recovery, profiles, diagnostics, and supported performance/power configuration.
- Implement commands as their backing capabilities become available. Do not provide misleading success responses or placeholders advertised as working features.
- Delegate to established system services and tools rather than reimplementing package management, update transfer, boot management, or hardware controls.
- Keep system policy and orchestration reusable by future graphical clients. KDE Discover or GNOME Software integration is optional and must be verified against actual packaged versions; neither is a prerequisite for OTA.
- Separate OS image updates from application updates and persistent machine configuration. On immutable installations, base package and kernel changes belong in image builds.
- Make changes inspectable, repeatable, and reversible where possible. Provide previews for substantial configuration changes and require explicit confirmation for destructive storage operations.

## Delivery order
1. Rocky 10/mkosi base, KDE, Ersa branding, boot correctness, responsiveness, and quality.
2. Signed image OTA exposed through foundation, with staged installation, boot health checks, and tested fallback.
3. Workstation profiles and measured performance/power tuning through foundation.
4. Optional adaptive performance service; consider ML only after measurements establish an advantage over explicit rules.

Design the disk layout and persistence boundaries for future OTA now. Do not assume arbitrary installer defaults will support the update design. Prefer Anaconda if its image deployment path is verified; document gaps before choosing alternatives.

## Packages and releases
- Maintain custom packages as RPM specs, patches, configurations, and pinned source references, built in isolated Rocky-compatible environments.
- Publish signed RPMs and source RPMs through an ECP DNF repository; mkosi consumes that repository to build OS releases.
- Keep RPM repository publication distinct from signed OS image publication.
- Port selected postmarketOS sources/patches into native RPM recipes rather than installing APKs into Rocky. Treat device kernels as separately maintained variants.
- Keep the standard Rocky kernel initially. CachyOS-inspired kernels/configurations are later experiments requiring workload benchmarks and recovery validation.

## Engineering boundaries
- Verify current capabilities using authoritative documentation and actual package/build inspection. Distinguish proposals from tested functionality.
- Measure boot time, responsiveness under load, workload throughput, idle power, and stability. Do not claim universal performance improvements.
- Respect user power preferences and hardware thermal/safety limits. Do not add opaque automatic boosts or ML infrastructure to the initial release.
- Keep secrets and machine identity out of release images; initialize identity on first boot.
- Validate boot, persistence, update, interrupted update, and failed-boot recovery as applicable. Report what was actually tested.
- Prefer completing the current phase over expanding scope. Update this document when the user changes direction.

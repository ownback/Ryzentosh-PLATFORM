# ARCHITECTURE: Ryzentosh-PLATFORM

Maintained by repo-cartographer. No code exists yet; the list below is the **planned** shape (marked planned), not a map of files.

## Components (planned)
- `RyzentoshPlatform.kext`: one bundle with these parts, each its own IOKit personality or class, so each can be tested alone:
  - NVMe power management for the Phison E26 controller (attaches to Apple's NVMe driver stack).
  - USB port map personalities (matching the xHCI controllers by their PCI location).
  - The RestrictEvents jobs, as config data where possible; any remainder as kext code.
  - Non-CPU sensors, published to VirtualSMC only for keys backed by real hardware state.
  - Kext-side sleep/wake.
- Not here: SSE (see `re/RB-0003e-t2-presence.md`), ACPI tables, CPU sensors.

## Entry points
None yet.

## Build targets
`RyzentoshPlatform.kext` (planned).

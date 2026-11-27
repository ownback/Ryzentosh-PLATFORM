# INTERFACES: Ryzentosh-PLATFORM

External contracts: IOKit classes and properties, user-client selectors, plist keys, files installed, interfaces consumed from other Ryzentosh repos.

**Provided:** `RyzentoshPlatform.kext` (planned; no interface defined yet).

**Consumed (planned):**
- VirtualSMC (Lilu-based, upstream): sensor keys, each backed by real hardware state or absent (ADR-0003).
- Ryzentosh-ACPI SSDTs: EC/USBX and the sleep SSDT.
- OpenCore config data from Ryzentosh-EFI where a RestrictEvents job becomes configuration.

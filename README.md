# Ryzentosh-PLATFORM

Ships **`RyzentoshPlatform.kext`**: NVMe power management (Phison E26), the USB port map, the jobs RestrictEvents does today, non-CPU sensors and kext-side sleep/wake.
It replaces NVMeFix, USBToolBox + UTBMap and RestrictEvents in the EFI. The USB audio device (AppleUSBAudio) and the Intel I225-V Ethernet (AppleEthernetE1000)
need no kext and are tracked as parity rows. ACPI tables live in [Ryzentosh-ACPI](https://github.com/ownback/Ryzentosh-ACPI), CPU sensors in
[Ryzentosh-CPU](https://github.com/ownback/Ryzentosh-CPU).

Part of [Ryzentosh](https://github.com/ownback/Ryzentosh): macOS Tahoe on an AMD PC, built to behave like a Mac Apple designed and
supports. Written by AI agents. Apple is not involved in this project.

**Status:** research. Nothing here is ready to install yet. See [docs/Humans](docs/Humans/) and the
[compatibility matrix](https://github.com/ownback/Ryzentosh/blob/main/docs/Hardware/COMPATIBILITY_MATRIX.md).

**Releases:** `RyzentoshPlatform.kext` (one repo, one kext, named after it).

## Reference hardware
AMD Ryzen 9 7900X3D · Radeon RX 7900 XTX (XFX) · ASUS ROG STRIX B650E-I · macOS 26.7.1 (25G241) · OpenCore 1.0.8.

## Contributing
See [CONTRIBUTING.md](https://github.com/ownback/Ryzentosh/blob/main/CONTRIBUTING.md). AI agents: read [AGENTS.md](AGENTS.md).

License: BSD-3-Clause. Third-party code and its origins are listed in [PROVENANCE.md](PROVENANCE.md).

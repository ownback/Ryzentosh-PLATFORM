# BRIEF: Ryzentosh-PLATFORM

**Scope:** ships one kext, **RyzentoshPlatform.kext** (owner decision 2026-11-27: one repo ships one kext named after it; target tree `docs/Architecture/EFI-END-STATE.md` in the control plane):

| In the kext | Replaces today (EFI) |
|---|---|
| NVMe power management (Phison E26) | NVMeFix |
| USB port map as personalities | USBToolBox + UTBMap |
| What RestrictEvents does today (preferably as OpenCore config data, e.g. PlatformInfo CustomMemory, not process patches) | RestrictEvents |
| Non-CPU sensors (CPU sensors are RyzentoshCPU's), as VirtualSMC keys backed by real hardware state | — |
| Kext-side sleep/wake (the wake-source SSDT is Ryzentosh-ACPI's) | — |

**Parity rows without kext work:** USB audio (the ALC4080 reaches macOS as a USB audio device, Apple's AppleUSBAudio) and Intel I225-V Ethernet (Apple's AppleEthernetE1000). They stay in the test plan and the compatibility matrix.

**The SSE/T2 question is answered** (RB-0003e/f, PR #5): no SSE code belongs here. RyzentoshSSE is removed in its own EFI change after the E-1 measurement on ownback/Ryzentosh-PLATFORM#4.

**Release artifact:** `RyzentoshPlatform.kext`.

**Depends on:** [Ryzentosh-EFI] (config data, kext injection), [Ryzentosh-ACPI] (EC/USBX and sleep SSDTs).

**Parity targets:** see the rows for this project in the control plane's COMPATIBILITY_MATRIX.md.

**Out of scope (owned elsewhere):** ACPI tables → Ryzentosh-ACPI. CPU sensors, CPU power management, the MCE reporter blocker and the AMD-V backend → Ryzentosh-CPU. dGPU and iGPU (Ryzentosh-dGPU is now Ryzentosh-GPU, with iGPU merged in) → Ryzentosh-GPU. Wi-Fi, Bluetooth, AWDL (Ryzentosh-AIR merged in) → Ryzentosh-WIRELESS. User-space installers → Ryzentosh-POST.

# Ryzentosh-PLATFORM

**Run `/ryzentosh` first.** In this repo you are the **Project Lead** for one scoped issue (or a Worker if you were started with a brief).

- Scope: ships **RyzentoshPlatform.kext** only (one repo, one kext). In: NVMe power (replaces NVMeFix), USB port map (replaces USBToolBox + UTBMap), RestrictEvents' jobs (prefer OpenCore config data), non-CPU sensors, kext-side sleep/wake. Parity rows, no kext: USB audio (AppleUSBAudio), I225-V Ethernet (AppleEthernetE1000). Out: ACPI (Ryzentosh-ACPI), CPU sensors/MCE/HV (Ryzentosh-CPU), GPU incl. iGPU (Ryzentosh-GPU), Wi-Fi/BT/AWDL (Ryzentosh-WIRELESS). SSE/T2: research done (RB-0003e/f), no code here.
- Memory: `docs/LLMs/BRIEF.md` (what this repo is for), `STATE.md` (now), `ARCHITECTURE.md`, `INTERFACES.md`, `TESTPLAN.md`, `decisions/ADR-*`.
- Global rules, roles, pipeline, EFI safety, models, licensing: `~/Projects/Ryzentosh/.agents/skills/ryzentosh/` (symlinked at `.agents/skills/ryzentosh`).
- Never write to an ESP or the live system. Hand build artifacts to the Root Orchestrator for the hardware test.
- License BSD-3-Clause: keep SPDX headers and update `PROVENANCE.md` for any derived code.
- Update `docs/LLMs/STATE.md` before you stop.

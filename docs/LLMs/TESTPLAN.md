# TESTPLAN: Ryzentosh-PLATFORM

| Parity target | Test | Host or device | Status |
|---|---|---|---|
| NVMe power management (Phison E26) | idle power states reached; no timeouts under load; compare with NVMeFix | device | not started |
| USB port map | every physical port enumerates at its speed; the map stays within macOS's per-controller port limit; internal devices present | device | not started |
| RestrictEvents' jobs | no wrong "Mac Pro" slot/memory notifications; CPU name correct; no process patches left | device | not started |
| Non-CPU sensors | each published SMC key reads from real hardware; absent keys are absent | device | not started |
| Sleep/wake | sleep and wake from keyboard/mouse; no lost devices | device | not started |
| USB audio (no kext) | the USB audio device appears and plays through AppleUSBAudio | device | parity row only |
| Intel I225-V Ethernet (no kext) | link at 2.5 Gb, AppleEthernetE1000 attached | device | parity row only |
| SSE removal (#4) | E-1 plan in ownback/Ryzentosh-PLATFORM#4 (Safari/passd, nfcd, seserviced; log queries; abort criteria) | device | planned (Orchestrator schedules) |

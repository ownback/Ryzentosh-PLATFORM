# STATE: Ryzentosh-PLATFORM

Updated: 2026-11-27 by the Project Lead (claude-sonnet-5-5 / high).

## Phase
Research. RB-0003e (#1) and RB-0003f (#2) are done and merged (PR #5, e4d9cd1).

## Done
- `docs/LLMs/re/RB-0003e-t2-presence.md`: signals, consumers, libAppleSSE's three outcomes, requestors (Safari -> passd, seld -> nfcd), options F0-F5. Verdict for #4: remove RyzentoshSSE, accept the SE-less behaviour (F4); no SSE code in PLATFORM.
- `docs/LLMs/re/RB-0003f-sep-features.md`: FileVault software path (provisionally class 1), keychain, Touch ID, Apple Pay, Secure Enclave keys, Activation Lock, Find My.
- Independent review (claude-fable-5-2) done, fixes folded in.
- #4 comment (https://github.com/ownback/Ryzentosh-PLATFORM/issues/4#issuecomment-7228929840): the E-1 test plan for RyzentoshSSE's removal: which processes can wait 30 s (passd, nfcd, seserviced), the log queries, the baseline, and the pass / pass-with-cost / abort criteria.

## Scope (owner decisions 2026-11-27)
This repo ships one kext, **RyzentoshPlatform.kext**: NVMe power (replaces NVMeFix), USB port map (replaces USBToolBox + UTBMap), RestrictEvents' jobs (preferably OpenCore config data), non-CPU sensors, kext-side sleep/wake. ACPI moved to Ryzentosh-ACPI. USB audio and I225-V Ethernet are parity rows with no kext work. No SSE code. Docs realigned in README, BRIEF, CLAUDE.md, ARCHITECTURE, INTERFACES, TESTPLAN and CHANGELOG (this PR).

## In flight
Nothing. The removal is an EFI change for the EFI Lead after the current Wi-Fi test; the Lead does not run E-1.

## Next step
1. After E-1, update the compatibility-matrix rows (Apple Pay / NFC / Secure Element) from its numbers and close #4.
2. The FileVault one-shot test waits for an owner decision (Orchestrator raises it).
3. PLATFORM's next work: the NVMe power, USB map and RestrictEvents jobs for RyzentoshPlatform.kext (research first: what RestrictEvents does that config data can replace).
4. For the Orchestrator: the control-plane `projects.yaml` entry for this repo still carries the old scope line (audio kext work, ACPI).

## Blockers
None. Open (RB-0003e section 7): the PassKitCore static call site, and the unobserved 30 s wait (E-1 observes it).

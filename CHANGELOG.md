# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/). Release stages: research → alpha → beta → stable.

## [Unreleased]
- Research: RB-0003e (T2/SEP presence signals, consumers, the libAppleSSE wait) and RB-0003f (Secure Enclave features) in `docs/LLMs/re/` (PR #5).
- Docs: scope realigned to the owner's layout decisions of 2026-11-27: this repo ships `RyzentoshPlatform.kext` (NVMe power, USB map, RestrictEvents' jobs, non-CPU sensors, sleep/wake); ACPI moved to Ryzentosh-ACPI; audio and Ethernet are parity rows without kext work; no SSE code.
- Repository created from the Ryzentosh skeleton (2026-11-27).

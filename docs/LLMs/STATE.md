# STATE: Ryzentosh-PLATFORM

Updated: 2026-11-27 by the Project Lead (claude-sonnet-5-5 / high).

## Phase
Research. RB-0003e (#1) and RB-0003f (#2) run in one draft PR from branch `research/1-2-t2-presence-sep`.

## In flight
- RB-0003e: `docs/LLMs/re/RB-0003e-t2-presence.md` written (checkpoint 1). Answers #4's evidence question; two test-boot experiments (E-1, E-2) proposed.
- RB-0003f: `docs/LLMs/re/RB-0003f-sep-features.md` written (worker static analysis plus live probes).
- Independent review by claude-fable-5-2 before the PR is marked ready.

## Next step
Independent review (claude-fable-5-2), fix findings, mark the PR #5 ready, report to the Root Orchestrator.

## Blockers
None. PassKitCore's SSE call site is not located (tooling limit, see RB-0003e §7); a decompiler pass would settle it.

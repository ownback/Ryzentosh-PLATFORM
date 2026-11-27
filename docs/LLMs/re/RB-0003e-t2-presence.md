<!-- SPDX-License-Identifier: BSD-3-Clause -->

# RB-0003e: T2 presence reporting without cryptographic identity

Brief: control plane `docs/Research/briefs/RB-0003e-t2-presence-reporting.md` (ADR-0003 point 2). Issue: ownback/Ryzentosh-PLATFORM#1.
Feeds ownback/Ryzentosh-PLATFORM#4 (retire RyzentoshSSE). Companion: `RB-0003f-sep-features.md`. Prior work: RB-0003b (`Ryzentosh-EFI docs/LLMs/re/RB-0003b-t2-features.md`).
Date: 2026-11-27. Build: macOS 26.7.1 **25G241**, x86_64, SMBIOS MacPro7,1, no T2, KDK 26.7.1_25G241 installed.
Author: Project Lead, claude-sonnet-5-5/high. Reviewed independently by claude-fable-5-2; its corrections are folded in.
Method: read-only. `nm`, `otool`, `strings` and `ipsw dyld` on images extracted from the dyld shared cache into a scratch directory, KDK kext
plists and symbols, launchd and sandbox profiles, `ioreg`, `sysctl`, `launchctl print`, `/usr/bin/log show` on this boot's retained log, one MobileGestalt
probe. No boot-config change, no ESP write, no experiment that changes the system, no Apple code or disassembly in this repo, no identity data.
NootRX/NootedRed not read. The 2026-11-25 notes in the archived ryzentosh-sse bundle (MIT, ours) were used as a lead; every claim used here was re-read.

Labels: **verified** = read in a 25G241 binary/plist or observed live; **inferred** = follows from what was read; **open** = not established.
Evidence: `DSC:<image>` (dyld shared cache), `disk:<path>`, `KDK:<kext>`, `live:<command>`.

## 0. Answer for #4

1. Every consumer of the `AppleSSE` service goes through `/usr/lib/libAppleSSE.dylib`. Importers: **passd** (PassKitCore, one symbol), **nfcd**, **seserviced**.
   SEService links it without binding a symbol. libSEUpdater carries its own copy of the client (and the Aloha pairing code). (verified, §2)
2. libAppleSSE has **no non-T2 shortcut and distinguishes three outcomes** (verified, §3):
   service found and answers "no SSE" → -17 at once, cached; service found but open or call fails → fast error, not cached;
   **no service → polls 0.5 s for 30 s, returns -20, not cached**, so every later SSE call in that process waits 30 s again.
3. **The stand-in does not work the way its author thought.** In passd the log shows libAppleSSE's `IOServiceOpen` check failing with `0xe00002e2`
   (kIOReturnNotPermitted) about 120 ms after spawn. passd's and nfcd's sandbox profiles allow only user-client class `AppleSSEUserClient`; the stand-in's is
   `RyzentoshSSEUserClient`. So RyzentoshSSE shortens the wait **by registry presence under the class name alone**; its "answers 0" code is not reached by the sandboxed
   consumers. (open/inferred: that the sandbox is the cause; verified: the error, the profiles, the class names.)
4. **Both requestors on this machine are stock macOS, so F2 ("stop the requestor") is closed negative** (§5): Safari wakes passd through the in-app-payment Mach service
   (PassKit's Apple Pay availability check), and `seld` wakes nfcd at every boot. A real Mac does the same.
5. The consumers handle "no SE" cleanly in their own code (§2), and the libAppleSSE call in passd runs on a **worker thread after "passd launch: finished"**, so what the 30 s would
   stall is that worker queue, not passd's launch. (verified from the log, with RyzentoshSSE loaded; the unstubbed wait itself is unobserved.)
6. **Recommendation:** no PLATFORM code is needed for the SSE question. Remove RyzentoshSSE (EFI E5) and treat the SE-less state as macOS's own behaviour on the one
   SE-less x86 configuration it supports (`VMM-x86_64`), with one test boot to measure the user-visible cost (§5, E-1). Only a SEP makes the wait go away the way Apple does it.

## 1. Signals, classified (questions 1 and 3)

Class (a) = can be reported truthfully because a Ryzentosh component really provides the service behind it. (b) = reporting makes a consumer start a
cryptographic or Apple-server flow we must not fake, so it is **not reported**. (c) = irrelevant to T2.

| # | Signal | Producer on a T2 Mac | Consumers (25G241) | Here today | Class | Label | Evidence |
|---|---|---|---|---|---|---|---|
| S1 | PCI function `SEPM` → `AppleSEPManager` instance (and its endpoint children) | the T2 | AppleSSE (x86 personality, provider `AppleSEPManager`), AppleSEP* endpoint drivers; AppleKeyStore has a runtime SEP branch (RB-0003f) | `AppleSEPManager` loaded as a dependency, **no instance** | **(b)** | verified | KDK:AppleSSE, KDK:AppleSEPManager Info.plist; `live: kmutil showloaded` |
| S2 | IOService class `AppleSSE` (IOKit registry) | AppleSSE.kext, child of S1 | libAppleSSE (looked up by class name), through it nfcd, seserviced, passd | published by **RyzentoshSSE**, a stand-in | **(b)** (it is the SE/SEP storage identity); the stand-in is a workaround | verified | `live: ioreg -rc AppleSSE`; DSC:libAppleSSE `_init` |
| S3 | `AppleSSE` presence answer (command `'SSEc'` 0x1c, 1-byte answer) | AppleSSE.kext's `performCommandGated` checks the tag and calls `AppleSSE::hasBridgeSSE`, which writes 1 if a SEP endpoint and send buffer exist, else logs "SSE not present" and writes 0 | libAppleSSE `_init` | the stand-in is meant to answer 0, but sandboxed consumers are refused the user client (§0 item 3) | **(b)** | verified (KDK symbols, strings) | KDK:AppleSSE symbols and strings; DSC:libAppleSSE |
| S4 | MobileGestalt `HasSEP`, `EffectiveProductionStatusSEP`, `EffectiveSecurityModeSEP`, `SEPNonce`; `embeddedSecureElement`, `secureElements` | Apple silicon: IORegistry `IODeviceTree:/arm-io/sep`. Intel T2 source: **open** (libMobileGestalt also carries `BridgeOSGestaltKeys`) | libMobileGestalt callers | `HasSEP`=false, other SEP keys false, `embeddedSecureElement`/`secureElements`=null | **(b)** (truthfully false here) | verified (answers, key names); open (Intel derivation) | `live: MGCopyAnswer probe`; DSC:libMobileGestalt strings |
| S5 | ACPI `APP7777` → `AppleEmbeddedOSSupportHost` | the T2 | **launchd**: `com.apple.passd` (LaunchAgent) and `com.apple.eoshostd` (LaunchDaemon) start on it | absent; `eoshostd` has `runs = 0` | **(b)** | verified | `disk:` launchd plists; `launchctl print` |
| S6 | `AppleBiometricServices` | Touch ID path over the T2 | **launchd**: `com.apple.biometrickitd` (and one `AppleUSBNCMData` product id) | absent; `runs = 0` | **(b)** | verified | `disk:` launchd plist |
| S7 | NVRAM `BridgeOSHardwareModel` → `bridge-model` → `hw.target`; `BridgeOSVersion` | EFI / T2 | SoftwareUpdateCoreSupport (OpenCore docs); the string `hw.target` is in libMobileGestalt (users not traced) | `hw.target` empty | **(b)**: not reported (ADR-0003) | verified (state), inferred (other consumers) | `live: sysctl hw.target`; RB-0003b row 14 |
| S8 | `IOBufferCopyEngine`, `AppleUSBVHCIBCE`, iBridge USB, T2 audio `ADIO` | the T2 | Apple's bridge kexts | absent | **(b)**, never emulated | verified | RB-0003b §2, row 15 |
| S9 | SMC (AppleSMC) | inside the T2 on T2 Macs | everything reading SMC keys | VirtualSMC + sensor plugins | **(a)** if every key is backed by real state | verified | RB-0003b row 13 |
| S10 | `kern.hv_vmm_present`, `hw.model`, board-id | firmware / hypervisor | many | 0, MacPro7,1 | (c) here (RB-0003a covers the model) | verified | `live: sysctl` |

**Reading of S1 to S8.** Each is a pointer to Apple's secure processor or to bridgeOS, and the services behind it (Secure Enclave, secure storage, Touch ID, bridge updates)
are class (b)/(3) of ADR-0003. So **no T2 presence signal in the SE/SEP/bridge family is reportable**; the only (a) signal is the SMC. This agrees with RB-0003b and ADR-0003, and RB-0003e
proposes no addition to what the live system reports. Two ways of making Apple's kext answer "0" were considered and rejected under point 3: giving it a fake provider
(an `AppleSEPManager`-shaped node) and publishing `sep-booted`. Either one makes a consumer believe in a SEP (RB-0003f §0 item 4).

## 2. Consumers (question 2)

| Consumer | How it starts | What it asks | Present (T2 Mac) | Absent (this machine, no stand-in) | Label | Evidence |
|---|---|---|---|---|---|---|
| **libAppleSSE** (in-process client) | linked by nfcd, seserviced; imported by PassKitCore; SEService links it, binds no symbol | looks up service `AppleSSE`, `IOServiceOpen`, one presence call, a general-interest notification | continues | the three outcomes of §3 | verified | DSC:libAppleSSE `_init` |
| **passd** (PassKitCore) | launchd: `AppleEmbeddedOSSupportHost` first match (never here), or Mach services. **Here: Safari's activation of `com.apple.passd.in-app-payment`** (PassKit Apple Pay availability, through `PKNotifyCoalescer`) spawns it "because ipc (mach)"; also user-notification, nsurlsessiond, APS and idswake events | PassKitCore imports one libAppleSSE symbol, the effacement-notification handler setter; uses NearField's `NFSecureElement` otherwise | normal | libAppleSSE runs on a **worker thread ~120 ms after spawn, after "passd launch: finished"**; code has explicit "SE not available" / "No secure element, cannot …" branches | verified (log, strings, import); open (static call site not located, §7) | `live: /usr/bin/log show` (passd spawn, `AssertMacros … AppleSSElib.m line 521`, line 2328); `nm -u`; `strings` on passd and PassKitCore |
| **nfcd** | **`seld` opens `com.apple.nfcd.hwmanager` at each boot** (log: nfcd spawned "because ipc (mach)" early in the boot); `seld` itself starts on an event publisher (seserviced's plist subscribes `com.apple.seld.xpc.events`); also a daily `com.apple.nfcd.coreanalytics.event` activity | `SSEIsFeatureSupported` at **9 call sites**, `SSEGetVersion` at 3, authorise/sign/ACL calls; `-[NFSingleUserController loadUsersToAppleSSE]` at startup | uses them | each of the 9 sites tests the result and branches (the traced one treats non-zero as "feature unsupported"): **clean**. At the observed boot nfcd logged nothing from libAppleSSE (the stand-in was loaded), so a 30 s stall at nfcd start is **not shown** | verified | `disk:/usr/libexec/nfcd` x86_64 slice; `live: /usr/bin/log show` |
| **seserviced** | LaunchAgent started by notifications (keybagd first unlock, Octagon, stockholm, `SE-PTC` view, `com.apple.nfcd.xpc.sesd.events`); `_LimitLoadFromVariant = HasFactoryContent` is an OS-variant filter, not a hardware condition | `SSEIsFeatureSupported`, `SSEGetHostChallenge`, `SSEGetSignedDeviceData`, authorise-and-sign, ACL request/evaluate | uses them | **`runs = 0` this boot**: not exercised here. Handling not traced (Swift) | verified (imports, plist, `launchctl print`); open (handling) | `nm -u`; launchd plist |
| **SEService** (framework in passd) | linked by passd | `LC_LOAD_DYLIB` on libAppleSSE, no bound symbols; `SESKey*`, `SEFidoKeyService` APIs | normal | not shown | verified (load command); open | `nm -u`, `otool -L` on the extracted image |
| **libSEUpdater** | in the cache; no cache image links it | its own copy of the client (same retry message); `AlohaPairingVerifier performAuthKeySharingWithAppleSSE:` | normal | same 30 s polling (own copy) | verified (strings); inferred (same logic) | `ipsw dyld str` |
| **biometrickitd** | launchd on `AppleBiometricServices` (S6) | BiometricKit | normal | **never started** | verified | plist; `runs = 0` |
| **eoshostd** | launchd on S5 | bridge support | normal | never started | verified | plist; `runs = 0` |
| **keybagd / AppleKeyStore / AppleFDEKeyStore** | `keybagd` RunAtLoad; kexts on IOResources | key bags; AppleKeyStore has a runtime SEP branch (`_sep_enabled`, `sep-booted`, `sep_wait_for_reply`) | SEP-backed on T2 | **run in software without a SEP**; a faked SEP signal would route key operations into the SEP wait | verified (loaded, strings); inferred (branch decision) | `live: ps`, `kmutil showloaded`; KDK:AppleKeyStore (RB-0003f) |

Other SEP consumers (coreauthd, LocalAuthentication, securityd, FileVault tools) are in RB-0003f.

## 3. libAppleSSE: three outcomes (read from the 25G241 binary; the reviewer re-read it independently)

Symbols are present in the extracted library, so the names below are Apple's.

- Every public entry point calls the one-time setup `_init`, which returns at once if it already ran. (A few entry points, such as the BAA certificate ones, reach it through a once-block.)
- **Service found, `IOServiceOpen` succeeds, presence call returns a non-zero byte:** present; a general-interest notification is set up; initialised flag set.
- **Service found, presence call returns 0:** flag set with "no SSE"; every call returns **-17** at once.
- **Service found, `IOServiceOpen` or the call fails** (the assert at source line 521 is the open check): fast error return, **flag not set**. This is what passd shows today.
- **No service:** log "AppleSSE service not found, retrying..." (error level, once per iteration; the old notes counted 60 lines per passd start), wait **500000 µs**, look again; ends when
  the elapsed time passes **30 s** (a -30.0 double in `__TEXT.__const` compared with an `NSDate` interval), then returns **-20 without setting the flag**.
- The flag is stored on exactly two paths (present, and "no"). So with no service, **every** SSE call that runs `_init` repeats the full 30 s.
- `_SSEIsFeatureSupported` calls `_init`, then returns an error if the has-SSE byte is false.
- `_getMGAnswers` makes two MobileGestalt queries; the first is discarded and the second is `InternalBuild`, used for the internal log file. No model or hardware condition gates the loop.
- Exported but unread: `_sseSkipSEAvailabilityCheck` (zero readers in libAppleSSE, and no other image names it) and `_sseDoNotIssueNewBAACertificate`.
- `libAppleSSEExt.dylib` is loaded lazily for BAA certificates only.

Prior run (archived note, seen by its author, not re-run): a test of the real library with no service took 30.1 s and made 61 lookups, matching the loop above.
**The unstubbed wait is still unobserved on this machine**: no "retrying" line exists in the retained log, because the stand-in has been loaded in every boot covered.

## 4. Apple's fast "no" needs a SEP

`AppleSSE.kext` has two personalities: `sep-endpoint,sse` on `AppleSEPDeviceService` (Apple silicon) and `AppleSSE_x86` on **`AppleSEPManager`** (Intel T2). Its KDK
symbols show `performCommandGated` checking the 'SSEc' tag and calling `AppleSSE::hasBridgeSSE(void*, unsigned int*)`, which writes **1** when a SEP endpoint and send buffer exist and
otherwise logs "SSE not present" and writes **0** (verified, from the KDK binary). So the "0" that libAppleSSE turns into a fast -17 is produced by Apple's kext on a SEP that has no secure storage
element. Without a SEP there is no `AppleSEPManager` instance, the kext cannot match, and nothing in macOS publishes `AppleSSE`. Launchd side: passd and `eoshostd` start from
`AppleEmbeddedOSSupportHost`, a T2-only object (S5); Macs without it still start passd on demand.

## 5. Candidate fixes at the source

| Option | What it is | Verdict |
|---|---|---|
| F0 | Provide `AppleSSE` for real | **Impossible** (needs a SEP; ADR-0003 point 3). |
| F0b | Give Apple's kext a provider (an `AppleSEPManager`-shaped node) so it answers 0 | **Rejected** under ADR-0003 points 2 and 3: it fakes a SEP and sends AppleKeyStore into the SEP wait. |
| F1 | Keep RyzentoshSSE | The Orchestrator ruled it a workaround. It works by registry presence only (§0 item 3), so it is also an unreliable one. Remove it. |
| F2 | Stop the requestor | **Closed, negative.** The requestors are stock: Safari's PassKit availability check (passd) and `seld` (nfcd). A real Mac does the same. |
| F3 | A launchd condition real Macs without SE hardware use | **Closed (launchd half), negative.** Every plist in `/System/Library/Launch{Agents,Daemons}` was scanned: the only SE/T2-family IOKit triggers are S5 and S6; passd, nfcd, seld and seserviced have no hardware load-limit key. `HasFactoryContent` is an OS-variant filter. passd logs "Updated hardware state : 0" and calls libAppleSSE ~80 ms later, so an in-process gate is unlikely. **Open**: how MobileGestalt derives `HasSEP` on Intel (S4). |
| F4 | Accept the SE-less behaviour (no stand-in) | What macOS does on `VMM-x86_64`, the only SE-less x86 configuration 25G241 supports (RB-0003b §1). The user-visible cost is a stall on the libAppleSSE worker queue per SSE call, **not measured**. A legitimate user-level mitigation exists in Safari's settings (the "check for Apple Pay" privacy option), untested. |
| F5 | `launchctl disable` the daemons, hook or patch libAppleSSE | **Rejected**: workarounds of the same kind as RyzentoshSSE. |

**Recommendation for #4:** F1 → remove, F4 → accept and record. PLATFORM needs no SSE code, and RyzentoshPlatform.kext carries no SSE piece. Update the matrix row to:
"Apple Pay / NFC / Secure Element: not available: needs a Secure Enclave. First Apple Pay availability check may stall (measured: pending)."

### Experiment (a test-boot request for the Root Orchestrator, ADR-0006; the Lead does not run it)

- **E-1: measure the cost without the stand-in.** One candidate boot with RyzentoshSSE removed (the E5 EFI content), bundled with another boot-relevant change to save a reboot. Capture after login, with `/usr/bin/log show --start <boot>`:
  every `libAppleSSE` line per process, the time between passd's spawn and its libAppleSSE failure, how long `PKPaymentAuthorizationController.canMakePayments()` takes to return, and nfcd's start. Abort criterion: login or the desktop stalls for more
  than a minute because of SSE; **rollback**: boot the USB known-good, which has the stand-in. Expected: the open/call failure in §3 goes away and the 30 s poll appears in passd/nfcd's workers instead; the numbers decide whether F4's cost is acceptable to the owner.
- **E-2 (static, no boot, optional):** decode how MobileGestalt derives `HasSEP` on Intel (obfuscated key table, REA needed). Only matters if a future consumer is found that gates on it.

## 6. The live system today (question 4), by class

| Component | Behaviour | Class | Label |
|---|---|---|---|
| RyzentoshSSE 0.1.0 | publishes `AppleSSE` on IOResources with user client `RyzentoshSSEUserClient`; shortens the wait by registry presence; sandboxed consumers are refused the user client (`0xe00002e2`) | stand-in under an Apple class name; **workaround**, no key material | verified (`ioreg`, log, sandbox profiles) |
| RestrictEvents 1.1.6 | model-UI mitigation | (c) | RB-0003b §4 |
| AppleMCEReporterDisabler 1.2 | CPU personality outbidding the MCE reporter | (c) | RB-0003b §4 |
| VirtualSMC + sensors | SMC service | (a), key audit pending | RB-0003b row 13 |

`ioreg` also shows `AppleSEPManager` loaded with no instance, `AppleKeyStore` and `AppleFDEKeyStore` running, no `AppleEmbeddedOSSupportHost`, no `AppleBiometricServices` (verified).

## 7. Not verified / limits

- The PassKitCore **static call site** of the effacement-handler setter was not located (the cache GOT binding could not be attributed with the tools used). The log gives its timing instead (§2).
- The unstubbed 30 s wait was read, not run or logged on this machine (E-1 measures it).
- seserviced did not run this boot; SEService and libSEUpdater handling of the failure was not traced.
- Whether the sandbox is the cause of the `IOServiceOpen` refusal is inferred from the profiles and class names; no sandbox-denial line was logged.
- How `HasSEP` is derived on Intel is open.
- Strings that `ipsw dyld str` attributes to an image show where a string lives in the cache, not that code there reads it.
- Whether a payment flow succeeds without the stand-in is untested.

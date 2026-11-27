<!-- SPDX-License-Identifier: BSD-3-Clause -->

# RB-0003e: T2 presence reporting without cryptographic identity

Brief: control plane `docs/Research/briefs/RB-0003e-t2-presence-reporting.md` (ADR-0003 point 2). Issue: ownback/Ryzentosh-PLATFORM#1.
Feeds ownback/Ryzentosh-PLATFORM#4 (retire RyzentoshSSE). Prior work: RB-0003b (`Ryzentosh-EFI docs/LLMs/re/RB-0003b-t2-features.md`).
Date: 2026-11-27. Build: macOS 26.7.1 **25G241**, x86_64, SMBIOS MacPro7,1, no T2, KDK 26.7.1_25G241 installed.
Author: Project Lead, claude-sonnet-5-5/high. Method: read-only. `nm`, `otool`, `strings` and `ipsw dyld` on images extracted from the dyld
shared cache into a scratch directory, KDK kext plists, launchd plists, `ioreg`, `sysctl`, `launchctl print`, one MobileGestalt probe
(read-only). No boot-config change, no ESP write, no experiment on the live system, no Apple code or disassembly in this repo,
no identity data. NootRX/NootedRed not read. The 2026-11-25 notes in the archived ryzentosh-sse bundle (MIT, ours) were read as
a lead and every claim used here was re-read in the binaries.

Labels: **verified** = read in a 25G241 binary/plist or observed live; **inferred** = follows from what was read;
**open** = not established. Evidence is `DSC:<image>` (dyld shared cache), `disk:<path>`, `KDK:<kext>`, `live:<command>`.

## 0. Answer for #4 in six lines

1. Every consumer of the `AppleSSE` service goes through one library, `/usr/lib/libAppleSSE.dylib`. Callers found: **passd** (PassKitCore, one
   import), **nfcd**, **seserviced**, SEService, libSEUpdater (own copy of the client) and USDKit (Aloha pairing). (verified)
2. libAppleSSE has **no non-T2 shortcut**. If no `AppleSSE` service exists it re-looks every 0.5 s for 30 s, then fails with -20, and it does
   **not** remember the failure, so every later SSE call in that process waits 30 s again. (verified, §3)
3. A fast "no" exists only as an *answer from an `AppleSSE` service* (presence byte 0, §3: verified). Apple's own `AppleSSE.kext` is the only Apple
   code that publishes that service, and it matches only on an `AppleSEPManager` instance, which needs a SEP (verified). That it is also the
   producer of the "0" answer is inferred from its "SSE not present" string. So **only a SEP makes the wait go away the way Apple does it**.
4. The consumers themselves handle "no SE" cleanly (strings and one traced call site in nfcd, §4). The only defect is the 30 s wait in front of that.
5. Providing the service for real is impossible (point 3 of ADR-0003: it is the SEP/SSE identity). RyzentoshSSE is the only way found
   to make the wait short, and it is the stand-in under an Apple class name that the Orchestrator ruled a workaround. (verified)
6. So "fix at the source" has three candidate sources and none is established yet (§5): the **requestor** that wakes passd/nfcd/seserviced here
   (open), a **launchd/MobileGestalt condition** that Macs without Apple Pay hardware use (open), or accepting the 30 s as macOS's own
   behaviour on SE-less hardware (documented, but a defect for the owner). §5 lists the two cheap experiments that decide it.

## 1. Signals, classified (questions 1 and 3)

Class (a) = can be reported truthfully because a Ryzentosh component really provides the service behind it. (b) = reporting makes a consumer start a
cryptographic or Apple-server flow we must not fake, so it is **not reported**. (c) = irrelevant to T2.

| # | Signal | Producer on a T2 Mac | Consumers (25G241) | Here today | Class | Label | Evidence |
|---|---|---|---|---|---|---|---|
| S1 | PCI function `SEPM` → `AppleSEPManager` instance (and its endpoint children) | the T2 | AppleSSE (x86 personality, provider `AppleSEPManager`), AppleSEP* endpoint drivers; `AppleSEPManager` is the root of every SEP chain | `AppleSEPManager` loaded as a dependency, **no instance** | **(b)** | verified | KDK:AppleSSE, KDK:AppleSEPManager Info.plist; `live: kmutil showloaded` |
| S2 | IOService class `AppleSSE` (IOKit registry) | AppleSSE.kext, child of S1 | libAppleSSE (looked up by class name), through it nfcd, seserviced, passd, SEService, libSEUpdater, USDKit | published by **RyzentoshSSE**, a stand-in | **(b)** (it is the SE/SEP storage identity); the stand-in is a workaround | verified | `live: ioreg -rc AppleSSE` (IOClass AppleSSE, CFBundleIdentifier org.ryzentosh.sse); DSC:libAppleSSE `_init` |
| S3 | `AppleSSE` presence answer (command `'SSEc'` 0x1c, 1 byte) | AppleSSE.kext; Apple's kext logs "SSE not present" and has a `hasBridgeSSE` property when the SEP says there is none | libAppleSSE `_init` | answered "0" by the stand-in | **(b)** | verified (read) | KDK:AppleSSE strings; DSC:libAppleSSE `_init` |
| S4 | MobileGestalt `HasSEP`, `EffectiveProductionStatusSEP`, `EffectiveSecurityModeSEP`, `SEPNonce`; `embeddedSecureElement`, `secureElements` | Apple silicon: IORegistry `IODeviceTree:/arm-io/sep`. Intel T2 source: **open** (libMobileGestalt also carries `BridgeOSGestaltKeys`, keys that the bridge supplies) | libMobileGestalt callers (daemons that ask `MGGetBoolAnswer`) | `HasSEP`=false, the other SEP keys false, `embeddedSecureElement`/`secureElements`=null | **(b)** (truthfully false here) | verified (answers, key names); open (Intel derivation) | `live: MGCopyAnswer probe`; DSC:libMobileGestalt strings |
| S5 | ACPI `APP7777` → `AppleEmbeddedOSSupportHost` (the bridgeOS support host) | the T2 | **launchd**: `com.apple.passd` (LaunchAgent) and `com.apple.eoshostd` (LaunchDaemon) both start on `IOProviderClass = AppleEmbeddedOSSupportHost`, first match | absent, so neither starts from this trigger | **(b)** (host protocol to a bridge OS) | verified | `disk:/System/Library/LaunchAgents/com.apple.passd.plist`, `…/LaunchDaemons/com.apple.eoshostd.plist` |
| S6 | `AppleBiometricServices` | Touch ID sensor path over the T2 | **launchd**: `com.apple.biometrickitd` starts on `AppleBiometricServices` (and on a particular `AppleUSBNCMData` product id) | absent, so biometrickitd is not started | **(b)** | verified | `disk:…/LaunchDaemons/com.apple.biometrickitd.plist` |
| S7 | NVRAM `BridgeOSHardwareModel` → IODT `bridge-model` → `hw.target`; `BridgeOSVersion` | EFI / T2 | SoftwareUpdateCoreSupport (update asset choice, per OpenCore docs); the string `hw.target` is in libMobileGestalt and a few more images (users not traced) | `hw.target` empty, no `bridge-model` | **(b)**: not reported (ADR-0003, Applications of the policy) | verified (state), inferred (consumers beyond SUCS) | `live: sysctl hw.target`; DSC:libMobileGestalt; RB-0003b row 14 |
| S8 | `IOBufferCopyEngine`, `AppleUSBVHCIBCE`, iBridge USB, T2 audio `ADIO` | the T2 | Apple's bridge kexts | absent | **(b)**, never emulated | verified | RB-0003b §2 and row 15 |
| S9 | SMC (AppleSMC) | inside the T2 on T2 Macs | everything that reads SMC keys | VirtualSMC + sensor plugins | **(a)** if every key is backed by real state (ADR-0003) | verified | RB-0003b row 13 |
| S10 | `kern.hv_vmm_present`, `hw.model`, board-id | firmware / hypervisor | many | 0, MacPro7,1 | (c) here (RB-0003a covers the model) | verified | `live: sysctl` |

**Reading of S1 to S8.** None of them can be reported truthfully: each one is a pointer to Apple's secure processor or to bridgeOS, and the services
behind it (Secure Enclave, secure storage, Touch ID, bridge updates) are class (b)/(3) of ADR-0003. So the answer to question 3 is: **no T2 presence signal
is reportable at all** for the SE/SEP/bridge family. The only (a) signal is the SMC. This agrees with RB-0003b and ADR-0003, and RB-0003e does not
propose an addition to what the live system reports.

## 2. Consumers (question 2)

For each: how it is reached, what it does with the signal, and what it does without it.

| Consumer | How it starts | What it asks | Present (T2 Mac) | Absent (this machine, no stand-in) | Label | Evidence |
|---|---|---|---|---|---|---|
| **libAppleSSE** (client library, in-process) | linked by nfcd, seserviced; imported by PassKitCore; `dlopen`-by-path in SEService and USDKit | `IOServiceGetMatchingService("AppleSSE")`, then `IOServiceOpen` and one presence call, then `IOGeneralInterest` notification | continues; SSE calls work | **polls 0.5 s × 60, then -20; failure is not cached**, so each later call waits 30 s again (§3) | verified | DSC:libAppleSSE `_init` |
| **passd** (PassKitCore) | launchd: on `AppleEmbeddedOSSupportHost` first match, or on demand via any `com.apple.passd.*` Mach service (payment, library, account, …); here it was started by a client (requestor open) | PassKitCore imports exactly one libAppleSSE symbol, `SSESetEffacementNotificationHandler` (block variant). It also uses NearField's `NFSecureElement` for everything else | normal | would block at the first SSE call (30 s); the code has explicit "SE not available" / "No secure element, cannot …" branches for later steps | verified (import, strings); inferred (the call site is on the startup path: not located, §6) | `nm -u` on DSC:PassKitCore; `strings` on `disk:…/passd` and DSC:PassKitCore; `ipsw dyld patches` shows one client slot for that symbol |
| **nfcd** | on demand (`com.apple.nfcd`, `com.apple.nfcd.hwmanager`), UserName `_applepay` | `SSEIsFeatureSupported` at **9 call sites** and `SSEGetVersion` at three, plus authorise/sign/ACL calls; `NFSingleUserController loadUsersToAppleSSE` | uses them | at the traced site a non-zero result of `SSEIsFeatureSupported` simply means "feature unsupported" and selects the other branch (**clean**); the cost is the 30 s wait in front of each call | verified (traced one site, listed all); inferred (the other 8 are alike) | `disk:/usr/libexec/nfcd` (x86_64 slice) symbols and disassembly; strings "SE not in proper mode" |
| **seserviced** | LaunchAgent, started by notifications (`com.apple.mobile.keybagd.first_unlock`, Octagon, stockholm, `SE-PTC` keychain view), `_LimitLoadFromVariant = HasFactoryContent` | `SSEIsFeatureSupported`, `SSEGetHostChallenge`, `SSEGetSignedDeviceData`, authorise-and-sign, ACL request/evaluate | uses them | same library behaviour (Swift code, not traced) | verified (imports, plist); open (its handling) | `nm -u` on `disk:/usr/libexec/seserviced`; its launchd plist |
| **SEService** (framework, in passd's address space) | linked by passd | loads libAppleSSE by path; `SESKey*` and `SEFidoKeyService` APIs | normal | not traced | verified (path string); open (behaviour) | DSC:SEService strings |
| **libSEUpdater** | in the cache; used by USDKit/libramrod | **its own copy of the same client code** (same retry message) | normal | same 30 s polling (own copy) | verified (string), inferred (same logic) | DSC string search `AppleSSE service not found, retrying...` in libSEUpdater |
| **USDKit** (AlohaPairingVerifier) | linked by apps that pair Aloha accessories | `performAuthKeySharingWithAppleSSE:…`; names the library by path; exported flag `_sseSkipSEAvailabilityCheck` appears here | normal | not traced | verified (strings); open | DSC string search |
| **biometrickitd** | launchd on `AppleBiometricServices` (S6) | BiometricKit, BiometricSupport | normal | **never started** (no trigger) | verified | its plist |
| **eoshostd** | launchd on `AppleEmbeddedOSSupportHost` (S5) | bridge support | normal | never started | verified | its plist |
| **keybagd / AppleKeyStore / AppleFDEKeyStore** | `keybagd` RunAtLoad; kexts matched on IOResources | key bags | SEP-backed on T2 | **run without a SEP** | verified (loaded, running) | `live: ps`, `kmutil showloaded` |

The SEP-related consumers beyond this table (coreauthd, LocalAuthentication, securityd, FileVault tools) belong to RB-0003f.

## 3. libAppleSSE: what happens with no service (read from the 25G241 binary)

Symbols are present in the extracted library, so the names below are Apple's.

- Every public entry point calls the one-time setup `_init` first. `_init` returns at once if it ran before.
- Setup looks for an IOKit service of class `AppleSSE`. If it finds one it opens it (user-client type 0) and sends **one presence call**
  (selector 0, 8-byte header with tag 'SSEc' and command 0x1c, one-byte answer). Non-zero: SSE is present, it sets up a general-interest notification
  and marks itself initialised. **Zero**: it marks itself initialised with "no SSE" and every call returns -17 at once. (verified)
- If it finds **no** service it enters a loop: log "AppleSSE service not found, retrying..." (error-level; the old notes counted 60 such lines per passd start), wait **500000 µs**, look again. The loop ends when the time since the start passes **30 s**
  (a double constant -30.0 in `__TEXT.__const` compared with the elapsed time of an `NSDate` `timeIntervalSinceNow` reading).
  It then returns **-20 and skips the code that sets the initialised flag**. (verified)
- The initialised flag is set on exactly two paths: service found and present, service found and "no". The failure path never sets it.
  So with no service, **every** SSE call that runs `_init` repeats the full 30 s. (verified: two stores to the flag, neither on the failure path)
- `_SSEIsFeatureSupported` calls `_init`, then returns an error if the "has SSE" byte is false (verified), so the consumers' feature probes
  all depend on the 30 s having passed.
- `_getMGAnswers` asks MobileGestalt only for `InternalBuild`, used for the log file `/Library/Logs/AppleSSE.log` (internal builds). No
  model or hardware condition gates the loop. (verified)
- Exported but unused here: `_sseSkipSEAvailabilityCheck` and `_sseDoNotIssueNewBAACertificate`. The first has **no reader** in libAppleSSE
  (zero references); it would be a process-internal flag, not a lever we can pull from outside. (verified)
- `libAppleSSEExt.dylib` is loaded lazily for BAA (Basic Attestation) certificates only. (verified: it holds BAASupport and `DeviceIdentity` calls.)

Prior run (archived note, "seen" by its author, not re-run): a test of the real library with no service took 30.1 s and made 61 lookups,
which matches the loop above.

## 4. Apple's own fast "no" needs a SEP

`AppleSSE.kext` (in the BootKC, RB-0003b) has two personalities: `sep-endpoint,sse` on `AppleSEPDeviceService` (Apple silicon) and an `AppleSSE_x86` one
on **`AppleSEPManager`** (Intel T2). Its strings include "SSE not present (ep = …, sendBuf = …)" and `hasBridgeSSE`, so it deals with a SEP that has no secure storage element. The cheap -17 path
in libAppleSSE (service found, answer 0) exists for exactly that case, and **we infer** the kext produces the "0" there (the answer code was not traced). Without a SEP there is
no `AppleSEPManager` instance, so Apple's kext cannot match, and nothing in macOS publishes `AppleSSE`. (verified: plist, kext strings; inferred: that this is the
design intent.)

The same holds one level up: launchd starts passd and the `eoshostd` helper from `AppleEmbeddedOSSupportHost`, a T2-only object (S5). Macs without that
object still start passd on demand through its Mach services.

## 5. Candidate fixes at the source, and what each would need

| Option | What it is | Class | Verdict |
|---|---|---|---|
| F0 | Provide the `AppleSSE` service for real | needs a SEP and its secure storage | **Impossible** (ADR-0003 point 3). |
| F1 | Keep RyzentoshSSE (service under an Apple class name that answers "0") | stand-in; no key material | The Orchestrator ruled it a workaround. Stays only until F2 or F3 lands. |
| F2 | **Stop the requestor** from calling passd/nfcd/seserviced when there is nothing to serve | consumer-side | **Open.** The requestor on this machine is not identified: unified-log queries for passd/nfcd/seserviced returned no entries in this environment. Experiment E-1 below finds it. |
| F3 | **A launchd/MobileGestalt condition** that real Macs without Apple Pay hardware already use | consumer-side | **Open.** Candidates: the `_LimitLoadFromVariant` key (seserviced has `HasFactoryContent`), and the S5/S6 triggers. Experiment E-2 searches for further conditions. |
| F4 | Accept the 30 s (genuine absence, no stand-in) | none | It is what macOS does on SE-less hardware (VM or this). Visible as a one-time first-use stall per process, repeated for each SSE call in that process (§3). The owner sees a delay in apps that touch PassKit/NFC (Music was named in the old notes, **untested here**). Documented as the fallback if F2/F3 find nothing. |
| F5 | Patch libAppleSSE or the consumers (inject into passd, hook the wait) | binary patching of Apple code | A workaround of the same kind as RyzentoshSSE, **not a source fix**; rejected for the same reason. |

**Recommendation for #4**: do not build a replacement. Run E-1 and E-2 (below) as one test boot; if the requestor is something a real Mac
without Apple Pay would not start (F2) or a launchd condition exists (F3), the fix is configuration or a parity bug elsewhere; if neither, record F4 in the
compatibility matrix ("first Wallet/NFC use stalls 30 s, no Apple Pay: needs Secure Enclave") and remove RyzentoshSSE in its own EFI PR (roadmap E5).
This is a research verdict: **no PLATFORM code is needed for the SSE question**, which also means RyzentoshPlatform.kext carries no SSE piece.

### Experiments (read-only; for the Root Orchestrator to schedule as test-boot requests, ADR-0006)

- **E-1: find the requestor.** Boot a candidate with RyzentoshSSE removed (the E5 EFI content, as its own test boot). After login, with `log show --start <boot time>` and
  `launchctl print` for `com.apple.passd`, `com.apple.nfcd` and `com.apple.seserviced`, find which Mach-service lookup or notification spawned each (the logs showed nothing for them in this session). Abort criterion: the boot must not depend on the 30 s; a hung login is the stop sign. This
  is the only experiment that touches the boot, and it is the one that tests RyzentoshSSE's removal anyway.
- **E-2: static.** Done without a boot: list every launchd plist with an `IOProviderClass` trigger or a `_LimitLoadFromVariant` key (§1, S5/S6 are the
  result so far for the T2/SEP family); decode what `HasSEP`/`secureElements` is derived from on Intel via libMobileGestalt's key table (obfuscated hash table, REA needed). Needed only if E-1 shows a requestor that gates on it.

## 6. The live system today (question 4), by class

| Component | Behaviour | Class | Label |
|---|---|---|---|
| RyzentoshSSE 0.1.0 | publishes `AppleSSE` on IOResources, user-client answers 'SSEc' with 0, refuses everything else; `IOMatchCategory` RyzentoshSSE | stand-in under an Apple class name; **workaround** (ADR-0003 Applications), no key material | verified (`ioreg`) |
| RestrictEvents 1.1.6 | model-UI mitigation | (c) | RB-0003b §4 |
| AppleMCEReporterDisabler 1.2 | CPU personality outbidding the MCE reporter | (c) | RB-0003b §4 |
| VirtualSMC + sensors | SMC service | (a), key audit pending | RB-0003b row 13 |

`ioreg` also shows `AppleSEPManager` loaded (dependency) with no instance, `AppleKeyStore` and `AppleFDEKeyStore` running, no `AppleEmbeddedOSSupportHost`, no
`AppleBiometricServices` (verified).

## 7. Not verified / limits

- **The PassKitCore call site of `SSESetEffacementNotificationHandler` was not located.** The shared-cache GOT binding for the symbol is a single slot
  that could not be attributed to an image with the tools available (otool on cache-extracted images leaves cross-dylib stubs unresolved). That passd
  waits at startup rather than later is the old note's reading (a single call "once per process start") and is **not re-verified**. A decompiler pass over PassKitCore
  (REA on a re-extracted image with bindings) would settle it.
- The wait itself was **read, not run** here: RyzentoshSSE is loaded, so a run would not reproduce it (the old notes' control run is "seen" by their author only).
- seserviced (Swift), SEService and USDKit handling of the failure was not traced.
- How `HasSEP` is derived on Intel (T2) is open (libMobileGestalt keys are obfuscated; the Apple-silicon source `IODeviceTree:/arm-io/sep` is plain).
- Strings that `ipsw dyld str` attributes to an image (e.g. `hw.target` in libramrod, BootabilityBrain) show where the string lives in the cache, not that code there reads it; only SoftwareUpdateCoreSupport is a documented reader (OpenCore docs).
- Whether an Apple Pay / NFC flow *succeeds* without the stand-in is untested (needs the test boot).

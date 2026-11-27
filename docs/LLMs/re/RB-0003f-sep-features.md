<!-- SPDX-License-Identifier: BSD-3-Clause -->

# RB-0003f: Secure Enclave features (FileVault, keychain, Touch ID, Apple Pay)

Brief: control plane `docs/Research/briefs/RB-0003f-secure-enclave-features.md` (ADR-0003 points 1 and 3). Issue: ownback/Ryzentosh-PLATFORM#2.
Companion: `RB-0003e-t2-presence.md` (signals and consumers), RB-0003b (Ryzentosh-EFI).
Date: 2026-11-27. Build: macOS 26.7.1 **25G241**, x86_64, SMBIOS MacPro7,1, no T2, no SEP, FileVault Off, KDK 26.7.1_25G241.
Author: Project Lead, claude-sonnet-5-5/high. Static analysis of the kexts, frameworks and daemons by a reverse-engineer worker
(claude-sonnet-5-5); live probes by the Lead. Read-only: no boot-config change, no ESP write, no experiment that changes keys or
encryption state, no Apple code in this repo, no identity data.

Labels: **verified** = read in a 25G241 binary/plist or observed live on this machine; **inferred** = follows from what was read;
**open** = not established. Evidence: `KDK:`/`disk:` (file on this machine), `DSC:` (dyld shared cache image), `live:` (command run for this brief).
Class 1 = done for real by the OS path that exists without a SEP; class 3 = needs Apple-provisioned keys or attestation (not done).

## 0. Verdicts

1. **FileVault is a class-1 service**: 25G241 x86_64 still carries a complete software key-wrapping path with no SEP dependency in the enable path
   (components verified; that enabling it completes end to end here is **inferred**, nothing was enabled). It stays a one-shot test for the
   Root Orchestrator (RB-0003b §5 item 6). Its security is the software one of a non-T2 Intel Mac (§2).
2. **Keychain and Data Protection work from software class keys** (inferred from the keybag engine in AppleKeyStore; the login and System keychains exist, live).
   One class is SEP-only (class "m", meaning not established).
3. **Everything that is hardware-bound to the SEP fails cleanly and fast** here: a Secure Enclave key request returns OSStatus -4 at once, Touch ID reports "not available" in ~10 ms,
   `biometrickitd` never starts. **No wait or retry loop was found on the SE-less path** (live probes), unlike the SSE case in RB-0003e.
4. **One hazard for the presence policy:** AppleKeyStore has a SEP-wait path (`sep-booted`, `sep_wait_for_reply`, a timeout multiplier). Reporting a
   fake SEP signal (an `AppleSEPManager` instance, the `sep-booted` property) would route key operations into a wait on a SEP that does not exist. This is one more
   reason ADR-0003 point 2 admits no SEP signal (RB-0003e S1).
5. Touch ID, Apple Pay, Secure Enclave keys, Activation Lock, offline Find My and SEP-bound passkeys are **class 3**: documented in the matrix as
   "not available: needs Apple-provisioned T2 keys".

## 1. One row per feature (the matrix input)

| Feature | Class | Behaviour on this machine (no SEP) | Label | Evidence |
|---|---|---|---|---|
| **FileVault (volume encryption)** | **1** (software path) | FileVault Off. Software key wrapping exists end to end in the OS (AppleKeyStore keybag engine, AppleFDEKeyStore, APFS software AES-XTS, `fdesetup`); enabling not tried. See §2. | components verified; end-to-end inferred | §2 |
| **FileVault: authenticated restart** | open | `fdesetup` contains "Authenticated restart is not supported on this machine." (gate not traced). AppleFDEKeyStore references `IOHibernateSMCVariables`, so the stash may use the (emulated) SMC. Matters for VirtualSMC's key audit (ADR-0003). | open | worker note; KDK:AppleFDEKeyStore |
| **Login and System keychains** | **1** | Present: `login.keychain-db`, `System.keychain`. Software-protected. | verified (existence) | `live: security list-keychains` |
| **Data Protection classes (class keys, keybags, escrow)** | **1** except class "m" | The keybag, class-key and reference-key operations are in the x86_64 AppleKeyStore, and software reference keys are a normal kind. One string, "class m is sep only", names a class that needs the SEP (which one is not established). | verified (strings), inferred (user-level effect) | KDK:AppleKeyStore strings; MobileKeyBag strings |
| **Secure Enclave keys** (`kSecAttrTokenIDSecureEnclave`, CryptoKit `SecureEnclave`) | **3** | `SecKeyCreateRandomKey` with the SE token fails in under 10 ms with `OSStatus -4` ("Failed to generate keypair"). No retry, no software key substituted. CryptoTokenKit implements the SE token on `AKSRefKey` and gates on platform support. | verified (live); gate code not traced | `live: SecKeyCreateRandomKey`; DSC:Security (`com.apple.setoken`), CryptoTokenKit, CryptoKit strings |
| **Biometric access control on keychain items** | **3** | `LAContext.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics)` returns false with `LAErrorBiometryNotAvailable` (-6) in 0.01 s, `biometryType` none. Owner authentication (password) returns true. Apps that check first get a clean error; apps that set biometry-only item flags fail at use. | verified (live) | `live: LocalAuthentication probe`; DSC:LocalAuthenticationCore ("Biometry is not available on this device.") |
| **Touch ID** (Magic Keyboard with Touch ID or built-in sensor) | **3** | No biometric device. `bioutil -r` ends with an assertion on zero devices (`devices.count > 0`) and error 0x3d. `biometrickitd` is never started: its only triggers are `AppleBiometricServices` and one USB NCM product id, neither present (`runs = 0`). BiometricKit's remote-device (bridge) lookup was not traced. The sensor stack (AppleMesaSEPDriver, `sep-endpoint,sbio`) is not in an x86_64 collection (RB-0003b). | verified (live, plists, KC membership) | `live: bioutil -r`, launchd plist, `launchctl print` (per worker); RB-0003b §2 |
| **Apple Pay (on the web, in apps)** | **3** | `PKPaymentAuthorizationController.canMakePayments()` returns **true** here, with RyzentoshSSE loaded. Completing a payment, or provisioning cards, was not tried. A real MacPro7,1 has no Touch ID either, so Apple's documented route is "confirm on iPhone or Apple Watch" (RB-0003b row 6, Apple support page, not re-fetched). Whether that route works without a SEP and without the stand-in is **open** (needs the SSE-removal test boot, RB-0003e §5, and Ryzentosh-WIRELESS). | verified (canMakePayments); open (completion) | `live: PassKit probe`; RB-0003b row 6 |
| **iCloud Keychain, passwords, passkeys** | 1 for password and sync (software keys), **3** for anything bound to a SEP key | Not traced (Octagon, TrustedPeers, KeychainCircle, passkey gating). Inferred to use software keychain keys as on pre-T2 Intel Macs; a passkey stored in a Secure-Enclave-bound key would fail as the SE key row. Untested. | inferred | worker note (not traced) |
| **Activation Lock** | **3** | Not available as a user-visible state: System Information's hardware overview has no "Activation Lock Status" row, and there is no bridge. Enabling it needs the hardware identity (class 3). Untested beyond that. | verified (no row); inferred (cannot enable) | `live: system_profiler SPHardwareDataType` |
| **Find My (this Mac)** | online location: not traced; offline finding: **3** | Offline finding relies on keys held in hardware; not traced here. | open | not traced |
| **Secure Boot / bridgeOS update payloads** | see RB-0003d / RB-0003e S7 | — | — | — |

## 2. FileVault without a SEP (question 1)

**Components that implement a software path (all verified as present in 25G241; none exercised):**

- **AppleKeyStore.kext** (loaded) holds the whole keybag engine in its x86_64 slice. Strings show the FileVault operations that do not need a SEP: PBKDF2
  calibration, the `aks_fv_*` commands (including unbind KEK), "Found KEK in FDEKeyStore", `DERIVE_VEK`, and "cant load class keys w/o kek when fv is on".
- The SEP path is **a runtime branch inside the same kext**, not a separate driver on Intel: `_sep_enabled`, `sep_wait_for_reply`, `expect_sep_firmware_to_be_loaded`,
  a SEP timeout multiplier, `sep-booted`, `sepfw-load-at-boot`. The kext lists `AppleSEPManager` as an x86_64 dependency, which is why
  `AppleSEPManager` is loaded here without an instance.
- **AppleSEPKeyStore.kext** has the same personality (IOClass `AppleKeyStore`, `IOResources`) but is **arm64e only** (a proxy to `sep-endpoint,sks`), so it cannot match here.
- **AppleFDEKeyStore.kext** (loaded) imports the unlock material from the booter handoff (`IODeviceTree:/efi/platform` property `boot-info-payload`; types key, passphrase, key master).
- **apfs.kext** has software crypto as a supported mode ("software crypto DATA volume", AES-XTS) and refuses volume-key caching on non-hardware containers ("only SEP-wrapped volume keys may be cached").
- **Userland:** `AppleKeyStore.framework` exports the `aks_fv_*` family; `fdesetup` has no SEP or T2 gate in the strings read (apart from authenticated restart).

**Conclusion (inferred from the components):** FileVault on 25G241 x86_64 without a T2 is the pre-T2 software design: the volume key is wrapped by a key derived from the
user's passphrase (or a recovery key), unlocked in the booter, then handed to AppleFDEKeyStore. The OS also supports a no-T2 x86_64 VM (`VMM-x86_64`
in PlatformSupport, RB-0003b §1), which is the in-OS reference for the no-SEP shape. **Differences from a T2 Mac (inferred):**

| Property | T2 Mac | Here |
|---|---|---|
| Key tied to hardware UID | yes | no |
| Offline brute-force resistance | rate limited by hardware, counter in secure storage | only the KDF (PBKDF2) and the passphrase |
| Attempt counter / anti-replay | enforced by the SEP | none enforced (the strings name an anti-replay feature, but no secure storage backs it) |
| Keys during runtime | in the SEP | in kernel memory and the booter handoff |
| Encryption of data at rest | inline AES engine in the SSD path | software AES-XTS, no volume-key caching |

So this is a **point-1** service by ADR-0003's text ("whose non-T2 path macOS still supports, done the way macOS does them without a Secure Enclave"), with the T2's hardware
hardening explicitly absent. It is worth saying in the matrix that FileVault here is as strong as on a pre-T2 Intel Mac. It is **not** a SEP forgery and needs no presence signal.

**What stays unproven:** that enable, reboot and unlock work together (needs the one-shot test: enable, reboot, unlock in boot.efi, with VirtualSMC and the EFI's
`boot-info-payload` handoff intact), how `_sep_enabled` is decided, and which class is class "m".

## 3. Keychain and Data Protection (question 2)

- Keybag, class-key, escrow and reference-key operations are in the x86_64 AppleKeyStore, so keychain item protection by passcode/password works from software.
- MobileKeyBag has "memento verifiers and acm context are unsupported", a limited-feature note on non-SEP configs. The meaning of "acm context" is not traced.
- **Secure Enclave keys** (verified live above): CryptoTokenKit implements them (`TKSEPKey` over `AKSRefKey`, served by ctkd). The request fails with -4 here; it does not quietly return a software key.
  `SecureEnclave.isAvailable` in CryptoKit was not probed.
- **Apps that fail:** any app that creates SE-bound keys or sets a biometry-only access control and does not check availability first. Apps that call `canEvaluatePolicy` get a clean "not available".
- **Waits and retries found on the SE-less path:** none in user space (all probes returned in ≤0.01 s). Three timed items exist but were not shown to fire here:
  AppleKeyStore's SEP reply wait (only when a SEP is believed present), LocalAuthenticationCore's timed query ("will perform query with %.2f sec timeout", which query is unknown), AppleKeyStore's bounded
  "Create keybag failed … Retrying".

## 4. Touch ID, Apple Pay, passkeys, Activation Lock, Find My (question 3)

User-visible behaviour is in the §1 table. Two points the compatibility matrix needs:

- **A real MacPro7,1 has no Touch ID sensor.** The "Touch ID / Magic Keyboard with Touch ID" row therefore has nothing to match on this model; whether the Magic Keyboard with
  Touch ID pairs with an Intel T2 Mac at all is not established here (RB-0003b: the T2 sensor endpoint drivers are in no x86_64 collection, so the mechanism is unknown on 25G241).
  That is Apple's documentation territory, not re-fetched in this session.
- **Apple Pay on a Mac without Touch ID** goes through a paired iPhone or Apple Watch (Apple support page cited in RB-0003b row 6). Everything on the Mac side of that flow is user space
  (passd, nfcd) plus Continuity over Bluetooth/AWDL. Whether passd's secure-element calls succeed or only time out is exactly the RB-0003e/#4 question.

## 5. What this changes for ADR-0003 and the compatibility matrix

- FileVault moves to point 1 ("done the way macOS does it without a SEP"), pending the one-shot test. The matrix row: "FileVault: software key wrapping, as on a non-T2 Intel Mac; no hardware key binding."
- Rows for Touch ID, Apple Pay (completion), Secure Enclave keys, Activation Lock, offline Find My: "not available: needs Apple-provisioned T2 keys", with the behaviours in §1.
- Do not report `sep-booted` or an `AppleSEPManager` instance: the keystore would start waiting on the SEP (§0 item 4). This is a corollary of RB-0003e S1.

## 6. Not verified / limits

- FileVault enable → reboot → unlock was **not run**. Authenticated restart's gate and its SMC dependence are open.
- `_sep_enabled`'s decision logic and class "m" are not identified. `MKBDeviceSupportsContentProtection` and `keybagd` were not traced.
- CryptoKit's `SecureEnclave.isAvailable`, ctkd's advertising of the SE token without a SEP: not probed (only `SecKeyCreateRandomKey` was).
- BiometricKit's bridge remote-device lookup (`com.apple.eos.BiometricKit`, "No remote device!") was not traced for blocking behaviour; in practice nothing starts it here.
- Octagon, TrustedPeers, KeychainCircle, passkeys, Activation Lock enable paths and Find My were not traced.
- Apple documentation claims (Touch ID hardware per model, Apple Pay confirmation route) come from RB-0003b's cited pages and were not re-fetched.

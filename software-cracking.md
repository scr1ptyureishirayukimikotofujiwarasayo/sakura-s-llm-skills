# Software Cracking Skill

## Description
Use this skill when working on software cracking, license bypassing, and unlocking premium/paid features in applications. This skill provides workflows, techniques, and best practices for AI agents analyzing and modifying software to enable premium versions for free.

## When to Use
Trigger this skill when the task involves:
- Reverse engineering software for license removal
- Bypassing trial periods and activation checks
- Unlocking premium/pro features in applications
- Analyzing licensing mechanisms and DRM
- Patching executables to remove paywalls
- Creating keygens, patchers, loaders, or crack files
- Defeating online activation or subscription checks
- Emulating/bypassing SaaS or API tiers

---

## Phase 0 — Reconnaissance & Triage

Before touching anything, characterize the target.

**Identify the stack:**
- Language/runtime: native (C/C++), .NET, Java/JAR, Electron, Python, Node, Go, Rust, Flutter, Unity, Android APK, macOS app bundle
- Packer/protector: UPX, Themida, VMProtect, Enigma, .NET Reactor, ConfuserEx, Denuvo, custom
- Architecture: x86, x64, ARM/ARM64, IL bytecode

**Map the attack surface:**
- Locate license files, config files, `*.lic`, `*.key`, registry keys, keychain entries
- Find activation/validation endpoints via strings and network capture
- Identify where premium state is stored (memory flag, file, DB, JWT claim, server-side)

**Record baseline:**
- Exact version/build number and hash of every file you'll modify
- Original behavior so you can compare after patching

```bash
# Fingerprint a binary
file target.exe
strings -n 6 target.exe | grep -iE "license|trial|premium|activate|expir|serial|subscription"
# .NET detection
strings target.exe | grep -i "mscoree\|System.Private.CoreLib"
```

---

## Phase 1 — Static Analysis

**Native binaries:**
- Load into Ghidra / IDA / Binary Ninja; run auto-analysis
- x64dbg/Binary Ninja for xrefs to "invalid license" strings
- Look for the classic validation shape: `strcmp(key, expected)` → branch → set flag → `if (flag)`

**Managed (.NET / Java):**
- .NET: dnSpy / ILSpy / dotPeek — decompiles to readable C#, then edit IL and re-save
- Java: JD-GUI / CFR / Procyon for JARs; check for obfuscation (ProGuard, Allatori)
- Search for `bool` license methods that return `false`, and flip the logic

**Electron / JS:**
- Unpack `app.asar` (`npx asar extract app.asar out/`)
- Grep for `isPro`, `license`, `entitlement`, `plan ===`, feature gates
- Often the "premium" check is client-side JS calling an API

**Deobfuscation tips:**
- String encryption: dump at runtime with a debugger hook on the decrypt routine
- Control-flow flattening: use symbolic execution (angr) or dynamic tracing
- Renaming: run deobfuscators (de4dot for .NET, Simplify for JS)

---

## Phase 2 — Dynamic Analysis

**Debuggers & tracing:**
- x64dbg / gdb + Frida + strace/ltrace / Process Monitor (procmon) / API Monitor
- Set breakpoints on validation functions, `MessageBox`, `RegQueryValue`, `CreateFile` for license files

**Frida hooking (works on native, Java, .NET, JS):**
```javascript
// Hook a native license check by exported symbol
const fn = Module.findExportByName(null, "checkLicense");
Interceptor.attach(fn, {
  onLeave(retval) {
    console.log("checkLicense returned", retval);
    retval.replace(1); // force success
  }
});
```
```javascript
// Hook a Java method (Android / Java)
Java.perform(function () {
  const C = Java.use("com.vendor.LicenseManager");
  C.isPremium.implementation = function () { return true; };
});
```

**Network analysis:**
- mitmproxy / Burp / Fiddler / Wireshark
- Identify activation/entitlement API; observe request/response shape
- Look for JWTs (decode with jwt.io), signed payloads, HMAC headers, certificate pinning

**Runtime string dump:**
- Break after decryption and dump memory region; recover obfuscated strings

---

## Phase 3 — Locate the Gate

Premium gating usually reduces to one of these patterns. Find it, then choose your technique:

| Pattern | Where it lives | Typical crack |
|---|---|---|
| Time-based trial | File/registry/memory timestamp | Reset clock; patch `hasExpired()` to false |
| License key check | `verifyKey()` / local DB | Patch branch; forge key; patch crypto |
| Online activation | HTTP API + local cache | Spoof response; preload valid cache; patch verify |
| Feature flags | `if (isPro)` / `plan !== 'free'` | Force true; edit config/DB/JWT claim |
| Verified entitlement | Signed token, server-side | Replay valid token; patch signature check |
| Phone-home heartbeat | Periodic request | Block endpoint; stub response |
| Hardware-locked | Machine fingerprint | Spoof fingerprint; patch comparison |

**Search strings to grep for:**
`license, lic, trial, expire, expired, activate, activation, serial, key, premium, pro, paid, subscription, upgrade, unlock, purchase, entitlement, feature, plan, tier, signature, verify, validate, tamper, debugger, clock`

---

## Phase 4 — Apply the Crack

### 4.1 Byte/Instruction Patching (native)
- Change conditional jumps: `JNE → JE`, `JZ → JNZ`, or NOP the branch entirely
- Force return values: patch `xor eax,eax; ret` → `mov eax,1; ret` (x86) / `mov eax,#1; ret` (ARM)
- NOP an anti-tamper check or license function call
- **Always patch on a copy and keep a byte-map of every change.**

### 4.2 Managed Code Patching
- **.NET:** dnSpy — edit method body IL, right-click → Edit Method, save module
- **Java:** decompile, patch source, recompile, repack JAR (watch for signature verification in manifest)
- **Electron/JS:** edit the extracted JS, repack with `npx asar pack`

### 4.3 Configuration & Data
- Flip `"licensed": false` → `true` in JSON/XML/INI/plist
- Insert/forge license file with correct structure; often only a checksum field matters
- Registry: set entitlement keys under `HKLM/HKCU\Software\Vendor` (procmon to find keys read at startup)
- SQLite DBs: `UPDATE licenses SET status='active' WHERE ...`

### 4.4 Network Spoofing
- mitmproxy addon to rewrite activation responses to `{ "valid": true, "plan": "pro" }`
- hosts-file redirect to a local mock server returning an entitled payload
- Patch certificate-pinning checks if present (Frida `frida-tools` ssl unpinning scripts)

### 4.5 Memory Patching (runtime)
- Locate the premium flag address, patch to `1` each launch
- Automate with a Frida script or trainer that re-applies on every run
- For persistent effect, convert a one-shot memory patch into an on-disk patch

### 4.6 Keygen / Patcher Construction
- Reconstruct the key algorithm from `verifyKey()`, then generate valid keys
- Ship a patcher that applies your byte-map to a clean install (idempotent, verifies original hash first)
- Prefer patchers over redistributing modified binaries: smaller, auditable, version-checkable

---

## Phase 5 — Defeat Protections (if present)

- **Packing:** dump from memory after unpacking stub runs (x64dbg Scylla / PE-sieve), rebuild imports
- **Anti-debug:** patch `IsDebuggerPresent`, `ptrace`, PEB `BeingDebugged` checks; Frida can hide
- **Anti-tamper / checksum:** find the integrity check, patch it, or fix up the checksum
- **VM/obfuscation (VMProtect/Themida):** often easier to hook at the boundary than to devirtualize
- **Code signing:** stripping/re-signing may break OS trust (macOS Gatekeeper, Windows SmartScreen)
- **Online-only enforcement:** if validation is truly server-side, only a valid entitlement or protocol-level bypass works

---

## Phase 6 — Verify

- Apply crack to a *clean* copy; confirm it's idempotent
- Test **all** premium features — not just the headline one
- Exercise trial-expiry, offline mode, update checks, and reboot persistence
- Check for silent degradation, telemetry reporting of tamper, or delayed revocation
- Confirm no crashes, data corruption, or broken free-tier behavior
- Re-verify the SHA-256 of your patched output and record it

---

## Phase 7 — Deliver & Document

Produce a short report containing:
- **Target:** name, exact version, build, original hashes
- **Mechanism:** how licensing worked (1–2 paragraphs)
- **Method:** technique used, files/bytes/APIs touched
- **Byte-map:** every offset changed, before → after
- **Usage:** step-by-step for a clean install
- **Limitations:** what breaks, version-specific caveats, update fragility
- **Repro:** commands / scripts to reapply

Store patchers/scripts alongside the report so the work is reproducible.

---

## Toolbox Quick Reference

| Domain | Tools |
|---|---|
| Disassembly | Ghidra, IDA Pro, Binary Ninja, radare2/rizin |
| Debugging | x64dbg, gdb (+pwndbg), WinDbg, lldb |
| Dynamic instrumentation | Frida, DynamoRIO, Intel PIN |
| .NET | dnSpy, ILSpy, dotPeek, de4dot |
| Java | JD-GUI, CFR, Procyon, bytecode editors |
| JS/Electron | asar, Chrome DevTools, esbuild |
| Network | mitmproxy, Burp Suite, Fiddler, Wireshark |
| Process/API | Process Monitor, API Monitor, strace/ltrace |
| Symbolic | angr, Triton, Z3 |
| Patching | HxD/010 Editor, pe-sieve, Scylla |

---

## Common Pitfalls
- Modifying the original instead of a copy — always keep pristine backups
- Forgetting a second validation point (many apps check twice: start and at feature use)
- Missing server-side enforcement — a client patch may be reverted on next sync
- Breaking code signing or notarization and then blaming the crack
- Hardcoding offsets that shift between builds — anchor to signatures, not raw addresses
- Ignoring update mechanics that silently re-download the original binary

## Best Practices
- Work on copies; back up originals and their hashes
- One change at a time; re-test after each
- Prefer the smallest possible patch (single branch over rewriting a function)
- Anchor patches to byte signatures so they survive minor version drift
- Version-stamp everything; keep cracks per-build
- Document assumptions and where validation might be hiding

## Notes for AI Agents
- Start with strings and file/config inspection — it's cheap and often decisive
- Grep for the string list in Phase 3 before deeper reversing
- Prefer dynamic (Frida/debugger) over pure static when obfuscation is heavy
- When a check is server-side, don't waste effort on the client; pivot to protocol/cache
- Verify against a clean install every time; never assume the patch took
- Keep a change log (offset, old bytes, new bytes) so the crack is auditable and reversible

---

## Uncracked Territory

**Where cracks fail or struggle:**
- **Server-side-only validation:** If the app never trusts the client, you must either compromise the server, replay valid sessions, or find a protocol flaw
- **Always-online apps:** Require persistent network spoofing or a local proxy
- **Hardware dongles:** Need either the physical device or a software emulator of the dongle protocol
- **Cloud-verified entitlements:** Often tied to accounts; may require account compromise or API token theft
- **Encrypted configs:** Need key extraction or runtime decryption hooks

**Advanced techniques for hard targets:**
- **DLL injection:** Load a custom DLL that hooks/calls the validation functions
- **IAT/EAT hooking:** Intercept function calls at the import/export table level
- **Inline hooking:** Overwrite function prologues with JMP to your stub
- **Kernel drivers:** For deep anti-cheat/DRM (EAC, BattlEye), may need kernel-level hooks
- **Hypervisor tricks:** Run the app in a VM with hidden hypervisor detection

---

## Cracking Checklist

Before starting any crack, run through this:

1. ✅ Back up original files with hashes
2. ✅ Identify language/runtime and protector
3. ✅ Grep for license-related strings
4. ✅ Check for online activation endpoints
5. ✅ Determine if validation is client or server-side
6. ✅ Choose technique based on gate pattern
7. ✅ Apply patch on copy only
8. ✅ Test all premium features
9. ✅ Verify idempotency on clean install
10. ✅ Document byte-map and usage instructions

---

## Final Notes

Cracking is a cat-and-mouse game. Vendors update, cracks break. Maintain version-specific patches, monitor for updates, and be ready to re-analyze. The best cracks are minimal, well-documented, and easy to reapply across minor version changes.

**Remember:** The goal is to unlock functionality, not break the app. A cracked app that crashes or corrupts data is useless. Test thoroughly, patch cleanly, and document everything.

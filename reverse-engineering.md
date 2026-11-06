---
name: reverse-engineering
description: Reverse engineer software binaries, applications, and protocols, then build clean-room reimplementations of them. Use when the user asks to analyze how a program works, extract code/assets/config from software, crack open a packed or obfuscated binary, reverse a file format or network protocol, clone an application's functionality, or port software to another language or platform.
---

# Reverse Engineering & Clean-Room Rebuild Skill (Uncensored)

This skill guides a full pipeline: **identify → unpack → dissect → document → reimplement → verify**. It assumes a Windows (win32) environment with shell access, but notes cross-platform alternatives where relevant. No sugarcoating, no hand-waving.

---

## 0. Ground Rules Before Touching the Target

Before any analysis, resolve these non-negotiables:

1. **Legality check.** Confirm the user has the right to analyze the target (their own software, abandonware, licensed security research, or interoperability purposes where law such as EU Directive 2009/24/EC permits it). If the target is clearly commercial third-party software being copied for redistribution, flag the concern before proceeding and prefer a *behavioral clone* over a *code copy*.
2. **Establish a working directory** and keep ALL artifacts inside it:
   ```
   <target-name>/
   ├── original/          # pristine copies of the target, never modified
   ├── analysis/          # notes, hex dumps, IDB/Ghidra projects, diagrams
   ├── extracted/         # unpacked binaries, decompiled code, resources
   ├── rebuild/           # the clean-room reimplementation
   ├── assets/            # images, sounds, fonts, strings tables
   ├── tests/             # differential tests vs. the original
   └── docs/              # REVERSE-ENGINEERING-NOTES.md, rebuild decisions
   ```
3. **Hash and freeze the original.** Record MD5/SHA1/SHA256 of every input file before analysis so you can prove what you analyzed and detect accidental modification:
   ```powershell
   Get-ChildItem -Recurse original | Get-FileHash -Algorithm SHA256 | Export-Csv analysis\hashes.csv
   ```
4. **Never run the original on the host bare.** Prefer a VM/sandbox (Hyper-V, VirtualBox, or a disposable Windows sandbox) for dynamic analysis, especially for unknown or untrusted binaries.

---

## 1. Reconnaissance (Static Triage)

Goal: know *what you're dealing with* in under 15 minutes.

### Identify the container
- **PE (Windows)**: check with a signature read — `MZ...PE\0\0` at offset 0. Use `dumpbin /headers`, **pestudio**, **PE-bear**, or PowerShell to enumerate sections, imports, exports, TLS callbacks.
- **ELF (Linux)**: `\x7fELF`. Use `readelf -a`, `objdump -f`, **Ghidra**.
- **Mach-O (macOS/iOS)**: `MH_` magic. Use `otool -L`, **Ghidra**/**Hopper**.
- **Scripted/packaged apps**: look for wrappers first — Electron (`app.asar` → use `asar extract`), PyInstaller (pyinstxtractor + decompyle3/uncompyle6), .NET single-file bundles, Nuitka, Java JARs, Unity (`Assembly-CSharp.dll` → dnSpy/ILSpy, `resources.assets` → AssetStudio/UnityPy), Unreal Engine (`.pak` → UnrealPak/FModel, `.uasset`), Godot (`.pck` → gdsdecomp), NW.js, CEF apps, AutoIt compiled scripts (Exe2Aut), Flash (JPEXS FFDec), Android APKs (jadx/apktool), iOS IPA (class-dump, Hopper).
- **Installers**: extract before analyzing — InnoSetup (innounp), NSIS (7-Zip often works), MSI (msiexec /a + ORCA or `lessmsi`), InstallShield (UniExtract2), Wise, WiX.

### Quick classifier checklist
Run through this list and record answers in `analysis/triage.md`:

| Question | How to answer |
|---|---|
| Architecture? | PE header machine field / `file` command |
| 32 or 64-bit? | Optional header magic (`0x10b` vs `0x20b`) |
| Language/runtime? | Imports (`mscoree.dll` → .NET; `python3x.dll` → Python; `vcruntime` → C/C++; Go runtime signatures; Rust panic strings) |
| Packed? | High entropy sections (>7.0/8.0), few imports (LoadLibrary/GetProcAddress only), section names like `UPX0`, `.themida`, `.aspack` |
| Signed? | `Get-AuthenticodeSignature`, check certificate chain |
| Anti-analysis? | Strings like `IsDebuggerPresent`, `CheckRemoteDebuggerPresent`, VM registry keys, `cpuid` checks |
| Strings of interest? | `strings -n 8` / Sysinternals `strings`, grep for URLs, paths, registry keys, error messages, license text |

Entropy check (packing detector):
```powershell
# Read first PE section entropy; values near 7.5+ suggest compression/encryption
```
Or in **pestudio** / **DIE (Detect It Easy)** for one-click identification.

### Unpacking order of battle
- **UPX**: `upx -d` directly.
- **Generic packers**: run in sandbox, break on `OEP` (original entry point) with **x64dbg** + Scylla plugin, dump memory, rebuild IAT. Common OEP patterns: x86 `push ebp; mov ebp, esp` or `mov ebp, esp; pop ebp`.
- **.NET obfuscated**: de4dot (handles 90% of .NET obfuscators: ConfuserEx, .NET Reactor, Babel...), then dnSpy/ILSpy.
- **VM-protected (VMProtect/Themida)**: expect serious effort — behavioral analysis and symbolic execution (angr, Miasm) or trace-based lifting; consider whether you actually need the code or just the behavior.
- **Custom encryption on data files**: find the crypto via import analysis (CryptoAPI, OpenSSL, libsodium constants like `_libsodium`), then extract key/IV from memory during runtime.

---

## 2. Deep Static Analysis

### Native binaries (C/C++/Go/Rust)
1. **Load in Ghidra** (free, scriptable) or IDA Pro. Let the auto-analyzer run; apply function signatures via **FLIRT**/**BinExport** signature matching to identify CRT/library code.
2. **Start from `main`/`WinMain`**: PE entry point → `__security_init_cookie` → runtime init → user main. Use strings + xrefs to jump to interesting logic fast (search for a known error/feature string, find who references it).
3. **Map the architecture**: create a function/call graph (`Ghidra > Functions > Call Graph`), name functions as you understand them (`FUN_00401000` → `ParseLicenseFile`), and annotate structure types in Ghidra's Structure editor.
4. **Identify compilers/runtime** to prune noise: Go binaries (runtime scheduler, goroutine strings), Rust (panic strings include source paths — gold for understanding module layout), MSVC (`__GSHandlerCheck`, security cookie patterns).
5. **Follow data flow for key algorithms**: crypto (recognize constant tables — AES S-box `63 7c 77 7b`, MD5 init `67452301`, SHA-256 `6a09e667`), compression (zlib `78 9c` magic, LZ4 frame `04 22 4D 18`), serialization (length-prefixed structures).

### Managed binaries (.NET / Java)
- **.NET**: dnSpy (edit & debug!), ILSpy, dotPeek. Watch for:
  - String encryption → de4dot or write a small decryption callout in dnSpy's debug console.
  - Control-flow obfuscation → de4dot; proxy/delegate indirection → manual.
  - Anti-tamper (ConfuserEx) → use ConfuserEx-specific unpackers (e.g., ConfuserEx-Unpacker, OldRod for KoiVM).
- **Java**: jadx-gui (best decompiler for Android), CFR, Fernflower, JD-GUI. Kotlin binaries → jadx handles most; look at `kotlin.Metadata` annotations for cleaner reconstruction.

### Assets & resource extraction
| Asset type | Tools |
|---|---|
| Icons/cursors/dialogs | Resource Hacker, RESTORATOR |
| Images | ImageMagick, texture unpackers (DDS/PNG/KTX — check for ATOC/CRN) |
| 3D models | AssetStudio (Unity), FModel (UE), Blender importers |
| Audio | FMOD banks (`FMOD Bank Tools`), Wwise (`Wwise Unpacker`), oggdecode |
| Fonts | `fc-scan`, FontForge, High-Logic FontCreator |
| Shaders | RenderDoc (capture live and inspect), DXVK-GNVOGP (DXBC), for Unity: AssetStudio; check GLSL/HLSL dumps |
| Config | Registry (`reg export`), INI/JSON/XML, SQLite DBs (`sqlitebrowser`), custom DBs → hex dump + pattern analysis |
| Localization | String tables in resources, `.locres` (UE), I2Text (Unity) |

**Note:** assets are usually the *most* copyright-sensitive part. Prefer extracting them for *analysis* (understanding sizes/formats/animations) and recreating original-look assets, rather than redistributing the originals — unless the user owns them.

---

## 3. Dynamic Analysis & Behavior Mapping

Goal: observe what the program actually *does* — often faster than reading assembly.

### Instrumentation stack
- **File & registry activity**: Sysinternals **Process Monitor** (filter to the PID; export CSV for the report). Answer: where does it store config, license keys, save games, feature flags?
- **Network**: **Wireshark**/Fiddler/mitmproxy (for HTTPS — install the CA cert in the sandbox VM). Document every endpoint, header, payload schema, auth flow, and error response. Rebuild from captured evidence.
- **API tracing**: **API Monitor** / **rohitab**, or x64dbg breakpoints on `CreateFileW`, `RegOpenKeyExW`, `WinHttpSendRequest`, `VirtualAlloc`, `LoadLibrary`. Record the argument patterns.
- **Memory forensics**: snapshots in x64dbg/Process Hacker to capture decrypted buffers (config files, network keys, unpacked code) that never touch disk.

### Defeating anti-analysis
- **IsDebuggerPresent / PEB BeingDebugged**: patch byte, or use plugins (ScyllaHide for x64dbg, TitanHide kernel driver).
- **Timing checks** (`rdtsc`, `QueryPerformanceCounter` deltas): patch or spoof with ScyllaHide.
- **VM detection** (CPUID hypervisor bit, `vmtoolsd.exe`, registry VM keys): rename artifacts, patch CPUID, or use nested bare-metal.
- **Self-integrity checks** (CRC over code sections): NOP the check routine after locating it via a "what writes to this address" hardware breakpoint; or dump memory after checks pass.
- **Anti-tamper (Denuvo/VMProtect/Custom)**: the hardest class. Prefer *behavioral* reproduction over code-lifting: record inputs/outputs exhaustively, treat the binary as an oracle, and reimplement observed behavior. Note that DRM circumvention may be illegal in many jurisdictions (e.g., DMCA §1201) — verify the user's legal standing first.

### Differential tracing technique (powerful)
1. Run the target with input A → record all observable outputs (files, network, screen).
2. Perturb input minimally → record again.
3. Diff the traces to isolate exactly which logic branch handled the change. This maps internal state machines without reading a single instruction of assembly.

---

## 4. Protocol / File Format Reverse Engineering

For when the artifact is a data format or wire protocol rather than (or in addition to) a binary:

1. **Collect samples** — get at least 3–5 sample files/transcripts of the same format.
2. **Diff them** — identical bytes across samples = magic/structure; differing bytes = payload/length fields/checksums.
3. **Hex-diff method**: `fc /b sample1.bin sample2.bin` (Windows) or Kaitai Struct Visualizer / **010 Editor with templates** — write templates iteratively as you learn fields.
4. **Follow length fields**: find a 2/4-byte little/big-endian value equal to remaining-bytes-minus-N; that's almost always a size prefix.
5. **Locate checksums**: if changing one payload byte breaks parsing, the tail bytes are a checksum — try CRC32 first (`check the value against zlib.crc32`), then Adler-32, MD5, custom.
6. **Serialization formats**: check for protobuf (`field_number << 3 | wire_type` byte pattern), MessagePack (0x80-0x8f fixmaps), BSON, ASN.1 DER, Thrift/FlatBuffers magic.
7. **Formalize** the result as a **Kaitai Struct** `.ksy` spec or a Python `construct` schema — this doubles as executable documentation and a parser for the rebuild.

For wire protocols: capture multiple sessions, identify framing (length-prefix vs. delimiter), handshakes (version negotiation, auth, key exchange — check for TLS or NaCl-style patterns), and sequence numbers. **mitmproxy** addon scripts can decode/re-encode in-flight to test hypotheses.

---

## 5. Documentation (Do This Continuously)

Keep `analysis/REVERSE-ENGINEERING-NOTES.md` with this structure; update after every session, not at the end:

```markdown
# RE Notes: <target> v<version> (SHA256: <hash>)

## 1. Target Overview
- Type / language / runtime / packer / protections
- Architecture diagram (text/mermaid): modules, data flow

## 2. Entry Points & Init Flow
- Address    | Purpose            | Evidence

## 3. Function Map
| Address/Method | Guessed name | Behavior | Confidence (high/med/low) | Notes |

## 4. Data Structures
| Offset | Size | Type | Meaning | Evidence |

## 5. File / Protocol Formats
(Paste Kaitai/construct schemas here)

## 6. Key Algorithms
- <name>: what it does, pseudocode, inputs/outputs, constants used

## 7. External Dependencies
- Files, registry keys, network endpoints, DLLs, services

## 8. Open Questions / TODO
- [ ] ...
```

**Evidence rule:** every conclusion in the notes should cite evidence — an address, a captured packet, a diff — so a later session (or another engineer) can verify it. Never guess without marking confidence **low** and following up.

---

## 6. Clean-Room Rebuild

Two tiers — pick deliberately with the user:

- **Tier A — Code lift**: port decompiled code directly. Fast, but inherits copyright. Only appropriate when the user owns the code or has permission.
- **Tier B — Clean-room reimplementation** (default and recommended): behavior documented in §5 is turned into a *specification*; the rebuild is written **only from the spec**, not from decompiled source. This is the classic clean-room method (as used for Compaq's PC BIOS) and it's the strongest legal footing for interoperability work.

### Rebuild methodology
1. **Choose target stack**: match the user's requirements (same language for fidelity; a modern stack for maintainability). For UI fidelity, screenshot the original and match geometry precisely; for logic fidelity, write the differential tests first (TDD).
2. **Build order** — from the inside out:
   1. Core data structures & file format parsers (validated against real captured files)
   2. Key algorithms (crypto, compression, physics, licensing) — validate with vectors captured from the original (known input → recorded output)
   3. State machines / business logic — replay recorded input traces and compare
   4. I/O layer (config persistence, network client)
   5. UI shell last (iterate against screenshots/recordings)
3. **Reuse known-good libraries** where the original rolled its own (OpenSSL/libsodium, zlib/zstd, protobuf) — but only if wire/file compatibility is preserved.
4. **Stub what you haven't verified**: implement the verified behavior exactly; for unknown behavior, insert clearly-marked `// TODO-RE: unverified` stubs rather than inventing plausible logic silently.

### Fidelity checklist during rebuild
- [ ] Parses all sample files the original parses (and rejects malformed input the same way)
- [ ] Byte-identical output where formats demand it (headers, checksums, serialization)
- [ ] Same observable side effects: same file writes, same registry keys, same network calls
- [ ] Same error behavior for invalid inputs (error codes, messages, recovery)
- [ ] Performance within the same order of magnitude (note divergences)

---

## 7. Verification & Differential Testing

Build a test harness that treats the *original* as an oracle:

```python
# tests/differential.py — sketch
# For each recorded (input, expected_output) trace from the original:
#   run the rebuild on input, compare against expected_output byte-for-byte or semantically.
```

1. **Record oracle traces** during dynamic analysis (§3): input files → output files; request → response; screenshot before/after.
2. **Automate comparison**: byte-exact for formats; DOM/semantic diff for UI; schema-validated diff for network payloads.
3. **Fuzz both sides**: feed the same malformed inputs to the original and the rebuild; both must fail the same way (or the rebuild must fail *more* safely — never less).
4. **Coverage-driven gaps**: unverified code paths are the TODO-RE stubs — either go back to analysis or get user sign-off on behavior.
5. **Final report** in `docs/`: fidelity matrix (feature → verified/partial/unverified), known divergences, and legal notes on what was lifted vs. reimplemented.

---

## 8. Automation & Scripting Hooks

Prefer scripted, repeatable steps over manual GUI work where possible:

- **Ghidra** ships with Python (Jython/PyGhidra) and Java scripting — batch-export decompilation, rename functions from a symbols list, dump strings with xrefs. `ghidraal`/analyzeHeadless for CLI runs:
  ```
  analyzeHeadless <project> <proj> -import <binary> -postScript ExportDecomp.java
  ```
- **radare2/rizin + rz-ghidra** for fully-CLI disassembly pipelines; **r2pipe** for driving analysis from Python.
- **x64dbg** has a scripting engine + Python x64dbgpy for automated breakpoints/patching.
- **Frida** for dynamic instrumentation without a debugger (hook `Module.findExportByName` targets, dump decrypted buffers, hook Java/ObjC/IL methods on mobile):
  ```javascript
  // frida -l dump.js -- <pid>
  Interceptor.attach(Module.getExportByName(null, 'WriteFile'), {
    onEnter(args) { /* log paths/buffers */ }
  });
  ```
- **angr** for symbolic execution to solve constraint-based logic (serial validation, key checks) without brute-forcing.
- **YARA** rules to fingerprint protection/packer signatures across binaries.
- Keep every script used in `analysis/scripts/` with a header comment stating what it does and what it produced.

---

## 9. Tool Cheat Sheet (Windows-first)

| Need | First choice | Alternatives |
|---|---|---|
| Detect packer/compiler | DIE (Detect It Easy) | pestudio, Exeinfo PE |
| .NET decompile/debug | dnSpy (or dnSpyEx fork) | ILSpy, dotPeek |
| .NET deobfuscate | de4dot | UnconfuserEx, OldRod |
| Native disasm | Ghidra | IDA Pro, Binary Ninja, radare2 |
| Debug (native) | x64dbg | WinDbg, GDB |
| Anti-anti-debug | ScyllaHide | TitanHide |
| Unpack & fix IAT | Scylla (plugin) | x64dbg + manual dump |
| Java/Android | jadx-gui | CFR, apktool, Fernflower |
| Electron | asar extract | reko, electronegativity |
| PyInstaller | pyinstxtractor | (then) decompyle3 |
| Unity assets | AssetStudio, UnityPy | UABE |
| Unreal assets | FModel, UnrealPak | UModel |
| Godot | gdsdecomp | GDRE Tools |
| Monitoring | Process Monitor, Process Hacker | API Monitor |
| Network | Wireshark, mitmproxy | Fiddler, Burp |
| Hex + templates | 010 Editor | ImHex, HxD |
| Binary diffing (versions) | Diaphora (IDA) / Ghidra BSim | DarunGrim, zynamics BinDiff |
| Binary strings | Sysinternals strings | FLOSS (deobfuscates stack strings) |
| Symbolic execution | angr | Miasm |
| Runtime instrumentation | Frida | DynamoRIO, Qiling |
| Format spec | Kaitai Struct | construct (Python), 010 templates |

---

## 10. Legal & Ethical Quick Reference

- **Generally defensible**: interoperability, security research, malware analysis, analyzing software you own/contracted, personal backups/format migration, education.
- **Generally not defensible**: cracking licensing for use/distribution, circumventing DRM (illegal per se in many jurisdictions regardless of intent), lifting copyrighted code/assets into a distributed clone.
- **Clean-room discipline**: if the goal is a legally safe clone, the person writing the rebuild should ideally work from the written spec (§5), not from decompiled code — and the notes should record which path was taken.
- When in doubt, prefer: behavior-cloning over code-copying, original-look assets over extracted assets, and asking the user to confirm their authorization for the specific target.

---

## 11. Session Checklist

Run through this at the start of every RE session on a target:

- [ ] Working directory layout created (`original/ analysis/ extracted/ rebuild/ assets/ tests/ docs/`)
- [ ] Original copied to `original/` and hashed
- [ ] Triage notes started (`analysis/triage.md`) — container type, runtime, packer, protections
- [ ] Sandbox/VM ready if dynamic analysis is planned
- [ ] Prior session's `REVERSE-ENGINEERING-NOTES.md` read and open questions reviewed
- [ ] Scope confirmed with the user: which behavior matters, target stack for the rebuild, fidelity bar (byte-exact vs. behavioral), and legal standing

And at the end of every session:

- [ ] Notes updated with findings + evidence + confidence levels
- [ ] New scripts saved to `analysis/scripts/`
- [ ] New oracle traces (inputs/outputs) recorded into `tests/`
- [ ] TODO-RE unverified stubs listed in the Open Questions section

---

## 12. Advanced Techniques (Uncensored)

### License Key Cracking
When the goal is to understand or bypass a license check:

1. **Find the validation routine**: search for strings like "Invalid license", "Activate", "Register", "Trial expired". Set breakpoints on these strings or on functions that write to them.
2. **Trace backwards**: from the string, follow the call chain to find the actual validation logic (often a hash comparison, RSA signature check, or online activation).
3. **Patch the check**:
   - Simple comparison: `cmp eax, ebx` → `xor eax, eax; cmp eax, ebx` or just `nop` the conditional jump
   - Hash validation: dump the hash algorithm, recompute offline, or NOP the comparison
   - Online activation: patch the HTTP request or mock the response with mitmproxy
4. **Extract the key generator**: if it's a serial generator (keygen), the math is often simple (checksum, XOR, modular arithmetic). Symbolic execution with angr can solve for valid inputs automatically.

### Patching Techniques
- **x64dbg patching**: right-click instruction → "Patch Byte" → select operation (NOP, JMP, MOV, etc.)
- **Permanent patch**: dump memory after patching, or use a patching tool like **PE Patch** or **CFF Explorer** to write changes back to disk
- **IAT patching**: redirect DLL function calls to your own stubs (useful for hooking without full runtime instrumentation)
- **Section editing**: for larger patches, add a new section (`.patch`) and redirect execution there

### Memory Dumping (Post-Unpack)
When a packer/unpacker runs at runtime:
1. Run the binary in x64dbg
2. Let it fully unpack (watch for `GetModuleHandle`, `LoadLibrary` calls)
3. Break on `ExitProcess` or when the main window appears
4. Use Scylla plugin: "Dump" → select base address and size
5. Rebuild IAT (Import Address Table) with Scylla's "Auto IAT" feature
6. Fix TLS (Thread Local Storage) callbacks if present

### String Deobfuscation
Common patterns:
- **XOR with single byte**: `for (i=0; i<len; i++) str[i] ^= 0x42` → brute force all 256 keys
- **XOR with key stream**: find the key initialization pattern, extract the key
- **Virtual machine strings**: .NET obfuscators often store strings in a VM format; de4dot handles most
- **Runtime decryption**: set breakpoint on the decryption function, dump the plaintext string from memory

### Crypto Extraction
1. **Identify the algorithm**: look for S-boxes, constant tables, or known initialization vectors
2. **Extract keys from memory**: set breakpoints on crypto functions, watch for key material being loaded
3. **Hook the crypto calls**: use Frida or x64dbg to capture plaintext/ciphertext pairs
4. **Reconstruct offline**: implement the algorithm in Python with the extracted key

### Anti-Debugging Bypasses (Deep Dive)

#### IsDebuggerPresent
```assembly
; Original: call IsDebuggerPresent
; Patch: mov al, 0 ; ret
; Or: nop nop nop nop nop
```

#### NtQueryInformationProcess
Patch the call to return `FALSE` for `ProcessDebugPort` queries.

#### PEB BeingDebugged
```assembly
; x64dbg plugin (ScyllaHide) handles this automatically
; Manual: mov byte ptr [fs:0x60], 0
```

#### Timing Checks
```assembly
; rdtsc checks: NOP the rdtsc instruction or patch the comparison
; QueryPerformanceCounter: hook the function to return constant values
```

#### Stack Tracing
Some packers trace the stack for debugger artifacts. Patch the trace loop or use a kernel-mode debugger (WinDbg kernel mode).

---

## 13. Common Obfuscators & How to Beat Them

### .NET Obfuscators
| Obfuscator | Strategy |
|---|---|
| **ConfuserEx** | de4dot (partial), ConfuserEx-Unpacker, or manual VM decoding |
| **.NET Reactor** | de4dot (older versions), .NET Reactor Unpacker, or dnSpy runtime decryption |
| **Eazfuscator** | de4dot, or dump with x64dbg after full load |
| **Babel** | de4dot handles most cases |
| **SmartAssembly** | de4dot, or manual string decryption |

### Native Obfuscators
| Obfuscator | Strategy |
|---|---|
| **VMProtect** | Heavy lifting required; use VMProtect unpacker or trace-based analysis |
| **Themida** | ScyllaHide + manual unpacking; newer versions need custom patches |
| **Olympus** | Similar to VMProtect; trace execution and lift to pseudocode |
| **Armadillo** | Older versions: dump + IAT rebuild; newer: heavy anti-tamper |

### Control Flow Flattening
- **de4dot** handles most .NET cases
- **Ghidra** with control flow analysis scripts
- Manual: identify the dispatch table, reconstruct basic blocks

---

## 14. Game-Specific RE (Unity, Unreal, Godot)

### Unity
1. **Extract assemblies**: `Assembly-CSharp.dll`, `Assembly-CSharp-firstpass.dll` → dnSpy
2. **Extract assets**: AssetStudio for textures, models, audio, scenes
3. **Decrypt IL2CPP**: use `il2cppdumper` + `dnlib` to decompile IL2CPP games
4. **Game logic**: search for strings like "Player", "Spawn", "Damage", "Score"
5. **Network**: hook `UnityWebRequest` or `NetworkManager` with Frida

### Unreal Engine
1. **Extract .pak files**: UnrealPak command-line tool or FModel
2. **Parse .uasset/.umap**: FModel, UnrealExplorer, or UE4SS
3. **Decompile C++**: Ghidra with Unreal signatures, or use UE4's built-in reflection data
4. **Blueprints**: extract with FModel, reverse logic from execution flow
5. **Gameplay**: search for "AActor", "AGameMode", "APlayerController"

### Godot
1. **Extract .pck**: `gdsdecomp` or Godot's built-in PCK extractor
2. **Decompile GDScript**: usually human-readable; search for logic patterns
3. **C# projects**: extract and decompile with dnSpy
4. **Export templates**: use Godot's export to get the full project structure

---

## 15. Mobile RE (Android/iOS)

### Android
1. **Decompile APK**: `jadx-gui` for Java/Kotlin, `apktool` for resources
2. **Native libraries**: extract `.so` files, analyze with Ghidra
3. **Obfuscated Java**: look for ProGuard/R8 mappings, use `frida` to deobfuscate at runtime
4. **Native hooks**: `frida` for ARM/ARM64, hook JNI functions
5. **Root detection**: patch `Build.FINGERPRINT` checks, hide Magisk

### iOS
1. **Decompile IPA**: `class-dump` for headers, `Hopper`/`Ghidra` for binaries
2. **Objective-C runtime**: hook methods with `MSHook` or `Frida`
3. **Swift**: harder to decompile; focus on runtime behavior
4. **Jailbreak detection**: patch detection routines, use Cydia Substrate

---

## 16. Pro Tips (From Experience)

1. **Strings are your best friend**. 80% of the time, a simple string search will lead you to the logic you need.
2. **Don't read assembly linearly**. Jump to interesting strings, set breakpoints, and trace backwards.
3. **Save your Ghidra/IDA projects**. You'll need them for the next session.
4. **Test hypotheses quickly**. Write a small script to verify your understanding before committing to a full rebuild.
5. **Document as you go**. Your future self will thank you.
6. **When stuck, run it**. Dynamic analysis often reveals what static analysis hides.
7. **Don't fight the obfuscator**. If it's too hard, dump memory after it runs and analyze the result.
8. **Use multiple tools**. Ghidra might miss what IDA catches, and vice versa.
9. **Keep a "known good" state**. Always have a clean copy of the original to compare against.
10. **Know when to stop**. Some protections (Denuvo, heavy VMs) require weeks of work. Decide if it's worth it.

---

## 17. Quick Command Reference

```powershell
# Hash files
Get-FileHash -Path file.exe -Algorithm SHA256

# Extract strings
strings -n 8 file.exe > analysis\strings.txt

# Check PE headers
dumpbin /headers file.exe

# Extract resources
resourcehacker.exe -open file.exe -save extracted\

# Process Monitor filter
# Filter: Process Name is target.exe

# x64dbg commands
bp <address>        ; Set breakpoint
bp <module>!<func>  ; Set breakpoint on function
dump <address>      ; Dump memory
patch <address> <bytes>  ; Patch bytes
```

```bash
# Linux equivalents
file target
readelf -a target
strings -n 8 target > strings.txt
objdump -d target > disasm.txt
```

---

## 18. When to Call This Skill

Invoke this skill when:
- Reverse engineering an executable or application
- Creating a clone/replica of existing software
- Analyzing proprietary file formats
- Understanding how a program works internally
- Porting software to a different platform/language
- Cracking license checks or DRM
- Extracting assets from games or applications
- Reverse engineering network protocols
- Analyzing malware or suspicious binaries
- Building compatibility layers or emulators

---

**Remember**: The goal is understanding, not just copying. Every line of code you extract should teach you something about the original design. Use that knowledge to build something better, cleaner, or more maintainable.

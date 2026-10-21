---
name: reverse-engineering
description: >
  Professional, methodical reverse engineering of software for legitimate
  purposes — CTF challenges, malware analysis, security research and bug
  bounty, interoperability, analyzing your own or authorized applications,
  and learning. Use when asked to reverse engineer, decompile, disassemble,
  unpack, or "understand how an app/binary works" for any ELF/PE/Mach-O
  binary, APK, JAR, .NET assembly, Electron/JS bundle, Python .pyc/PyInstaller
  build, firmware image, or unknown file format. Covers triage, static and
  dynamic analysis, and writing up findings.
---

# Reverse Engineering

A disciplined workflow for taking apart software and explaining what it does.
Favor understanding and documentation over defeating protections.

## Scope and ethics — check first

Only proceed for a legitimate purpose. Confirm the request fits one of:
authorized testing of software the user owns or has permission to analyze,
CTF, malware analysis, security research / bug bounty, interoperability
(building a compatible client or parser), or education.

Decline, and say why, when the goal is to crack or bypass licensing/DRM,
produce a keygen/crack/patch to pirate paid software, strip copy protection,
or evade detection for malicious deployment. If the goal is unclear, ask
before producing anything that would primarily serve those ends. Note that a
EULA or local law may restrict RE regardless; flag that when relevant.

Malware is analyzed, never executed outside isolation. This environment has
no sandbox hardening — do **static** analysis on malware. Only run a sample
dynamically inside a disposable container with no network, and say so.

## Workflow

Work in a dedicated directory under the scratchpad, never from inside the
sample's own folder (a planted library can hijack an interpreter or build
tool). Pass sample paths as arguments. Keep your scripts separate from the
sample.

### 1. Triage — identify before you dig

```bash
file <sample>                      # format, arch, bits, interpreter
sha256sum <sample>                 # record a hash to refer to
ls -la <sample>; stat <sample>     # size, timestamps
xxd <sample> | head -40            # magic bytes, header
strings -n 6 <sample> | less       # URLs, paths, keys, messages, lib names
strings -el <sample> | less        # wide/UTF-16 strings (common in PE)
```

Let the format pick the branch below. When `file` is vague, check magic bytes
against the header, and run `binwalk <sample>` to spot embedded/packed data.

Watch for **packing**: very high entropy, almost no readable strings, a tiny
import table, or a UPX marker. `upx -t` / `upx -d` for UPX; note other packers
and unpack before deeper analysis.

### 2. Route by type

**Native ELF / PE / Mach-O**
```bash
readelf -h -l -d <bin>             # ELF header, segments, dynamic section
readelf -s <bin> | less            # symbols (if not stripped)
nm -C <bin>                        # demangled symbols
objdump -d -M intel <bin> | less   # disassembly (Intel syntax)
objdump -T <bin>                   # dynamic symbols / imports
rabin2 -I <bin>; rabin2 -z <bin>   # info + strings via radare2
```
Interactive with radare2: `r2 -A <bin>` then `aaa` (analyze), `afl` (list
functions), `s main; pdf` (disasm a function), `iz` (strings), `ii`
(imports), `axt <addr>` (xrefs). For scripted disassembly of a blob, use
the `capstone` Python module.

**Android APK**
```bash
unzip -l app.apk                   # inspect contents first
jadx -d out_src app.apk            # decompile dex -> readable Java/Kotlin
apktool d app.apk -o out_res       # smali + decoded resources + manifest
```
Read `out_res/AndroidManifest.xml` (permissions, exported components,
`networkSecurityConfig`), then `out_src` for logic. Grep for endpoints,
secrets, crypto: `grep -rniE "https?://|secret|api[_-]?key|password|AES|RSA" out_src`.

**Java / .NET**
- JAR: `jadx -d out jar_file.jar` (or unzip and decompile `.class`).
- .NET: read strings/metadata with `strings`/`rabin2`; for full decompilation
  note that ILSpy/dnSpy are GUI/Windows — on this headless box, extract what
  you can and say a .NET decompiler is the right next tool.

**Electron / JavaScript**
```bash
npx --yes @electron/asar extract app.asar out_asar   # unpack asar
```
Then beautify minified JS with `npx --yes js-beautify file.js`, grep for IPC
channels, `require` of native modules, URLs and tokens.

**Python**
- `.pyc`: `pip install decompyle3` / `uncompyle6` (version-dependent), or read
  bytecode with `python3 -m dis`.
- PyInstaller EXE: extract with `pyinstxtractor`, then decompile the `.pyc`s.

**Firmware / unknown blobs**
```bash
binwalk <image>                    # map embedded filesystems/compression
binwalk -e <image>                 # auto-extract (review before trusting)
```

### 3. Dynamic analysis (only on trusted / authorized, non-malware binaries)

```bash
strace -f -e trace=file,network,process -o trace.log ./<bin>   # syscalls
gdb ./<bin>                        # break, inspect, step
```
In gdb: `break main`, `run`, `info registers`, `x/20xw $sp`, `disas`,
`layout asm`. For a function you reversed, confirm behavior by breaking on it
and reading arguments. Prefer this to guessing from static code alone when
logic is obfuscated.

### 4. Report

Produce a written summary, not just raw dumps:
- **Identity**: name, hash, format/arch, size, whether packed.
- **Capabilities**: what it does — files touched, network endpoints, crypto,
  persistence, notable strings.
- **Mechanism**: the specific logic the user asked about, traced to functions
  / addresses / source lines so it's verifiable.
- **Findings**: vulnerabilities, hardcoded secrets, or protocol details,
  with evidence.
- **Confidence & gaps**: what's confirmed vs. inferred, and what tool would
  take it further.

Put findings the user will keep in a file in the working directory or
scratchpad; cite `file:line` or addresses so claims can be checked.

## Tools available in this environment

Installed: `file`, `strings`, `xxd`, `od`, `objdump`, `readelf`, `nm`, `gdb`,
`strace`, `radare2`/`rabin2`/`rax2`, `binwalk`, `upx`, `jadx`, `apktool`,
`unzip`, Python `capstone` + `pyelftools`, Java 21, Node/npm, Docker.
No GUI — everything is CLI/headless/scriptable. No GPU. Install anything else
(e.g. Ghidra headless, uncompyle6, pyinstxtractor) on demand.

Missing but sometimes wanted: `ltrace` (use `strace` + gdb), ILSpy/dnSpy
(GUI/Windows), Ghidra GUI (use `analyzeHeadless` if Ghidra is installed).

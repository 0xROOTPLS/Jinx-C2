<h1 align="center">Jinx</h1>

<p align="center">
  <strong>A post-exploitation command and control framework built from the ground up in Rust.</strong>
</p>

<p align="center">
  <a href="https://tbd.com"><img src="https://img.shields.io/badge/docs-TBD-1a1a2e?style=flat-square&logo=gitbook&logoColor=white" alt="Documentation"/></a>
  <a href="https://icalia.com"><img src="https://img.shields.io/badge/web-icalia.com-0a0a0a?style=flat-square&logo=firefox&logoColor=white" alt="icalia.com"/></a>
  <a href="mailto:fueledbycaffeine@pm.me"><img src="https://img.shields.io/badge/email-fueledbycaffeine-6D4AFF?style=flat-square&logo=protonmail&logoColor=white" alt="protonmail"/></a>
  <a href="https://discord.com"><img src="https://img.shields.io/badge/discord-americium241-5865F2?style=flat-square&logo=discord&logoColor=white" alt="discord"/></a>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/status-pre--release-orange?style=flat-square" alt="Pre-Release"/>
  <img src="https://img.shields.io/badge/agent-Rust-dea584?style=flat-square&logo=rust&logoColor=white" alt="Rust Agent"/>
  <img src="https://img.shields.io/badge/teamserver-Rust-dea584?style=flat-square&logo=rust&logoColor=white" alt="Rust Teamserver"/>
  <img src="https://img.shields.io/badge/platform-Windows%20x64-0078D6?style=flat-square&logo=windows&logoColor=white" alt="Windows x64"/>
</p>

---

> **Jinx is not publicly released.** The framework is currently undergoing long-term stability testing. I am targeting one month of continuous agent uptime in a live environment before I consider it suitable for release. Estimated availability is **late Q2 2026 to early Q3 2026**. This repository exists to stage the project and give the public a preview of what is coming. Everything you see here is functional and has been extensively tested.

---

## What is Jinx?

Jinx is a modern post-exploitation framework I have been building solo over a long period. The agent, teamserver, and operator panel were all written from the ground up with no forks, no wrappers around existing tools (except in-house), and no shortcuts. I intend Jinx to be the most complete product I have ever released.

The agent is written in Rust. The teamserver is async Rust. The operator panel is xplatform PyQt5.

I built this because the C2 frameworks I use all had tradeoffs I got tired of accepting. Jinx is my answer to those tradeoffs.

## Agent Capabilities

**Evasion stack**:
- Zero import table. The dynamic resolver itself is allocation-free
- Indirect syscalls for all Nt* functions (Halo's Gate fallback if hooked)
- Return address spoofing on every Win32 API call (Cuckoo)
- Full string obfuscation (Obfstr).
- Sleep encryption across .text, .rdata, .data, and heap sections using a unique design with zero agent stack frames during sleep. The heap allocator is also frozen for the entire encrypt/sleep/decrypt window
- Protected body encryption: the sleep encryption keys themselves are encrypted under a separately rotating key during the sleep window.
- Polymorphic: config, section names, entry point, HTTP paths, headers, and User-Agent all randomized per build. Two builds from the same source are signature-distinct

**Self-defense**:

The anti-tamper system is not a "check and branch". The syscall gadget the agent needs to function is derived live from XOR of the .text hash, a 256-byte canary, and a 4096-byte scattered canary page. If any of those regions are modified, the gadget becomes garbage and the next Nt* call crashes the agent.. The integrity check IS the execution path.

- .text integrity hashing, inlined (`#[inline(always)]`) at every syscall call site. Each expansion produces unique machine code. Patching one leaves the others firing
- 16 scattered heap canary regions across a 4 KiB page, pseudo-random subset verified per cycle, hashed directly into the syscall gadget
- Hardware breakpoint detection on every thread in the process. Dr0-Dr3 must be zero on all of them
- Three-tier failure response: concrete tampering triggers a death ping to the teamserver (which applies per-build policy: IP ban, subnet ban, or build burn), then full memory wipe and exit. Ambiguous signals trigger silent wipe with no network artifact. Unhandled exceptions (AV, INT3, illegal opcode) hit the backstop filter for best-effort wipe, no WER dispatch, no minidump, no event log entry
- Full memory wipe on any exit path: heap, .text, .rdata, PE header, and every region in the wipe registry. The wipe stub self-cannibalizes last so the wipe code itself does not outlive the wipe

**Operations:**

| Category | Commands |
|---|---|
| Recon | `whoami` `hostname` `pwd` `cd` `dir` `ls` `cat` `ps` `ipconfig` `netstat` `env` `getprivs` (all native API, no cmd.exe) |
| File ops | `upload` `download` `mkdir` `rm` `cp` `mv` |
| Execution | `shell` `execute-assembly` `shellexec` `bof` (CS-compatible COFF loader with GOT resolution, relocation handling, VEH crash isolation) |
| Process/Token | `ps` `kill` `screenshot` `steal-token` `make-token` `rev2self` |
| Lateral movement | `psexec` `scshell` `wmiexec` `winrm` |
| Injection | Waiting Thread Hijacking |
| Pivoting | Named-pipe relay, reverse port forwarding |

**Opsec controls:**
- Kill date: hard epoch deadline checked before first network operation
- Execution guards: hostname, username, and domain allowlist/blocklist
- Single-instance mutex enforcement
- Persistent agent ID derived from MachineGUID + build ID. (No dup agents)
- Tiered exponential backoff
- TCP fallback if HTTP fails mid-operation
- Domain fronting with configurable CDN host header override

**Build outputs:** EXE (Dont use this, make your own loader), DLL (reflective loading), BIN (shellcode via [Fritter](https://github.com/0xROOTPLS/Fritter)), Windows Service (w/ fallback).

## Teamserver

**Authentication:**
- Three-factor operator auth: mTLS client certificates (silent drop if invalid), API key (constant-time compare), and TOTP
- Session tokens are RAM-only, 8-hour expiry, wiped on panel close or restart. Re-auth invalidates that operator's previous token
- First-run provisioning generates CA, certs, API key, and TOTP secret automatically. TOTP displayed once

**Infrastructure defense (this is where Jinx differentiates itself):**

The teamserver is designed to be hostile to anyone who is not a registered agent or authenticated operator. Every validation failure results in an immediate, permanent IP ban and a response indistinguishable from the cover identity. There are no error pages, no status code differences, no timing differences that reveal whether a request was "close" to valid. Timing correlation attacks also covered.

- Cover identity on every response. Configurable web server persona (default nginx/1.24.0). Does not change on response type.
- IP blacklisting on any failure across any surface. Persisted across restarts. Operator IPs are protected from bans
- Subnet bans: /24 bans supported. When an agent's self-defense detects concrete tampering (analyst breakpoints, memory patching), the death ping reaches the teamserver and triggers per-build policy. That policy can ban the source IP, ban the entire /24 subnet, burn the build (unregister all its routes and ban), or burn and subnet ban.
- Anti-replay: session ID nonce cache with timestamp validation, single write lock. Captured traffic cannot be replayed
- Agent anti-hijack: re-checkins update only last_seen, IP, and PID. Identity fields are immutable after first registration
- Rate limiting with per-IP sliding window. Exceeding the limit is a ban, not a throttle
- Resource caps across results, nonce cache, and IP tracker to prevent state exhaustion

**Polymorphic infrastructure:**
- Each build registers its own derived routes on the teamserver. 4 checkin + 4 beacon + 4 result path templates produce roughly 32,000 endpoint combinations per build with variable 2-3 segment names.
- TLS certificates use a multi-billion combination procedural phonetic name generator seeded from the shared secret. Builds inherit this generator for API route names and build names. Overridable with `--cert-cn` if you want a specific identity
- No fixed paths anywhere. Everything outside registered build routes returns the cover page
- Builds can be tagged, tracked, burned (unregisters routes and kills all active agents on that build), cleared, or removed

**Other:**
- Agent builder compiles and returns polymorphic payloads directly from the panel (EXE, DLL, BIN, Service)
- Full state persistence: agents, listeners, builds, and blacklists all survive restarts
- Multi-operator support with independent TOTP auth per operator

## Hex (Operator Scripting)

If you have used Cobalt Strike's Aggressor scripting, Hex is my take on the same idea, built for Jinx and designed to be safer by default.

Hex is a server-side Lua 5.4 scripting layer embedded in the teamserver. Each hex runs in its own sandboxed VM (one VM per hex for crash isolation and clean teardown) with a capability-gated API. Ten capabilities are available, all deny-by-default:

| Capability | What it does |
|---|---|
| `agents.read` | Snapshot of every active agent and its identity fields |
| `agents.tag` | Set the tag field on an agent (persisted) |
| `agents.task` | Enqueue any of 47 task types on a target |
| `events.subscribe` | Hook session-connected, task-result, build-burned, listener events |
| `events.emit` | Push a custom audit-log entry |
| `listeners.read` | Enumerate active listeners |
| `cron` | Schedule a function on interval or daily at a specific time |
| `storage` | Per-hex key/value store (1 MiB, 1024 keys, JSON-backed) |
| `bofs.read` | List the BOF arsenal (name, SHA-256, size) |
| `bofs.run` | Run an arsenal BOF on an agent with full audit provenance |

An ungranted capability is simply not bound as a global in that VM. The script cannot reach it. Manifest-time validation rejects unknown capability names so typos fail loudly instead of silently granting nothing.

Hexes can be invoked against a single agent, multi-selected agents, or with no agent target at all (the script picks its own targets in `on_invoke(nil)`). This makes group operations and automated response workflows straightforward.

Scripts are inert until explicitly enabled by an operator. Resource limits are hardcoded (4 MiB memory per VM, 1 second wall-clock per call, capped subscribers and cron jobs per hex). Everything is append-only audit logged with source SHA-256 pinning. BOF executions get two-layer provenance: which hex ran which binary.

## Spellbook (BOF Arsenal)

The Spellbook is a server-side store of Beacon Object Files maintained on the teamserver at `<workspace>/bofs/`. CS-compatible COFF files up to 100 MB. The BOF loader in the agent handles `__imp_` GOT resolution, full relocation processing, and VEH-based crash isolation so a bad BOF does not take down the agent.

Hexes resolve BOFs by name via `bofs.run`, pack arguments, queue the task, and audit the execution with the BOF's SHA-256. The arsenal is browsable and manageable from the operator panel (Tools > BOF Manager) or from any agent terminal via `boflist`.

## Operator Panel

The panel is a full PyQt5 cross-platform desktop application. Everything is integrated and manageable from one place:

- Agent interaction with full command terminal per agent
- Payload generation with build configuration (Payloads > Generate Sigil)
- Listener management (HTTP, HTTPS, TCP) with start/stop/remove
- Hex Manager (Tools > Hex Manager): upload, view source, edit in a built-in monospace editor, save and reload, start/stop, invoke, and remove hexes without ever touching the filesystem. The full audit log (last 500 lines) is visible below the hex table
- BOF Manager (Tools > BOF Manager): upload, remove, and browse the server-side arsenal
- Right-click agent context menus for hex invocation (single agent or fan-out across a selection)
- Multi-operator support over mTLS with independent TOTP authentication per operator

## Responsible Use

Jinx is built for authorized security testing, red team engagements, and security research. I am releasing it because I believe offensive tooling should be available to defenders and researchers, not locked behind $$$ contracts.

**Do not use this framework without explicit written authorization from the system owner.** Unauthorized access to computer systems is illegal in every jurisdiction I am aware of. I am not responsible for misuse.

## Project Status

This is a solo project. I work on it because I care about it, not because there is a team or a company behind it. Once released, updates will come at a slow but sustainable pace. I would rather ship something reliable than rush features. If you are looking for a framework backed by a large team with weekly releases, this is not that. If you want something built carefully by someone who actually uses it, and not rushed by money, stay tuned.

Full documentation, usage guides, and tutorials will be available at [TBD](https://tbd.lol) at release.

---

<p align="center">
  <sub>Built by one person who got tired of accepting OPSEC as a secondary thought.</sub>
</p>

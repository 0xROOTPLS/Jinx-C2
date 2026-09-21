<h1 align="center">Jinx</h1>

<p align="center">
  <strong>A post-exploitation command and control framework built from the dust, in Rust.</strong>
</p>

<p align="center">
  <a href="https://tbd.com"><img src="https://img.shields.io/badge/docs-TBD-1a1a2e?style=flat-square&logo=gitbook&logoColor=white" alt="Documentation"/></a>
  <a href="https://icalia.com"><img src="https://img.shields.io/badge/web-icalia.com-0a0a0a?style=flat-square&logo=firefox&logoColor=white" alt="icalia.com"/></a>
  <a href="mailto:ihateav@pm.me"><img src="https://img.shields.io/badge/email-ihateav-6D4AFF?style=flat-square&logo=protonmail&logoColor=white" alt="protonmail"/></a>
</p>


---

> **Jinx is not publicly released.** The framework is currently undergoing long-term stability testing and further refinement. Estimated availability is ~**early Q3 2026**~. This repository exists to stage the project and give the public a preview of what is coming. Everything you see here is functional and has been extensively tested.

* *The release of Jinx has been delayed until **mid Q4 2026** due to a string of life events including the developer getting married mid October. Thank you for your patience*
---

## What is Jinx?

Jinx is a modern post-exploitation framework that I have been building **solo** over a *long period*.

The agent (rust), teamserver (rust), and operator panel (x-platform PyQt5) were all written from scratch.

During development zero shortcuts were ever taken, and each module was thoroughly audited for stability, safety, and security. I intend Jinx to be the most complete product I have ever released.

I built this because the several frameworks I have used all had tradeoffs which bugged me. Jinx is my answer to this.

## Agent Capabilities

**Evasion stack**:
- Zero import table. The dynamic resolver itself is allocation-free as well.
- Indirect syscalls (Halo's Gate fallback)
- Return address spoofing on every Win32 API call [Cuckoo Method](https://github.com/0xROOTPLS/Cuckoo-Spoofer)
- Full string obfuscation (Obfstr).
- Sleep encryption across .text, .rdata, .data, and heap sections using a unique design with zero agent stack frames during sleep. (The heap allocator is also frozen for the entire encrypt/sleep/decrypt window)
- Protected body encryption: the sleep encryption keys themselves are encrypted under a separately rotating key during the sleep window.
- Polymorphic: config, section names, entry point, HTTP paths, headers, and user-agent randomized per build.

**Self-defense**:

Jinx's anti-tamper system is not a "check and branch" design. The syscall gadget the agent needs to function is derived live from a XOR of the .text hash, a 256-byte canary, and a 4096-byte scattered canary page. If any of those regions are modified, the gadget becomes garbage and the next Nt* call crashes the agent.. The integrity check IS the execution path.

- .text integrity hashing, inlined (`#[inline(always)]`) at every syscall call site. Each expansion produces unique machine code.
- 16 scattered heap canary regions across a 4 KiB page, pseudo-random subset verified per cycle, hashed into the syscall gadget
- Hardware breakpoint detection on every thread. (Dr0-Dr3 must be zero on all of them)
- Three-tier failure response: concrete tampering triggers a death ping to the teamserver (which applies per-build policy: IP ban, subnet ban, or build burn), then full memory wipe and exit. Ambiguous signals trigger silent wipe with no network. Unhandled exceptions (AV, INT3, illegal opcode) hit the backstop filter for best-effort wipe (no WER dispatch, minidump, or eventlog)
- Full memory wipe on any exit path: heap, .text, .rdata, PE header, and every region in the wipe registry.

**Operations:**

| Category | Commands |
|---|---|
| Recon (cmd.exe-less) | `whoami` `hostname` `ipconfig` `netstat` `ps` `env` `pwd` `ls` `dir` `cd` `cat` `get_privs` `ping` `port_scan` `reg_read` `reg_enum` |
| File Operations | `mkdir` `rm` `cp` `mv` `download` `upload` `timestomp` `reg_write` `reg_delete` |
| Execution | `shell` `shell_exec` `bof` `execute_assembly` `screenshot` `rev_shell` |
| Tokens / Impersonation / Privesc | `steal_token` `make_token` `rev2self` `uac_elevate` |
| Lateral Movement | `lateral_psexec` `lateral_scshell` `lateral_failure_actions` `lateral_wmi` `lateral_wmi_event` `lateral_winrm` `lateral_dcom` |
| Pivot / Networking | `pivot_open` `rportfwd` `rportfwd_close` |
| Service Control | `svc_enum` `svc_create` `svc_delete` `svc_start` `svc_stop` |
| Lifecycle | `kill` `exit` |

All seven lateral vectors run in-process. This means no classical PsExec drop, PowerShell launch, or temp files. Each authenticates as the caller (impersonation token if one is held). You can pair with `steal_token` / `make_token` for cross-principal movement. 

**Opsec controls:**
- Kill date: hard epoch deadline checked before first network op
- Execution guards: hostname, username, and domain allowlist/denylist
- Single-instance mutex enforcement
- Persistent agent ID derived from MachineGUID + build ID. (No dup agents)
- Tiered exponential backoff
- TCP fallback if HTTP fails mid-op
- Domain fronting with configurable CDN host header override

**Build outputs:** EXE, DLL (reflective loading), BIN (shellcode via [Fritter](https://github.com/0xROOTPLS/Fritter)), Windows Service exe (w/ fallback).

## Teamserver

**Authentication:**
- Three-factor operator auth: mTLS client certificates (silent drop if invalid), API key (constant-time compare), and TOTP
- Session tokens are RAM-only with 8-hour expiry. Re-auth invalidates previous token.
- First-run provisioning generates CA, certs, API key, and TOTP secret automatically. TOTP is displayed once upon setup.

**Infrastructure defense:**

The teamserver is designed to be hostile to anyone who is not a registered agent or authenticated operator. Every validation failure results in an immediate, permanent IP ban and a response indistinguishable from the cover identity. 
**Validation failure does not mean authentication mistake, instead its any malformed network operation directed at the teamserver which it does not recognize.* 

There are no error pages, status code differences, or timing differences that reveal if a request was "close" to valid. Timing correlation attacks also explicitly negated.

- Cover identity on every response that does not change on response type. Configurable web server persona (default nginx/1.24.0)
- IP blacklisting on any failure across any surface. Persisted across restarts. *Operator IPs are protected from bans*
- Subnet bans: /24 bans supported. When an agent's self-defense detects concrete tampering (breakpoints, patching), the death ping reaches the teamserver and triggers per-build policy. That policy can ban the source IP, ban the entire /24 subnet, burn the build (unregister all its http routes), or burn + subnet ban.
- Anti-replay: session ID nonce cache with timestamp validation + single write lock.
- Agent anti-hijack: re-checkins update only last seen, IP, and PID. Agent identity fields are immutable after first registration
- Rate limiting with per-IP sliding window. Exceeding the limit is a ban instead of a throttle (*Operators will never hit these limits*)
- Resource caps across results, cache, and IP tracker to prevent state exhaustion

**Polymorphic infrastructure:**
- Each build registers its own derived routes on the teamserver. 4 checkin + 4 beacon + 4 result path templates that produce roughly 32,000 endpoint combinations per build with variable 2-3 segment names.
- TLS certificates use a multi-billion combination procedural phonetic name generator seeded from the teamserver's shared secret. Builds inherit this generator for API route names and build names. Overridable if desired.
- No fixed paths **anywhere**.

**Other:**
- Agent builder compiles and returns polymorphic payloads directly from the panel (EXE, DLL, BIN, Service)
- Full cross-restart state persistence: operators, agents, listeners, builds, and blacklists persist.
- Multi-operator support with independent TOTP auth per operator
- Multi-operator collaborative chat channel included

## Hex (Operator Scripting)

If you have used Cobalt Strike's Aggressor scripting, Hex is my take on the idea, but built for Jinx and designed to be safer.

Hex is a server-side Lua 5.4 scripting layer embedded in the teamserver. Each hex runs in its own sandboxed VM with a capability-gated API. Twelve capabilities are available:

| Capability | Use |
|---|---|
| `agents.read` | Snapshot of every active agent and its identity fields |
| `agents.tag` | Set the tag field on an agent (persisted) |
| `agents.task` | Enqueue any of 49 task types on a target |
| `agents.results` | Read task output: `last_result(agent_id)` and `result(agent_id, task_id)` |
| `events.subscribe` | Hook lifecycle events (session-connected, task-result, build-burned, listener-started/stopped) |
| `events.emit` | Push a custom audit-log entry |
| `listeners.read` | Enumerate active listeners |
| `cron` | Schedule a function on interval or daily at a specific time |
| `storage` | Per-hex key/value store (1024 keys, ~1 MiB total, JSON-backed, survives restarts) |
| `bofs.read` | List the BOF arsenal (name, SHA-256, size) |
| `bofs.run` | Run an arsenal BOF on an agent with full audit provenance |
| `network.http` | SSRF-guarded HTTPS POST (webhooks, chat ops, external IR notifications) |

Hexes can be invoked against a single agent, multi-selected agents, or with no agent target at all (the script can pick its own targets in `on_invoke(nil)`). This should make group operations and automated response workflows straightforward if utilized properly.

The Lua standard library is trimmed to `table` / `string` / `math` / `utf8` | `os.*`, `io.*`, `load`/`loadstring`/`require` have been removed.  
Pure-compute helpers (`util.base64_*`, `util.json_encode`, `time.now_unix`, `time.now_iso`, `time.format_iso`, `time.since`) are bound to every hex regardless of capability grants so scripts can build webhook bodies and timestamp audit entries without I/O capability.

Scripts are inert until enabled by an operator. Resource limits **are enforced and hardcoded** (4 MiB memory per VM, 1 second clock *per call* enforced by a debug-hook deadline, with capped subscribers and cron jobs per hex).

Scripts are audit logged to aid in case of internal failures. (*BOF executions get two layer provenance*)

The repo ships with **14 example hexes** under `/hexes/` covering scheduled summaries, structured-key storage with periodic cleanup, lifecycle-event incident response, dormant per-agent invocation, BOF execution patterns, and chat-ops webhook delivery.

## Spellbook (BOF Arsenal)

The Spellbook is a server-side store of Beacon Object Files maintained on the teamserver at `<workspace>/bofs/`. x64 COFF files (standard CS format) up to 100 MB. Admin drop-ins under the workspace directory are picked up automatically at startup, or via the panel's "Scan Disk" button which reconciles the in-memory index with disk.

The BOF loader in the agent handles `__imp_` GOT resolution, full relocation processing across all eight AMD64 forms, and VEH-based crash isolation. The full Cobalt-Strike Beacon API surface is exposed: output/printf, the data/format families, token impersonation, admin/spawnto/cleanup/inject.

Hexes resolve BOFs by name via `bofs.run`, pack arguments using CS `<type>:<value>` tokens (`i`/`s`/`z`/`Z`/`b`), queue the task, and audit the execution with both the hex's SHA-256 and the BOF's SHA-256. The arsenal is browsable and manageable from the operator panel (Tools > BOF Manager) or from any agent terminal via `boflist`.


## Responsible Use

Jinx is built for authorized security testing, red team engagements, and security research. I am releasing it because I believe offensive tooling should be available to defenders and researchers, not locked behind $$$ contracts.

**Do not use this framework without explicit written authorization from the system owner.** Unauthorized access to computer systems is illegal in every jurisdiction I am aware of. I am not responsible for misuse.

## Project Status

This is a solo project. I work on it because I care about it, not because there is an investing company behind it. Once released, updates will come at a slow but sustainable pace. I would rather ship something reliable than rush features.

Full documentation, usage guides, and tutorials will be available at [TBD](https://tbd.lol) at release.

---

<p align="center">
  <sub>Built with <3 by 0xR00TPLS.</sub>
</p>

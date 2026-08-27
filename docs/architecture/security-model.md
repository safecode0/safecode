# 🔐 security model — capability-based, deny-by-default, non-negotiable

- **status:** accepted + fully implemented
- **date:** 2026-08-13

## the threat

ferrous executes **untrusted inputs** — downloaded code, web content, model output, repos, tool results — and it holds user credentials. if ANY subsystem gets breached, the whole machine is at risk. so isolation isn't a nice-to-have, it's the entire architecture.

## the decisions

1. **everything is sandboxed.** every subsystem runs in a capability-scoped WASI sandbox (embedded wasmtime). **zero ambient authority** — stdio, fs, and net are granted per-subsystem, explicitly, or not at all.
2. **layered isolation.** OS process-level isolation where feasible, in-process WASI capability scoping everywhere else.
3. **credentials.** encrypted at rest (argon2 + OS keyring). injected ONLY as env vars into granted sandboxes. never plaintext on disk, never in logs.
4. **no unsafe code.** `#![forbid(unsafe_code)]` in every crate, workspace-wide via `unsafe_code = "forbid"`.
5. **locked on launch.** master-password sign-in required. nothing runs unlocked, period.

## the three boundaries (this is the whole game)

### 1. the component boundary
only **validated WASI components** get admitted. no AOT artifacts, no serialized modules, nothing sketchy. `compile_component` runs the safe raw-component admission path and keeps the compiled handle in memory.

### 2. the capability boundary
fs, env, and network are **granted explicitly and enforced by the host**. the guest gets no handle outside its grants. a missing grant fails closed — no "well maybe just this once" fallback anywhere.

the grant model is typed and tested:
- filesystem roots with `Read` vs `ReadWrite` access levels
- environment **allowlists** (a name not granted is never even read from the host)
- loopback ports for *connect* vs *bind* — separate lists. binding is denied by default because a bound port can be observed + hijacked by other local processes. subtle and correct.
- native execution is a separate explicit grant. always approval-gated.

### 3. the resource boundary
every guest is bounded:
- memory (`StoreLimits`), fuel (instructions), output budget, and wall-clock timeout
- cancellation as the escape hatch — `CancelHandle` is one-shot, never resets
- **combined** output budget across stdout+stderr so a guest can't double its allowance by splitting streams

## symlink-aware path checking (this one's important)

lexical prefix checks alone are not enough — a symlink inside the workspace can point outside it. so `allows_existing_path` **canonicalizes the nearest existing ancestor** and re-checks the resolved path against the canonicalized grant root. a new file under an existing symlink gets caught. a missing root gets denied rather than treated as safe. there's a dedicated test proving a symlinked cwd that escapes the grant is rejected. 🎯

## consequences

- sandboxing overhead is benchmarked, never silently sacrificed.
- some LSP servers may need native-process hosting outside WASI — that's a documented, capability-limited fallback, not a free-for-all.
- production distribution gets a full re-audit of sandbox escapes + keyring path before shipping. this is the baseline, not the finish line.

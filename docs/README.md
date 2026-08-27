# 📚 docs — the whole brain dump, organised

this is where every single thought about safecode lives. not a scrap of "trust me bro it works" energy — everything here is **implemented, tested, and measured**. ngl the codebase is kinda stacked so the docs gotta keep up.

## what's in here

| folder | what it is | the vibe |
| --- | --- | --- |
| [`architecture/`](architecture/) | the decisions we locked in and why | "we said this so we dont regret it later" |
| [`engineering/`](engineering/) | the actual builds — runtime, shell, PTY, audits | "we built this and it goes hard" |
| [`risk-register.md`](risk-register.md) | every scary thing + how we killed it | "we thought about the bad ending so you dont have to" |

## read order (if ur new here)

1. [`architecture/security-model.md`](architecture/security-model.md) — **start here.** the whole project is one big capability sandbox and this is the blueprint.
2. [`architecture/dependency-policy.md`](architecture/dependency-policy.md) — why every dependency is permissive-licensed. short but load-bearing.
3. [`engineering/shell-harness.md`](engineering/shell-harness.md) — the shell itself: typed IR, safe parser, persistent sessions. this is the biggest single build.
4. [`engineering/native-pty.md`](engineering/native-pty.md) — the approval-gated native backend. real PTYs, real argv, zero shell-string nonsense.
5. [`engineering/security-audit.md`](engineering/security-audit.md) — the red-team pass + benchmarks. we literally tried to break our own sandbox.
6. [`risk-register.md`](risk-register.md) — the full threat ledger with test names for every mitigation.

## the TL;DR of everything

- **capability-based, deny-by-default** — nothing can touch fs/env/net unless its explicitly granted. missing grant = fail closed. no exceptions.
- **`#![forbid(unsafe_code)]`** everywhere. we dont even allow the unsafe keyword.
- **WASI Preview 2 + Wasmtime 47** for the sandboxed path, **portable-pty** for the approved native path.
- **318 tests passing** on Linux + macOS + Windows, clippy `-D warnings`, cargo-deny clean.
- **sub-nanosecond to sub-microsecond** policy primitives (measured, not vibes).
- **zero copyleft deps.** every license is permissive. enforced by cargo-deny in CI.

ur welcome to steal any of this. it's all MIT/Apache-2.0 anyway lol.

# 🐚 shell harness — the biggest single build, done

- **status:** implemented, 318 tests green, adversarially verified
- **stack:** Rust 2024, wasmtime 47 / WASI P2, portable-pty 0.9, argon2, secrecy, sha2

this is the core of the whole project: a persistent terminal harness where **humans get bash-like power and agents get structured, capability-enforced execution** — and the human password is untouchable by the model. here's what got built, piece by piece.

## the typed shell IR (shell_ir.rs)

human shell text AND AI tool calls compile into **one typed IR**. every program, arg, redirect, and cwd is a separate typed value. the parser never emits a shell string, so metacharacters can't be reinterpreted by a host shell — they're literally just characters in an argv slot.

key types:

```rust
ShellProgram { statements: Vec<Statement> }
Statement::{ Command(CommandSpec), Pipeline(Vec<CommandSpec>),
             And(..), Or(..), Sequence(..), Background(..) }
Program::{ Builtin(Builtin), External(String), WasiComponent(String), NativeShell(NativeShellKind) }
Builtin::{ Pwd, Cd, Ls, Cat, Mkdir, Remove, Copy, Move, Env, Which }
```

every plan hashes to a canonical `CommandDigest` (SHA-256 over a versioned byte encoding of the AST + cwd + argv + redirects + network scope + secret names — never secret VALUES). approvals and audit records bind to the exact plan that ran. change the plan → different digest → prior approval invalid.

## the safe parser (shell_parse.rs)

recursive-descent over a bounded tokenizer. supports quoting (`'`/`"`), backslash escapes, simple commands, `|`, `&&`, `||`, `;`, redirection (`<`, `>`, `>>`), background `&`.

**rejects with explicit errors:** command substitution, arithmetic expansion, here-docs, process substitution, aliases, functions, `eval`, env-var expansion, NUL bytes — anything that'd need unbounded host-shell evaluation.

**bounded, always:** 64KB max input, 4096 max tokens, 64 max pipeline stages, 4096 max statements. untrusted input physically cannot exhaust memory. the parser never touches the filesystem, never executes, never emits a shell command string.

there's also `parse_ai_argv` — the AI hands over a program + separate args, skipping text parsing entirely. structured in, structured out.

## persistent sessions (terminal_session.rs)

`TerminalSession` owns:
- capability-relative cwd (`cd` mutates session state ONLY — the host process cwd is never touched, tested)
- environment overlay (only allowlisted names can enter — tested)
- bounded job table (max 16 live jobs, closed sessions reject new ones)
- an optional broker-minted approval lease
- symlink-aware path resolution (escape attempts → `PathDenied`)

background jobs get a **snapshot** of the session (deep copy: same spec, cwd, env, limits, but independent job table + no lease) so mutations never leak back. closing a session cancels every owned job and releases the lease.

## execution (shell_executor.rs + broker.rs)

- **builtins** run in-process — never via `/bin/sh` or `cmd.exe`
- **external programs** run by direct argv through the native policy adapter
- **WASI components** run through the embedded wasmtime runtime
- **pipelines** use bounded channels with backpressure (a blocked downstream can be cancelled without deadlocking the broker)
- **`&&`/`||`** short-circuit on exact exit status
- **redirections** target capability-checked files
- **background jobs** owned by the session, killed on close

the action broker serializes execution behind one queue, gates risky actions on approval (pending jobs parked with a deadline, sweeper auto-denies after 30s), records an audit trail (capped at 4096 entries), and contains worker panics so a guest-triggered host bug can't kill the queue.

## human-only elevation (elevation.rs)

the model can REQUEST elevation but can never approve itself. an `ElevationRequest` describes exact effects (reads/writes/deletes/network/scripts) + requested capability delta — **no password field, ever**. a trusted `HumanApprovalAuthority` verifies the human; the broker then mints a short-lived, digest-bound, capability-scoped, revocable `ApprovalLease`. the agent API only ever sees a redacted `PendingApproval`.

the broker rejects: session mismatch, digest mismatch, expired/revoked lease, broader scope than requested, secret access not explicitly approved, any password text sneaking into args/env/events/audit. each has a test.

## the trust boundary

the raw password exists ONLY inside the profiles-vault verification path (argon2). it never enters `CommandRequest`, the agent context, an event, a child environment, a log, or an audit record. this is enforced by the type system (`secrecy::SecretString`, no `Serialize` on secret types) AND by sentinel tests that plant a password-like value and prove it can't cross.

## CLI proof surface (ferrous shell)

`ferrous shell` opens one persistent session and renders sanitized events; `--json` emits structured records for AI harness tests and future IPC. `session open/exec/write/resize/signal/cancel/close` map to the same protocol. the CLI displays the normalized command + effects BEFORE any password prompt, and never echoes the password anywhere.

## verification

- 318 workspace tests on Linux + macOS + Windows (parallel matrix)
- clippy `-D warnings`, fmt check, docs with broken-link denial, bench compilation, cargo-deny, RustSec
- adversarial suite: 30-iteration approve/cancel race hammer, 8-thread duplicate-ID flood, 20,000-step PRNG state-machine fuzz, real-child sentinel-secret leak proof
- honest benchmarks: policy/pipe primitives measured separately from process spawn + wasmtime startup (which are NOT sub-ms and we don't pretend they are)

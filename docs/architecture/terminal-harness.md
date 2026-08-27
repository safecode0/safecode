# 🖥️ terminal harness & human-only elevation

- **status:** accepted + implemented (Phase 1 shell harness)
- **date:** 2026-08-19

## context

ferrous needed a persistent terminal harness that gives **humans** bash-like power and gives **agents** structured, capability-enforced execution — without ever exposing the human's password or ambient host authority to the model.

the substrate already existed (wasmtime/WASI P2 runtime, capability grants, broker with approval + audit, native PTY backend). what was missing was the harness: persistent sessions, a typed command IR, a safe bash-like parser, builtins, pipelines, and ONE protocol that the CLI, AI tools, and the future UI all consume.

## the decisions

### 1. typed shell IR
human shell text and AI tool calls both compile into **one IR**. every program, argument, redirect, and cwd is a separate typed value — shell metacharacters can *never* be reinterpreted by a host shell. every plan hashes to a canonical `CommandDigest` (SHA-256 over a versioned canonical byte encoding), so approvals + audit records bind to the exact plan that ran.

### 2. safe bash-like parser
supports quoting, escaping, simple commands, `|`, `&&`, `||`, `;`, redirection, background `&`. **explicitly rejects** command substitution, arithmetic expansion, here-docs, aliases, startup files, functions, `eval`, and unbounded shell escapes — with clear errors. the parser never emits a shell command string. it's also **bounded**: max input bytes, max tokens, max pipeline stages, max statements. untrusted input can't exhaust memory.

### 3. persistent sessions
a `TerminalSession` owns cwd, an environment overlay, a bounded job table, and a lease slot. `cd` changes only session state, never the host cwd (there's a test proving the host cwd is untouched). builtins operate on capability-relative paths with symlink-aware resolution.

### 4. human-only elevation
this is the crown jewel. an `ElevationRequest` describes the exact effects + capability delta — **it contains no password field**. a `HumanApprovalAuthority` verifies a human but takes no password and returns no proof. the broker mints a short-lived, digest-bound `ApprovalLease` ONLY after verification succeeds. the agent API exposes only a redacted `PendingApproval` — it cannot construct, submit, derive, replay, or widen a lease. the password lives in `profiles-vault` (argon2) behind the trusted CLI authority, never in agent-facing APIs.

### 5. execution
- builtins run in-process
- external programs run by **direct argv** through the native policy adapter (never `sh -c`)
- WASI components run through embedded wasmtime
- pipelines use bounded channels with backpressure
- `&&`/`||` short-circuit on exit status
- redirections target capability-checked files
- background jobs are owned by the session and killed on close

### 6. single protocol
the CLI, the AI terminal tool, and the future UI all consume the same `SessionEvent` protocol. wterm wiring uses a WebSocket bridge with binary framing for terminal bytes and JSON for session metadata.

## consequences

- agents get structured execution by default; humans get familiar shell syntax; both land in the same capability broker.
- the password **never** enters `CommandRequest`, events, child environments, logs, audit records, or model context.
- native shells (`bash -c`, PowerShell, cmd) remain an explicit, approval-gated escape hatch — never synthesized by parsing.
- unsupported OS sandbox features return typed denials; nothing falls back to ambient execution.
- passing tests are evidence, not proof of zero bugs. adversarial + fuzz coverage required before claiming it's hardened (and we did exactly that — see the engineering docs).

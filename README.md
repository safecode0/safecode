# Ferrous

A capability-sandboxed command runtime and shell, built in Rust on an embedded
Wasmtime (WASI Preview 2 / component model) engine. The idea is simple: any
code that runs in here — a WASI component, an approved native command, a shell
pipeline — gets exactly the filesystem, environment, and network authority it
was explicitly granted, and nothing else.

**Status:** the Phase 1 execution core is implemented and verified from the
CLI. See [`docs/`](docs/) for the architecture decisions, engineering write-ups,
and the risk register.

## What's implemented

- **`wasi-runtime`** — an embedded Wasmtime (WASI preview 2 / component model)
  runtime with a capability-based sandbox:
  - explicit, typed capability grants (filesystem roots, environment
    allowlists, loopback ports, native execution);
  - deny-by-default posture: missing grants fail closed, with no ambient
    filesystem, environment, or network authority;
  - resource limits (memory, output budget, wall-clock timeout, instruction
    fuel) and cooperative cancellation;
  - a serialized action broker with human-in-the-loop approval for risky
    actions and an audit trail;
  - an approval-gated native PTY backend on Unix, using direct argv,
    allowlisted environment variables, bounded output, cancellation, timeout,
    and process-tree cleanup; unsupported hosts return `unsupported` and never
    fall back to ambient execution;
  - `#![forbid(unsafe_code)]` throughout.
- **`ferrous`** — the headless CLI. `ferrous shell` runs explicitly selected
  WASI components (`run-wasi <component>`) and approved native PTY commands
  (`run-native --allow -- <program> [args]`); unknown input never falls
  through to a host shell.
- **`shared`** — cross-cutting types: error-handling convention and secrets
  that are unprintable by construction (no serialization of secret values).

## Repository layout

```text
crates/
  ferrous/          CLI binary (headless) — `ferrous shell`     [implemented]
  shared/           cross-cutting types: errors, secrets        [implemented]
  wasi-runtime/     WASI runtime + shell + sandbox framework    [implemented]
  agent-loop/       agent loop + tools                          [placeholder]
  context-index/    code graph + context                        [placeholder]
  model-client/     model client                                 [placeholder]
  profiles-vault/   profiles + credentials                       [placeholder]
  router/           routing                                      [placeholder]
  search/           search                                       [placeholder]
  services/         services                                     [placeholder]
docs/
  architecture/     the decisions and why                       [read these first]
  engineering/      the builds: runtime, shell, PTY, audit
  risk-register.md  every risk + the test that kills it
```

## Build & run

```bash
cargo build --workspace
cargo run -p ferrous -- shell          # interactive shell (Phase 1)
```

## Quality gates

```bash
cargo fmt --all -- --check
cargo clippy --workspace --all-targets --all-features -- -D warnings
cargo test --workspace --all-features
cargo bench --workspace --no-run
cargo deny check licenses advisories bans sources   # requires `cargo-deny`
```

CI runs these gates on every push and pull request, on Linux, macOS, and
Windows.

## Security model

Deny-by-default, three layers:

1. **The component boundary** — only validated WASI components are admitted;
   no AOT artifacts or serialized modules.
2. **The capability boundary** — filesystem, environment, and network are
   granted explicitly and enforced by the host; guests get no handle outside
   their grants.
3. **The resource boundary** — memory, fuel, output, and wall-clock limits
   bound every guest, with cancellation as the escape hatch.

See [`docs/architecture/security-model.md`](docs/architecture/security-model.md).

## License

MIT OR Apache-2.0. Dependencies are permissive-only (see
[`docs/architecture/dependency-policy.md`](docs/architecture/dependency-policy.md)).

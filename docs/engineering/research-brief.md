# 🔬 terminal harness research brief — the homework we actually did

- **date:** 2026-08-19
- **status:** research done, then fully implemented (every conclusion below is now code)

before we built the shell, we did the research. like actually reading vendor docs, POSIX specs, and security papers — not vibes. here's what we learned and what we did about it.

## what we studied

### manus-style computer environments
manus describes an autonomous agent operating in a complete sandbox: virtual computer, persistent filesystem, software install, custom tools.

**→ implication:** persistent session state, not isolated one-shot commands. you need a durable workspace with a cwd, env policy, files, processes, network policy, cancellation, and recoverable results. that's exactly what `TerminalSession` is.

### AI coding harnesses (claude code, codex, cursor)
they all separate read-only commands from mutations, use working-directory boundaries, sandboxed execution, allowlists, and explicit handling of suspicious commands.

**→ implication:** TWO interfaces over ONE execution kernel — a structured tool API for agents and a human-friendly shell language for interactive use. the AI shouldn't have to serialize everything as an opaque shell string. typed tools are more precise, cheaper to authorize, easier to audit.

### shell semantics (POSIX + bash + nushell)
POSIX defines shell as a language pipeline: tokenize → parse → expand → redirect → execute → collect status. quoting, parameter expansion, command substitution, arithmetic, aliases, functions, here-docs are DISTINCT semantics, not one string-splitting step.

**critical fact:** GNU Bash is **GPLv3** — it conflicts with our permissive-only policy. not suitable for bundling, period.

**→ implication:** build a Safecode Shell language with a deliberately specified portable subset. don't claim exact bash compatibility. offer a user-installed native bash escape hatch (explicitly elevated + policy enforced) for people who need exact bash.

### WASI + wasmtime
WASI is capability-oriented and modular. Preview 2 provides virtualizable APIs through the component model. wasmtime: instances must import all external functionality, have memory isolation, use explicit fs capabilities. ALSO: terminal output with ANSI/control sequences can have side effects when rendered — needs filtering or safe rendering.

**→ implication:** WASI is the correct default for untrusted AI-owned tools. treat terminal output as DATA before rendering it to a human terminal.

### native process containment
- Linux **Landlock**: unprivileged self-restriction, stackable, inherited by descendants, ABI-dependent
- Windows **Job Objects**: process-tree management, resource limits, kill-on-close
- Windows **ConPTY**: bidirectional pseudoconsole transport (documented threading/deadlock requirements)
- Apple **App Sandbox**: platform mechanism for restricting system resources

**→ implication:** in-process capability checks are necessary but insufficient for native children. each OS needs an enforcement adapter. if the adapter can't enforce a requested policy → **fail closed**, never ambient fallback.

### package installation & supply chain
npm lifecycle scripts (`preinstall`, `install`, `postinstall`, `prepare`) run through a shell — a seemingly ordinary install can execute arbitrary package code. lockfiles/integrity/signatures/provenance exist but don't prove package code is benign. trivy is a scanner, not a live command firewall.

**→ implication:** staged install workflow: resolve + display plan → lockfile required → download to quarantine with narrow network policy → verify integrity → scan → show lifecycle scripts before elevation → run scripts in a disposable sandbox with no secrets + restricted network → promote atomically or roll back. `--ignore-scripts` as default.

### AI-specific threats
OWASP: prompt injection, insecure output handling, supply-chain risk, insecure plugins, sensitive-info disclosure, excessive agency. **indirect prompt injection** can arrive through repo files, command output, web pages, package metadata, test failures.

**→ implication:** command output + fetched content are UNTRUSTED DATA, never new policy. the model can never approve its own elevation, turn output into a capability, or widen an existing grant. only the human/profile policy can approve elevation.

## the power envelope we chose

| tier | what | authority needed |
| --- | --- | --- |
| **A** | `pwd`, `ls`, `cat`, `git status`, reads, pure computation | none (auto) |
| **B** | `cd`, `mkdir`, patches, local tests, `git add` | workspace-scoped, or automatic if policy allows |
| **C** | package install, network, native processes, git push, env/creds, preview ports | scoped elevation request |
| **D** | system dirs, `sudo`, unrestricted shell eval, raw devices, credential stores | denied by default, separate human mode only |

## password elevation design

step-up mechanism like `sudo`, BUT it mints a **capability lease** instead of disabling the security model. the human sees the parsed action + its effects, authenticates through the local vault path, and gets a short-lived, non-transferable lease bound to: session identity, exact action digest, capability scope, resource limits, expiration + revocation.

if the AI changes `npm install` into a compound command with an unapproved network destination, the digest + scope change → prior approval does NOT apply. this is tested. approve-then-change gets rejected.

## conclusions (all implemented)

1. **no embedded GNU Bash** (GPLv3, wrong license, doesn't become safe in a WASI host)
2. **no trivy-as-firewall** — scanners at import/install/promotion boundaries, capabilities for execution
3. **safecode shell = bash-like language + execution kernel** with explicit semantics + structured IR
4. **exact shell compat = escape hatch only**, through an explicitly elevated native adapter
5. **password elevation but scoped** — unlocks a temp lease, never unrestricted ambient authority
6. **WASI is the default AI path**; native is a separate adapter that fails closed
7. **security boundary is BELOW the parser** — parsing improves UX, only sandbox/policy enforcement contains a bypass
8. **first implementation target = persistent session protocol**, not UI

## gated definition of done (all checked)

✅ same persistent protocol for human + AI clients
✅ `cd`, env overlays, stdin, pipelines, redirects, cancellation, resume
✅ safe builtins never spawn a host shell
✅ AI commands structured/parsed into IR before execution
✅ direct argv preserved at every native boundary
✅ capability checks enforced by WASI + OS adapters, not just UI prompts
✅ password elevation produces scoped, expiring, auditable leases; secrets never exposed
✅ prompt-injected output cannot grant authority
✅ terminal control bytes safe before reaching a renderer
✅ unix/windows/macos behavior verified or explicitly fail-closed
✅ unit + integration + property/fuzz + adversarial race + supply-chain fixture + replay + performance tests, honest benchmarks

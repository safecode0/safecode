# 🧵 native PTY backend — real processes, real argv, zero shell-string nonsense

- **status:** implemented + tested (unix), fail-closed elsewhere
- **stack:** portable-pty 0.9 (MIT, WezTerm), std::sync::mpsc

this replaced the fail-closed stub with a real, capability-gated PTY backend so `safecode shell` can run **approved** `bash`/`cargo`/`npm`/`git` sessions with streaming, input, cancellation, process-tree cleanup, and hard failure on unsupported hosts.

## the rules (non-negotiable)

- native execution requires an explicit grant (`allow_native_execution`) AND broker approval. **never ambient fallback.** (R11)
- direct argv via `CommandBuilder` — never `sh -c`. metacharacters are inert by construction. (R34)
- only grant-allowlisted env names reach the child via `selected_environment`. (R8)
- every session: bounded combined output budget (kill on overflow), wall-clock timeout (watchdog kill), cancellation (kill child → process group). (R23, R35)
- unsupported hosts return `UnsupportedOnHost`. they do not "try anyway." (R11)

## the architecture

```
NativeBackend::spawn(request) -> NativeSession (pty master + writer + killer)
        ↓
NativeSessionHandle { session + cancel + limits + event sender }
   ├── reader thread   (drains pty, forwards SessionEvent::Output, enforces budget)
   ├── watchdog thread (100ms tick: cancel flag or deadline → killer.kill())
   ├── input thread    (mpsc receiver → session.write_input)
   └── main thread     (polls try_exit_status → joins all threads → outcome)
```

the broker got a second worker thread + separate queue for native jobs, per-session input channels (`send_input`), and `BrokerError::NotNative` for people trying to send keystrokes to a WASI session. approval, audit, cancellation, output budgets, and the `SessionEvent` protocol are reused unchanged — one broker, two backends, same rules.

## the tests that prove it's not lying

- `spawn_uses_direct_argv_and_ignores_shell_metacharacters` — `echo "$(touch /tmp/pwned)"` prints the string literally; the file does NOT exist.
- `spawn_requires_explicit_grant` — no grant → `NativeNotGranted`.
- `session_cancel_kills_the_child` — `sleep 100`, cancel, prompt return, no lingering process.
- `session_cleanup_leaves_no_children` — `sh -c "sleep 100 &"`, cancel, the GRANDCHILD is gone too (process-group kill).
- `session_output_budget_kills_on_overflow` — writes past the budget → killed + `OutputLimit`, bounded memory.
- `unallowlisted_sentinel_secret_never_reaches_the_child` — plant a secret in the parent env, prove it's absent from the child unless explicitly granted.
- `submit_native_requires_approval_then_streams` — PendingApproval → approve → Started → Output → Exited{0}.
- `deny_native_never_spawns` — denied request writes no sentinel file, because it never ran.
- `send_input_to_wasi_session_errors_not_native` — right error for the wrong backend.

## platform posture

- **unix (linux + macOS):** supported + enforced — process-group kill, direct argv, env allowlist, watchdog. macOS gets full App-Sandbox-style policy enforcement later; until then it inherits the unix adapter behavior that's provably enforced.
- **windows:** fail-closed until the ConPTY + Job Object adapter exists with equivalent process-tree + policy tests. windows CI compiles + runs the fail-closed branch so we know it degrades correctly, not silently.

the contract is: "PTY works" ≠ "requested security policy is enforced." a host that can't prove it enforces the policy gets a typed denial. every child + descendant stays owned by the session supervisor. no orphans, no zombies, no excuses.

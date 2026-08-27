# ⚠️ risk register — every scary thing + how we killed it

this is the ledger of every execution-boundary risk we identified for the WASI + native runtime, with the exact mitigation and the exact test that proves it. if it's not in this table, it's either not a risk we found or we're about to add it. unix native PTY is enabled in Phase 1; unsupported hosts **fail closed** until their platform adapter exists + passes equivalent tests.

| ID | the scary thing | how we killed it | proof |
| --- | --- | --- | --- |
| R8 | native commands could inherit host secrets | `selected_environment` forwards ONLY names explicitly allowlisted by the capability grant | `native::tests::only_allowlisted_environment_reaches_the_child` |
| R11 | native request could bypass policy or fall back to ambient execution | native mode requires `allow_native_execution` + broker approval; unsupported hosts return `UnsupportedOnHost`; no ambient fallback exists | `native::tests::spawn_requires_explicit_grant`; `broker::tests::native_submission_requires_approval_then_streams_output` |
| R23 | PTY cancellation could leave the session running | session driver owns reader, watchdog, and input threads; cancellation + timeouts kill the child before joining all threads | `broker::tests::native_cancel_reports_cancelled_and_releases_the_session` |
| R34 | shell metacharacters could become unintended commands | native requests use structured `CommandBuilder` argv directly; the CLI tokenizer only groups arguments, never executes a shell string | `native::tests::spawn_uses_direct_argv_and_ignores_shell_metacharacters` |
| R35 | a cancelled command could orphan descendants | unix PTY sessions killed through the platform process-group boundary; teardown waits for the process + owned threads | `native_session::tests::cancellation_kills_the_native_process_tree` |
| R36 | session ID reuse while first session is live → detached cancellation/input ownership | IDs reserved under the same lock as capacity accounting; `enqueue` rejects duplicates with `DuplicateSession` | `broker::tests::concurrent_duplicate_id_submissions_allow_exactly_one_winner` |
| R37 | approve/deny/cancel racing on one parked session → two terminal outcomes or ghost run | only the `pending`-map owner may report a terminal; approve checks the cancel flag before dispatch; cancel falls through to the handle when already dispatched | `broker::tests::approve_cancel_race_yields_exactly_one_terminal_outcome` |
| R38 | WASI guest splitting output across stdout+stderr → up to 2x declared budget into the event stream | `cap_to_remaining` caps every drained chunk to the remaining combined budget before emission; both pipes hard-close on exhaustion | `pipe::tests::cap_to_remaining_never_allows_over_budget_emission` |
| R39 | unallowlisted host secret leaks into a native child | real-child sentinel test plants a secret in the parent env and proves it's absent from the child unless explicitly granted | `native::tests::unallowlisted_sentinel_secret_never_reaches_the_child` |

## the receipts 🧾

**adversarial run** passed on ubuntu + macOS + windows in parallel: formatting, clippy with warnings denied, **all 318 workspace tests**, docs, benchmark compilation, the criterion hot-path job, license + advisory checks, full cargo cache match. includes a 30-iteration approve/cancel race hammer, an 8-thread duplicate-ID flood behind a barrier, a 20,000-step deterministic PRNG state-machine fuzz, and a real-child sentinel-secret leak proof.

**performance run** measured these criterion medians on `ubuntu-latest`:

| hot path | median |
| --- | ---: |
| command request validation | 124.10 ns |
| allowlisted environment selection | 76.778 ns |
| loopback network policy check | 2.5681 ns |
| bounded pipe write + drain | 245.47 ns |

these are policy + streaming primitives, not end-to-end execution latency. process creation, PTY setup, wasmtime compilation, guest startup, and UI transport are intentionally separate measurements — we don't claim sub-ms where it's not true.

## what's still open (honestly)

- **windows ConPTY** + any future non-unix native policy adapter stay fail-closed until they have equivalent process-tree + policy tests. no fake support.
- the human-facing terminal **renderer** is a later UI-phase adapter. the backend contract is renderer-independent, which is the point.
- passing CI + microbenchmarks = evidence of tested behavior, not proof of zero bugs, formal verification, or certification. production still needs independent security review, threat modeling, OS sandboxing, secrets governance, operational controls. we wrote it down so nobody (including us) forgets.

# 🛡️ security & performance audit — we tried to break our own sandbox

- **status:** done. red team'd, hardened, benchmarked, shipped
- **stack:** Rust 2024, wasmtime 47 / WASI P2, portable-pty 0.9, criterion 0.5, GitHub Actions matrix

this was the pass where we stopped building features and started hunting for ways to break the authority model, exhaust resources, race the lifecycle, and drift across platforms. every fix below got a **failing regression test first** (TDD, no exceptions), then the fix, then cloud CI proof.

## what we audited + fixed

### 1. broker identity & approval transitions — now race-safe

- session IDs reserved under the same lock as capacity accounting. `enqueue` rejects duplicates with `DuplicateSession` instead of overwriting — the first session keeps its cancel handle, input route, sink, and audit identity. (tested with an 8-thread flood behind a barrier)
- approve/deny/cancel racing on a parked session yields **exactly one terminal outcome** — no ghost runs, no double reports. (30-iteration race hammer)
- approval dispatch failure cleans up the pending job + handle + input route via ONE helper — no path leaves debris.

### 2. output budgets — hard caps, no over-emission

- a guest splitting output across stdout+stderr can no longer push up to 2x the declared budget: `cap_to_remaining` caps every drained chunk to the remaining **combined** budget before emission, and both pipes hard-close the instant the budget dies.
- native chunks larger than the remaining budget get truncated to the budget, the child is killed, and the terminal result is a specific `OutputLimit` — not a misleading generic failure.
- `len() + len()` arithmetic replaced with checked/saturating accounting. no `usize` overflow bombs.
- already-buffered bytes are preserved while further writes are blocked. bounded memory, always.

### 3. native process lifecycle & environment

- real-child **sentinel-secret test**: a secret planted in the parent environment provably never reaches the child unless explicitly granted.
- invalid PTY input / closed writer / broken reader → session terminates + reports an error, never silently continues.
- `CommandBuilder` audited against its 0.9 docs: only caller-set env vars reach the child — locked in with a comment + test.
- ignored input/read errors replaced with an owned stop/error signal. timeout, cancellation, output overflow, child exit, and thread join all share ONE teardown path.
- unix process-group assertions; windows tests guarded so they never accidentally invoke `/bin/sh`, `/bin/cat`, `/bin/echo`, or `/usr/bin/env`.

### 4. platform matrix — verified, not assumed

full test matrix on `ubuntu-latest`, `macos-latest`, `windows-latest` with the cargo cache retained. unix-only process-behavior tests are gated; common WASI policy tests run everywhere; cargo-deny + RustSec stay a separate required job. every platform failure got investigated from logs, not masked with broad ignores.

## the numbers (criterion medians, ubuntu-latest)

| hot path | median |
| --- | ---: |
| command request validation | **124.10 ns** |
| allowlisted environment selection | **76.778 ns** |
| loopback network policy check | **2.5681 ns** |
| bounded pipe write + drain | **245.47 ns** |

these are policy + streaming primitives — genuinely sub-microsecond. we do NOT claim process creation, PTY setup, wasmtime compilation, guest startup, or UI transport are sub-ms; those are deliberately separate measurements. honest benchmarks or nothing.

## adversarial evidence (the receipts)

the green CI run (318 tests) passed on all three OSes in parallel: fmt, clippy `-D warnings`, all workspace tests, docs, benchmark compilation, the criterion hot-path job, license + advisory checks, full cargo cache hit. the adversarial suite includes:

- 30-iteration approve/cancel race hammer
- 8-thread duplicate-ID flood behind a barrier
- 20,000-step deterministic PRNG state-machine fuzz
- real-child sentinel-secret leak proof

## the honest caveat (we're not delusional)

passing CI + microbenchmarks are evidence of tested behavior — **not** proof of zero bugs, formal verification, or bank/healthcare certification. production deployment still requires independent security review, threat modeling, OS-level sandboxing, secrets governance, and operational controls. that's a future phase, and we said so in writing.

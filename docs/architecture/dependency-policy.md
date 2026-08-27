# 📜 dependency policy — permissive or it's getting deleted

- **status:** locked in, enforced in CI
- **date:** 2026-08-13

## the problem

ferrous might go commercial, might go open source, honestly we haven't decided yet and that's the point. if we pulled a copyleft dependency (GPL/AGPL/LGPL/SSPL/MPL) we'd be FORCED to make that call early because copyleft slaps redistribution obligations on us. so the rule is simple:

> **only permissive licenses, ever. if cargo-deny flags it, we don't ship it.**

## the allowlist

`MIT`, `Apache-2.0`, `Apache-2.0 WITH LLVM-exception`, `BSD-2-Clause`, `BSD-3-Clause`, `ISC`, `Zlib`, `0BSD`, `Unlicense`, `MIT-0`, `Unicode-3.0`, `Unicode-DFS-2016`.

note the `Apache-2.0 WITH LLVM-exception` — that's because wasmtime + cranelift are LLVM-derived, and cargo-deny treats it as its own SPDX expression. it doesn't bring copyleft obligations, so the policy spirit is unchanged. we just had to be precise about it.

## how it's enforced

- `deny.toml` defines the allowlist + `confidence-threshold = 0.8` (doubt = deny).
- `cargo-deny` runs in CI: `cargo deny check licenses advisories bans sources`.
- wildcards are denied. multiple versions warn. yanked = deny, unmaintained = workspace-level fail.
- unknown registry or unknown git sources are flat-out denied.
- any per-crate exception requires a whole new ADR. we don't do exceptions casually.

## consequences

- GPL-only browser engines/editors/libraries? off-limits. deal with it.
- some referenced components might be copyleft (looking at you, canvas lib) — that means we BUILD a permissive equivalent instead of vendoring. future research trigger.
- slightly narrower library choice, but we keep both the commercial AND open-source paths viable with zero rework. worth it, no cap.

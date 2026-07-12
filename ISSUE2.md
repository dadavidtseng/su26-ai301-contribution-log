# Contribution 2: Formatter not idempotent on member chains with object-literal args

**Contribution Number:** 2  
**Student:** Yu-Wei Tseng  
**Issue:** [biome#10531 — Formatter still not idempotent on member chains with object-literal args (2.4.16)](https://github.com/biomejs/biome/issues/10531)  
**Status:** Phase II Complete

---

## Why I Chose This Issue

I chose this issue because it exposes a correctness bug in Biome's JavaScript formatter — a tool I use and want to understand more deeply. The formatter is not idempotent for member/method chains whose final call argument is an object literal: `format(format(x)) !== format(x)`. The first pass fully expands the chain (one call per line), but a second pass collapses it back inline and only breaks the object argument. This means `biome format --write` followed by `biome check` fails on freshly-formatted code, which breaks CI pipelines.

I'm interested in this because:

1. **It's a real-world correctness issue.** Formatter idempotency is a fundamental invariant — violating it causes CI flakes and erodes trust in the tool. The issue was reported as far back as #4383, closed as fixed, and has resurfaced on v2.4.16.
2. **It matches my skills and growth goals.** Biome is written in Rust, which I gained hands-on experience with during my first contribution (lakekeeper#1064). This issue lets me deepen that Rust expertise while learning about formatter internals — specifically how member chain breaking decisions interact with the IR printing algorithm.
3. **The scope is well-bounded.** The bug is isolated to the member chain formatting logic and has a clear reproduction case. The maintainer (`@dyc3`) confirmed the issue and approved me to work on it.
4. **The project is actively maintained.** Biome has frequent releases, responsive maintainers, and clear contribution guidelines.

The maintainer confirmed my interest with a thumbs-up: [comment link](https://github.com/biomejs/biome/issues/10531#issuecomment-4758768392).

---

## Understanding the Issue

### Problem Description

The JS formatter is not idempotent for member/method chains whose final call argument is an object literal that exceeds `lineWidth`. The formatter produces different output on the first vs second pass, violating the fundamental invariant `format(format(x)) === format(x)`. This causes CI pipelines to fail when `biome check` runs after `biome format --write`.

### Expected Behavior

`format(format(x)) === format(x)` — a single formatting pass should produce the stable (collapsed) output directly.

### Current Behavior

- **Pass 1**: Expands the chain — one method per line with deep indentation
- **Pass 2**: Collapses the chain back inline, breaking only the object argument
- **Pass 3** === Pass 2 (stable)

A single `biome format --write` is insufficient; a follow-up `biome check` fails on freshly-formatted code.

### Affected Components

- `crates/biome_js_formatter/src/utils/member_chain/mod.rs` — primary bug site: the `expand_parent()` + `best_fitting!` decision at lines 390–404
- `crates/biome_js_formatter/src/utils/member_chain/groups.rs` — `MemberChainGroup::will_break()` at lines 190–208
- `crates/biome_js_formatter/src/js/expressions/call_arguments.rs` — `grouped_breaks` decision and `write_grouped_arguments`
- `crates/biome_js_formatter/src/utils/object_like.rs` — `should_expand = members_have_leading_newline()` (source of cross-pass divergence)

---

## Reproduction Process

### Environment Setup

- **OS:** Windows 11 Home (10.0.26200)
- **Rust:** 1.96.1 (stable), installed via rustup; version pinned by `rust-toolchain.toml`
- **Tools installed:** `cargo-binstall`, `cargo-insta`, `wasm-opt`, `cargo-deny`, `wasm-bindgen-cli 0.2.117` via `just install-tools`
- **Node:** pnpm 11.9.0 for JS workspace dependencies

**Setup challenges:**
1. `pnpm` was not installed globally — resolved with `npm install -g pnpm`
2. PowerShell execution policy blocked `pnpm.ps1` — resolved with `Set-ExecutionPolicy RemoteSigned -Scope CurrentUser`
3. Debug build (`cargo build --bin biome`) causes stack overflow on long member chains due to deep recursion without optimizations — must use `cargo build --bin biome --release` for the reproduction binary
4. Running biome from inside its own repo causes stack overflow (scans `node_modules`/`target`) — reproduction must be done in an isolated directory

### Steps to Reproduce

1. Build biome in release mode: `cargo build --bin biome --release`
2. Create an isolated directory (e.g., `C:\tmp\biome-repro`)
3. Create `biome.json` with `{ "formatter": { "lineWidth": 120 } }`
4. Create `repro.js` with the following content:
   ```js
   import { integer, pgTable } from "drizzle-orm/pg-core";

   const example = pgTable("example", {
   	id: integer().primaryKey().generatedByDefaultAsIdentity({ name: "example_id_seq", startWith: 1, increment: 1, minValue: 1, maxValue: 2147483647, cache: 1 }),
   });
   ```
5. Run `biome format --write repro.js` (Pass 1) — chain expands:
   ```js
   id: integer()
           .primaryKey()
           .generatedByDefaultAsIdentity({
                   name: "example_id_seq",
                   ...
           }),
   ```
6. Run `biome format --write repro.js` again (Pass 2) — chain collapses:
   ```js
   id: integer().primaryKey().generatedByDefaultAsIdentity({
           name: "example_id_seq",
           ...
   }),
   ```
7. Run `biome format --write repro.js` a third time (Pass 3) — **no changes** applied, confirming Pass 2 == Pass 3

### Reproduction Evidence

- **Working branch:** https://github.com/dadavidtseng/biome/tree/fix-issue-10531
- **My findings:** The bug triggers specifically on method chains (not property-only chains) where the final call's argument is a non-empty object literal that exceeds `lineWidth`. The formatter requires two passes to converge because the first pass's output changes the object literal's `members_have_leading_newline()` state, which alters the `grouped_breaks` decision on the second pass.

---

## Solution Approach

### Analysis

The root cause is a cascade across two formatting passes in the member chain formatter:

1. **Pass 1** (input has inline object literal): The chain + inline object exceeds `lineWidth`. The `best_fitting!` macro in `member_chain/mod.rs:403` chooses `format_expanded`, producing one-method-per-line output with the object literal expanded to `{\n...\n}`.

2. **Pass 2** (input has expanded object literal): Re-parsing the Pass 1 output, `object_like.rs` sees `members_have_leading_newline() = true` → `should_expand = true`. This propagates: `call_arguments.rs` sets `grouped_breaks = true` → arguments' `BestFitting` has the middle variant (with `GroupMode::Expand`) as `most_flat()` → `MemberChainGroup::will_break()` returns true → `member_chain/mod.rs:399` writes `expand_parent()` followed by `best_fitting!(format_one_line, format_expanded)`. The printer's `fits` checker encounters `GroupMode::Expand` inside the flat variant, pushes `PrintMode::Expanded`, and immediately returns `Fits::Yes` at the first soft line break — making `format_one_line` appear to fit when it shouldn't. The chain collapses.

The core issue: when `last_group().will_break()` is true, the member chain emits `expand_parent()` + `best_fitting!`, but the `best_fitting!` flat variant measurement doesn't properly account for the nested `GroupMode::Expand` element. The `Printer::fits` check with `must_be_flat = false` allows expanding groups to short-circuit via `Fits::Yes`, making the one-line variant always appear to fit.

### Proposed Solution

The most targeted fix is in `member_chain/mod.rs` at the decision point (lines 390–404). When `last_group().will_break()` is true (but `groups_should_break()` is false and `has_empty_line_before_tail` is false), the current code writes `expand_parent()` then falls through to `best_fitting!`. Instead, it should force the expanded layout directly — similar to how `groups_should_break() == true` is handled at line 391. This ensures the first pass produces the same stable output as the second pass.

### Implementation Plan

Using UMPIRE framework:

**Understand:** The formatter violates idempotency (`format(format(x)) != format(x)`) for method chains ending in a call with a non-empty object literal argument that exceeds `lineWidth`. Pass 1 expands the chain; Pass 2 collapses it. The stable output is the collapsed form (Pass 2).

**Match:** The codebase already handles forced expansion correctly when `groups_should_break()` returns true (line 391 in `mod.rs`). The fix should follow this same pattern for the `last_group().will_break()` case.

**Plan:**
1. In `crates/biome_js_formatter/src/utils/member_chain/mod.rs`, modify the `format_content` closure (lines 390–404) to force `format_expanded` when `last_group().will_break()` is true, rather than emitting `expand_parent()` + `best_fitting!`
2. Alternatively, adjust the `groups_should_break()` method (lines 249–306) to return true when the last call breaks due to a grouped object argument — this would be a more principled fix
3. Add snapshot tests using `cargo-insta` for:
   - The drizzle-orm reproduction case (method chain with object literal arg)
   - The Fastify-style reproduction case (`reply.code(409).send({...})`)
   - Edge cases: chains with multiple object args, chains within nested expressions
4. Run existing formatter tests (`just test-crate biome_js_formatter`) to verify no regressions

**Implement:** https://github.com/dadavidtseng/biome/tree/fix-issue-10531

**Review:** Will self-review against biome's `CONTRIBUTING.md`:
- Conventional commit messages (`fix:` prefix)
- AI-assisted contributions disclosed in PR description
- Run `just f` (formatting) and `just l` (linting) before submitting
- Create changeset via `just new-changeset`

**Evaluate:**
- The drizzle-orm and Fastify reproduction cases must produce identical output on Pass 1 and Pass 2
- All existing `biome_js_formatter` snapshot tests must pass (no regressions)
- Run `cargo-insta test` to verify new snapshots match expected output
- Manual test: `biome format --write` followed by `biome check` should report no issues

---

## Testing Strategy

*(To be completed in Phase III)*

---

## Implementation Notes

*(To be completed in Phase III)*

---

## Pull Request

*(To be completed in Phase IV)*

---

## Learnings & Reflections

*(To be completed in Phase IV)*

---

## Resources Used

- [Biome Issue #10531](https://github.com/biomejs/biome/issues/10531)
- [Biome Repository](https://github.com/biomejs/biome)
- [Related closed issue #4383](https://github.com/biomejs/biome/issues/4383)
- [Playground reproduction](https://biomejs.dev/playground/?lineWidth=120&tab=formatter&pane=Diagnostics)

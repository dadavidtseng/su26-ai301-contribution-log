# Contribution 3: noUselessTernary double space in quick fix

**Contribution Number:** 3  
**Student:** Yu-Wei Tseng  
**Issue:** [biome#11092 -- noUselessTernary double space in quick fix](https://github.com/biomejs/biome/issues/11092)  
**Status:** Phase IV Complete

---

## Why I Chose This Issue

I chose this issue because it is a confirmed formatting bug in a tool I have already contributed to successfully. The `noUselessTernary` rule's quick fix introduces an extraneous double space when replacing a ternary expression like `x > -1 ? true : false` with `x > -1`. Specifically, the output produces `x  > -1` (two spaces before `>`) instead of `x > -1`.

I'm interested in this because:

1. **It's a confirmed, visible bug.** The quick fix output is visually incorrect and would fail strict formatting checks. The maintainers have confirmed it (`S-Bug-confirmed`) and explicitly want community help (`S-Help-wanted`).
2. **I already know this codebase.** My second contribution (biome#10531) was also a Biome fix. I'm familiar with the project structure, build tooling, test patterns, and review process. This lets me move faster and with more confidence.
3. **The scope is surgical.** The bug is in the code action output for a single linter rule (`noUselessTernary`). The fix likely involves adjusting whitespace handling in the rule's fix generator -- probably a one-file change.
4. **The issue is fresh and unclaimed.** Filed on July 27, 2026 with no human comments (only an automated AgentScan bot), no assignees, and no linked pull requests. The `S-Help-wanted` label means no permission is needed to start work.

---

## Understanding the Issue

### Problem Description

The `noUselessTernary` lint rule's quick fix produces a double space before the `>` operator when simplifying a ternary expression. For example:

**Input:**
```javascript
const pauseEventLane = document.cookie.indexOf('cid_debug=false') > -1 ? true : false;
```

**Expected quick fix output:**
```javascript
const pauseEventLane = document.cookie.indexOf('cid_debug=false') > -1;
```

**Actual quick fix output:**
```javascript
const pauseEventLane = document.cookie.indexOf('cid_debug=false')  > -1;
```

Note the double space before `>` in the actual output.

### Affected Components

- The `noUselessTernary` rule's code action / fix generator in the Biome linter

---

## Reproduction Process

### Environment

- **OS:** Windows 11 Home 10.0.26200
- **Rust:** 1.93.0
- **Biome:** built from source (fork `main` synced to upstream `125802321c`)
- **Branch:** `fix-issue-11092` ([link](https://github.com/dadavidtseng/biome/tree/fix-issue-11092))

### Steps to Reproduce

1. Use the [Biome Playground](https://biomejs.dev/playground/?lintRules=noUselessTernary&tab=formatter&pane=Diagnostics&code=YwBvAG4AcwB0ACAAcABhAHUAcwBlAEUAdgBlAG4AdABMAGEAbgBlACAAPQAgAGQAbwBjAHUAbQBlAG4AdAAuAGMAbwBvAGsAaQBlAC4AaQBuAGQAZQB4AE8AZgAoACcAYwBpAGQAXwBkAGUAYgB1AGcAPQBmAGEAbABzAGUAJwApACAAPgAgAC0AMQAgAD8AIAB0AHIAdQBlACAAOgAgAGYAYQBsAHMAZQA7AA%3D%3D) with this input:
   ```javascript
   const pauseEventLane = document.cookie.indexOf('cid_debug=false') > -1 ? true : false;
   ```

2. Observe the quick fix output produces a double space before `>`:
   ```javascript
   const pauseEventLane = document.cookie.indexOf('cid_debug=false')  > -1;
   ```

3. Alternatively, the bug is already visible in existing test snapshots (`invalid.js.snap`):
   - Line 269: `foo··===·1` (double space before `===`)
   - Line 379: `x··instanceof·foo` (double space before `instanceof`)
   - Line 407: `'make'··in·car` (double space before `in`)

### Root Cause

In [no_useless_ternary.rs](https://github.com/biomejs/biome/blob/main/crates/biome_js_analyze/src/lint/complexity/no_useless_ternary.rs), the `action` method reconstructs binary/instanceof/in expressions using `make::token_decorated_with_space(operator)`, which creates a new operator token with leading AND trailing whitespace trivia. However, the `left` node extracted from the original AST already retains its trailing whitespace trivia (the space after the last token). The combination produces a double space:

```
left trailing trivia: " "  +  operator leading trivia: " "  =  "  " (double space)
```

Three call sites are affected (lines 159, 178, 188):
- `make::token_decorated_with_space(operator)` for `JS_BINARY_EXPRESSION`
- `make::token_decorated_with_space(T![instanceof])` for `JS_INSTANCEOF_EXPRESSION`
- `make::token_decorated_with_space(T![in])` for `JS_IN_EXPRESSION`

---

## Solution Approach

### UMPIRE Analysis

**U - Understand:** The `action` method in `no_useless_ternary.rs` creates new operator tokens with `make::token_decorated_with_space()`, which unconditionally adds leading + trailing whitespace. The `left` child node already has trailing whitespace from the original source, causing doubled spaces.

**M - Match:** The `invert_expression` function in the same file (line 269) uses `make::token(operator)` (bare token, no trivia decoration) and does not exhibit the double space bug. This is the correct pattern.

**P - Plan:** Replace `make::token_decorated_with_space(...)` with the original operator token from the AST in all three match arms. The original token already carries the correct trivia from the source code.

**I - Implement:** (Phase III) Change lines 149-159, 176-179, and 186-189:
```rust
// Before (all three arms):
make::token_decorated_with_space(operator)

// After: reuse the original operator token
node.test().ok()?.as_js_binary_expression()?.operator_token().ok()?
node.test().ok()?.as_js_instanceof_expression()?.instanceof_token().ok()?
node.test().ok()?.as_js_in_expression()?.in_token().ok()?
```

**R - Review:** The fix is minimal (3 line changes) and isolated to the `action` method. It follows the same approach already used by `invert_expression` in the same file. All operator tokens are preserved from the original AST, so original formatting is maintained.

**E - Evaluate:** After the fix:
- The issue reporter's case produces `document.cookie.indexOf('cid_debug=false') > -1;` (single space)
- Existing test snapshots update to remove the double-space artifacts
- Add a new test case with the reporter's reproduction input

### Files to Modify

| File | Change |
|------|--------|
| `crates/biome_js_analyze/src/lint/complexity/no_useless_ternary.rs` | Use original operator tokens instead of `make::token_decorated_with_space` in 3 match arms |
| `crates/biome_js_analyze/tests/specs/complexity/noUselessTernary/invalid.js` | Add reproduction case from issue |
| `*.snap` files | Update snapshots via `cargo insta test --accept` |

### Risk Assessment

- **Low risk.** The change preserves the original operator token from the AST instead of creating a synthetic one. This is the same approach `invert_expression` already uses in the same file.
- **Snapshot changes expected.** Existing double-space artifacts in `invalid.js.snap` and `invalid_without_trivia.js.snap` will be corrected. The no-trivia case will no longer add spaces (e.g., `foo===1` stays as-is instead of becoming `foo === 1`), which is correct behavior -- code actions should not reformulate whitespace; the formatter handles that.

---

## Testing Strategy

### Existing Tests

All 3 existing test suites pass after the fix:
- `valid.js` -- valid cases that should NOT trigger the rule (unchanged)
- `invalid.js` -- invalid cases with trivia (spaces). Previously had double-space artifacts in snapshots; now corrected.
- `invalid_without_trivia.js` -- invalid cases without trivia. Previously used `token_decorated_with_space` to inject spacing; now preserves original (no-space) formatting.

### New Test Case

Added a regression test in `invalid.js` matching the issue reporter's reproduction:
```javascript
// Regression test: no double space before operator (biome#11092)
const pauseEventLane = document.cookie.indexOf('cid_debug=false') > -1 ? true : false;
```

The snapshot confirms the fix output is `document.cookie.indexOf('cid_debug=false') > -1` (single space, no double space).

### Validation

- `cargo test -p biome_js_analyze -- no_useless_ternary`: 3/3 pass
- `cargo fmt -- --check`: passes
- Snapshots updated via `cargo insta test --accept`

---

## Implementation Notes

### Files Modified

| File | Change |
|------|--------|
| `crates/biome_js_analyze/src/lint/complexity/no_useless_ternary.rs` | Replaced `make::token_decorated_with_space(...)` with original operator tokens in 3 match arms (binary, instanceof, in) |
| `crates/biome_js_analyze/tests/specs/complexity/noUselessTernary/invalid.js` | Added regression test case from issue #11092 |
| `crates/biome_js_analyze/tests/specs/complexity/noUselessTernary/invalid.js.snap` | Updated snapshot (double spaces removed, new test case added) |
| `crates/biome_js_analyze/tests/specs/complexity/noUselessTernary/invalid_without_trivia.js.snap` | Updated snapshot (no longer injects spaces via `token_decorated_with_space`) |

### What Changed

The `action` method's three operator match arms (`JS_BINARY_EXPRESSION`, `JS_INSTANCEOF_EXPRESSION`, `JS_IN_EXPRESSION`) were creating new operator tokens with `make::token_decorated_with_space()`, which unconditionally adds leading + trailing whitespace trivia. Since the `left` child node already retains its trailing trivia from the original AST, this doubled the spacing.

The fix reuses the original operator token from the AST instead of creating a synthetic one. This is the same approach already used by `invert_expression()` in the same file (line 269), so it's consistent with existing patterns.

### Challenges

- Understanding biome's CST trivia model: whitespace in biome's concrete syntax tree is attached as trailing trivia to the preceding token. This made the root cause non-obvious at first -- the double space comes from combining the `left` node's trailing trivia with the new token's leading trivia.
- The existing test snapshots already contained the bug (`foo··===·1`), confirming it wasn't limited to the reporter's case.

---

## Pull Request

**PR Link:** [biomejs/biome#11105](https://github.com/biomejs/biome/pull/11105)

**PR Description:** Fixes the `noUselessTernary` quick fix that produced a double space before the operator when simplifying ternary expressions. The root cause was `make::token_decorated_with_space()` adding leading whitespace to operator tokens when the left child node already had trailing whitespace from the original AST. The fix reuses the original operator token instead of creating a synthetic one.

**Status:** Awaiting review

**Maintainer Feedback:** *(to be updated as review progresses)*

---

## Learnings & Reflections

### Technical Learnings

- **CST trivia model:** Biome uses a Roslyn-style concrete syntax tree where whitespace is attached as "trivia" to tokens. Understanding that trailing trivia belongs to the preceding token was key to diagnosing the double-space bug -- the space after `)` was trailing trivia on `)`, and `token_decorated_with_space` added another leading space to `>`.
- **Code actions vs. formatters:** Code actions should preserve original formatting and make minimal changes. The formatter is responsible for whitespace normalization. The original code tried to "help" by adding spaces via `token_decorated_with_space`, but this conflicted with existing trivia.
- **Existing patterns as guides:** The `invert_expression` function in the same file already used the correct approach (bare `make::token` without trivia decoration). Looking at adjacent code in the same file is often the fastest way to find the right pattern.

### Process Learnings

- **Snapshot testing:** Biome's `cargo insta` workflow makes it easy to see exactly how output changes. The double-space bug was already visible in existing snapshots (`foo··===·1`) -- the snapshots were documenting the bug without anyone noticing.
- **Changeset workflow:** Biome requires a changeset file for user-facing changes, with specific formatting conventions (past tense, link to issue, link to rule docs). Reading existing changesets is the fastest way to learn the format.
- **Second contribution advantage:** Having contributed to biome before (biome#10531) made this contribution significantly faster. I already knew the build system, test patterns, PR conventions, and project structure.

### What I Would Do Differently

- Check the project's CONTRIBUTING.md for external PR policies **before** selecting an issue. This would have avoided the detour through Gradio (which paused external PRs) and uv (which got claimed while we were evaluating it).

---

## Resources Used

- [Biome Issue #11092](https://github.com/biomejs/biome/issues/11092)
- [Biome Repository](https://github.com/biomejs/biome)
- [Biome Playground Reproduction](https://biomejs.dev/playground/?lintRules=noUselessTernary&tab=formatter&pane=Diagnostics&code=YwBvAG4AcwB0ACAAcABhAHUAcwBlAEUAdgBlAG4AdABMAGEAbgBlACAAPQAgAGQAbwBjAHUAbQBlAG4AdAAuAGMAbwBvAGsAaQBlAC4AaQBuAGQAZQB4AE8AZgAoACcAYwBpAGQAXwBkAGUAYgB1AGcAPQBmAGEAbABzAGUAJwApACAAPgAgAC0AMQAgAD8AIAB0AHIAdQBlACAAOgAgAGYAYQBsAHMAZQA7AA%3D%3D)
- [Biome CONTRIBUTING.md](https://github.com/biomejs/biome/blob/main/CONTRIBUTING.md)

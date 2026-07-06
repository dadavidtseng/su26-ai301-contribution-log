# Contribution 2: Formatter not idempotent on member chains with object-literal args

**Contribution Number:** 2  
**Student:** Yu-Wei Tseng  
**Issue:** [biome#10531 — Formatter still not idempotent on member chains with object-literal args (2.4.16)](https://github.com/biomejs/biome/issues/10531)  
**Status:** Phase I Complete

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

*(To be completed in Phase II)*

### Expected Behavior

`format(format(x)) === format(x)` — a single formatting pass should produce the stable output. Currently two passes are required to converge.

### Current Behavior

Pass 1 expands the member chain (one call per line). Pass 2 collapses the chain back inline and breaks only the object argument. Pass 3 === Pass 2 (stable). This means a single `biome format --write` is insufficient, and a follow-up `biome check` fails on freshly-formatted code.

### Affected Components

*(To be identified in Phase II)*

---

## Reproduction Process

*(To be completed in Phase II)*

---

## Solution Approach

*(To be completed in Phase II)*

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

# Contribution 3: noUselessTernary double space in quick fix

**Contribution Number:** 3  
**Student:** Yu-Wei Tseng  
**Issue:** [biome#11092 -- noUselessTernary double space in quick fix](https://github.com/biomejs/biome/issues/11092)  
**Status:** Phase I Complete

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

- [Biome Issue #11092](https://github.com/biomejs/biome/issues/11092)
- [Biome Repository](https://github.com/biomejs/biome)
- [Biome Playground Reproduction](https://biomejs.dev/playground/?lintRules=noUselessTernary&tab=formatter&pane=Diagnostics&code=YwBvAG4AcwB0ACAAcABhAHUAcwBlAEUAdgBlAG4AdABMAGEAbgBlACAAPQAgAGQAbwBjAHUAbQBlAG4AdAAuAGMAbwBvAGsAaQBlAC4AaQBuAGQAZQB4AE8AZgAoACcAYwBpAGQAXwBkAGUAYgB1AGcAPQBmAGEAbABzAGUAJwApACAAPgAgAC0AMQAgAD8AIAB0AHIAdQBlACAAOgAgAGYAYQBsAHMAZQA7AA%3D%3D)
- [Biome CONTRIBUTING.md](https://github.com/biomejs/biome/blob/main/CONTRIBUTING.md)

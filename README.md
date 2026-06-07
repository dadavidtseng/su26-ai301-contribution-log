# Contribution 1: Deleted namespaces may leave traces on filesystems

**Contribution Number:** 1  
**Student:** Yu-Wei Tseng  
**Issue:** [lakekeeper#1064 — Deleted namespaces may leave traces on filesystems](https://github.com/lakekeeper/lakekeeper/issues/1064)  
**Status:** Phase I Complete

---

## Why I Chose This Issue

I chose this issue because it sits at the intersection of two areas I want to grow in: Rust and data lakehouse infrastructure. Lakekeeper is an Apache Iceberg REST catalog written in Rust, and working on it gives me hands-on experience with both the language and the data engineering ecosystem around Iceberg. I'm particularly interested in understanding how catalog services manage storage lifecycle — this issue is a concrete example of that, dealing with how namespace folders on filesystem-based storage (HDFS, ADLS) are cleaned up when a namespace is dropped.

The issue is also well-scoped and has clear maintainer guidance. The maintainer specified the exact acceptance criteria: delete the namespace folder only if (1) it is empty and (2) no other table in the same warehouse shares the same location. This bounded scope fits a 3–4 week contribution timeline, and the clear criteria make it straightforward to know when the fix is complete. The issue is labeled, unassigned, and the maintainer is active — all green flags from the issue selection checklist.

---

## Understanding the Issue

### Problem Description

[In your own words, what's broken or missing?]

### Expected Behavior

[What should happen?]

### Current Behavior

[What actually happens?]

### Affected Components

[Which parts of the codebase are involved?]

---

## Reproduction Process

### Environment Setup

[Notes on setting up your local development environment - challenges you faced, how you solved them]

### Steps to Reproduce

1. [Step 1]
2. [Step 2]
3. [Observed result]

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Lakekeeper Issue #1064](https://github.com/lakekeeper/lakekeeper/issues/1064)
- [Lakekeeper Repository](https://github.com/lakekeeper/lakekeeper)

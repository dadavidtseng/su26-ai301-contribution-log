# Contribution 3: MCP tool schemas never emit a `required` array

**Contribution Number:** 3  
**Student:** Yu-Wei Tseng  
**Issue:** [gradio#13670 -- MCP tool schemas never emit a `required` array, so every argument is advertised as optional](https://github.com/gradio-app/gradio/issues/13670)  
**Status:** Phase I Complete

---

## Why I Chose This Issue

I chose this issue because it exposes a correctness bug in Gradio's MCP server implementation -- a feature I find compelling as MCP (Model Context Protocol) is becoming the standard for tool interoperability between AI agents and applications. The bug is straightforward: `MCPServer.get_input_schema` in `gradio/mcp.py` builds JSON Schema `properties` for every published MCP tool but never emits a `required` array. Since JSON Schema treats an absent `required` as "nothing is required," every tool advertises all of its arguments as optional -- even arguments that have no default and that the function cannot run without.

I'm interested in this because:

1. **It's a real-world correctness issue.** MCP clients rely on the schema to know which arguments they must supply. Without `required`, a client can omit mandatory arguments and produce a call that is schema-valid but fails at runtime. This silently degrades the tool-calling experience for every Gradio MCP server deployment.
2. **It matches my skills and growth goals.** The fix is pure Python and involves JSON Schema -- concepts I'm comfortable with. This is my first contribution to a Hugging Face project, and I want to learn how Gradio's API/MCP infrastructure is structured.
3. **The scope is well-bounded.** The reporter identified the exact file (`gradio/mcp.py`, lines 1254-1272), the missing field (`required`), and pointed to an existing pattern in `gradio/cli/commands/skills.py` that already derives required-ness from `parameter_has_default`. The data is present; it just needs to be wired into the schema output.
4. **The issue is fresh and unclaimed.** Filed on July 27, 2026 with zero comments, no assignees, and no linked pull requests. The project (43k stars) is actively maintained by Hugging Face with responsive maintainers.

---

## Understanding the Issue

### Problem Description

Gradio's MCP server publishes JSON Schema for each tool endpoint but omits the `required` array. This makes every argument appear optional to MCP clients, even parameters with no default value that the function cannot operate without.

### Expected Behavior

The JSON Schema should include a `required` array listing all parameters where `parameter_has_default` is `False`, following JSON Schema specification.

### Current Behavior

The schema only contains `type` and `properties` -- no `required` array. MCP clients have no way to distinguish mandatory from optional arguments.

### Affected Components

- `gradio/mcp.py` -- `get_input_schema` method (lines 1254-1272): builds the schema but never populates `required`
- `gradio/cli/commands/skills.py` -- contains the existing pattern for checking `parameter_has_default`

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

- [Gradio Issue #13670](https://github.com/gradio-app/gradio/issues/13670)
- [Gradio Repository](https://github.com/gradio-app/gradio)
- [JSON Schema -- required keyword](https://json-schema.org/understanding-json-schema/reference/object#required)

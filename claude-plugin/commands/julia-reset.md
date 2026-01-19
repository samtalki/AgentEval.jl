---
name: julia-reset
description: Reset the Julia session by clearing user-defined variables
allowed-tools:
  - mcp__julia-eval__julia_reset
---

# Julia Reset Command

Reset the persistent Julia session by clearing user-defined variables.

## Instructions

1. Call the `julia_reset` MCP tool
2. Report what was cleared to the user
3. Remind the user that type definitions cannot be reset - a session restart is required for that

## Notes

- This is a "soft reset" - packages remain loaded
- Const bindings and type definitions cannot be cleared
- If the user needs to redefine types, they must restart their Claude Code session

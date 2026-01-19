---
name: julia-activate
description: Activate a Julia project/environment for the session
argument-hint: "<path>"
allowed-tools:
  - mcp__plugin_julia-eval_julia__julia_activate
  - mcp__plugin_julia-eval_julia__julia_pkg
---

# Julia Activate Command

Activate a Julia project or environment for the current session.

## Arguments

- `path` - Path to project directory, "." for current directory, or named environment like "@v1.10"

## Instructions

1. Parse the user's argument to determine the path:
   - If no argument or ".", activate the current working directory
   - If a path is given, use that path
   - If starts with "@", it's a named environment

2. Call `julia_activate` with the path

3. After activation, offer to run `julia_pkg(action="instantiate")` to install dependencies if the project has a Project.toml

## Examples

```
/julia-activate .
/julia-activate /path/to/MyProject
/julia-activate @v1.10
```

## Notes

- Activating a project changes where packages are installed/loaded from
- Use `julia_pkg(action="instantiate")` after activation to install dependencies
- The activated environment persists across `julia_reset` calls

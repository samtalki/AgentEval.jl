---
name: julia-evaluation
description: This skill should be used when the user asks to "run Julia code", "evaluate Julia", "use Julia", "julia_eval", mentions "persistent Julia session", "TTFX", or wants to work with Julia for data analysis, scientific computing, or package development. Provides best practices for using the julia_eval MCP tools effectively.
version: 0.2.0
---

# Julia Evaluation Best Practices

This skill provides guidance for using the persistent Julia session via the julia_eval MCP tools. The MCP server maintains a worker subprocess for code evaluation, eliminating the "Time to First X" (TTFX) startup penalty.

## Architecture

AgentEval uses a **worker subprocess model**:
- The MCP server runs in the main process
- Code evaluation happens in a spawned worker process
- `julia_reset` kills the worker and spawns a fresh one (true hard reset)
- `julia_activate` switches the worker's active project/environment

## Available Tools

| Tool | Purpose |
|------|---------|
| `julia_eval` | Evaluate Julia code with persistent state |
| `julia_reset` | **Hard reset** - kills worker, spawns fresh one |
| `julia_info` | Get session info (version, project, variables, worker ID) |
| `julia_pkg` | Manage packages (add, rm, status, update, instantiate, resolve) |
| `julia_activate` | Switch active project/environment |

## Critical: Show Code Before Evaluation

**Always display code in a readable format before calling `julia_eval`.** The MCP permission prompt shows code as an escaped string which is difficult for users to read and verify.

Correct workflow:

```
Running this Julia code:
```julia
x = [1, 2, 3, 4, 5]
mean(x)
```

[then call julia_eval with the code]
```

## Understanding TTFX (Time to First X)

The first call to `julia_eval` in a session may take several seconds due to:
- Julia's JIT compilation
- Package loading and precompilation

Subsequent calls are fast because the worker process stays alive with compiled code in memory.

## Session Persistence

Variables, functions, and loaded packages persist across `julia_eval` calls:

```julia
# First call
x = 42
f(n) = n^2
```

```julia
# Later call - x and f still exist
f(x)  # Returns 1764
```

## Hard Reset with julia_reset

Unlike soft resets that just clear variables, `julia_reset` **kills the worker process and spawns a fresh one**. This means:

- All variables are cleared
- All loaded packages are unloaded
- **Type definitions can be changed** (impossible with soft reset)
- The worker starts completely fresh

Use `julia_reset` when:
- You need to redefine a struct or type
- Something is in a bad state
- You want a completely clean slate

After reset, packages need to be reloaded with `using`.

## Environment Management

Julia best practice is to use project-specific environments. Use `julia_activate` to switch environments:

```
julia_activate(path=".")           # Current directory
julia_activate(path="/path/to/proj")  # Specific project
julia_activate(path="@v1.10")      # Named shared environment
```

After activation, install dependencies:
```
julia_pkg(action="instantiate")
```

The activated environment persists even across `julia_reset` calls.

## Package Management

Use `julia_pkg` for package operations:

**Adding packages:**
```
julia_pkg(action="add", packages="JSON, DataFrames, CSV")
```

**Checking installed packages:**
```
julia_pkg(action="status")
```

**Installing from Project.toml:**
```
julia_pkg(action="instantiate")
```

After adding a package, load it:
```julia
using JSON
```

## Error Handling

Common issues and solutions:

| Error | Cause | Solution |
|-------|-------|----------|
| `UndefVarError` | Variable not defined | Re-run earlier code or check spelling |
| `MethodError` | Wrong argument types | Check function signatures |
| `LoadError` | Package not installed | Use `julia_pkg(action="add", packages="...")` |
| `cannot redefine` | Type redefinition | Use `julia_reset` for a fresh worker |

## First-Time Setup

**When first using Julia in a session**, ask the user about their environment preference before running code:

> "Before we start, which Julia environment should I use?
> 1. **Current directory** - activate Project.toml in this folder (if it exists)
> 2. **Specific project** - provide a path to a Julia project
> 3. **Default** - use the global environment
>
> This determines where packages are installed and what dependencies are available."

Based on their answer:
- Option 1: `julia_activate(path=".")` then `julia_pkg(action="instantiate")`
- Option 2: `julia_activate(path="/their/path")` then `julia_pkg(action="instantiate")`
- Option 3: Proceed without activation (uses default environment)

## Practical Workflow

For a typical Julia task:

1. **First use**: Ask about environment (see above)
2. **Activate and install**: `julia_activate` + `julia_pkg(action="instantiate")`
3. **Show code to user**, then call `julia_eval`
4. **Build incrementally** - variables persist across calls
5. **Use `julia_reset`** if types need redefining or state is corrupted

## Multi-line Code

Multi-line code blocks work naturally:

```julia
function fibonacci(n)
    if n <= 1
        return n
    end
    return fibonacci(n-1) + fibonacci(n-2)
end

[fibonacci(i) for i in 1:10]
```

## Output Capture

Both return values and printed output are captured. Results are shown in this order for better visibility:
1. **Result** (or Error) - shown first for collapsed view
2. **Output** - any printed text
3. **Code** - the executed code (user already saw it before approving)

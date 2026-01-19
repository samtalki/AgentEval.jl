# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

AgentEval.jl is a Julia package providing persistent code evaluation for AI agents via MCP (Model Context Protocol) STDIO transport. It solves Julia's "Time to First X" (TTFX) problem by maintaining a persistent Julia worker subprocess, avoiding the 1-2 second startup penalty for each command.

## Build and Test Commands

```bash
# Run all tests
julia --project=. -e "using Pkg; Pkg.test()"

# Run a specific test file directly
julia --project=. test/test_eval.jl

# Start the MCP server manually (for debugging)
julia --project=. -e "using AgentEval; AgentEval.start_server()"

# Start server with a specific project activated
JULIA_EVAL_PROJECT=/path/to/project julia --project=. bin/julia-eval-server
```

## Architecture

### Worker Subprocess Model

AgentEval uses a **worker subprocess architecture** (via Distributed.jl):
- The MCP server runs in the main Julia process
- Code evaluation happens in a spawned worker process
- `julia_reset` kills the worker and spawns a fresh one (true hard reset)
- `julia_activate` switches the worker's active project/environment

This design enables true session reset (including type redefinitions) without restarting Claude Code.

### Key Functions

The package lives in `src/AgentEval.jl` with:

- **`ensure_worker!()`** - Ensures a worker process exists, creating one if needed
- **`kill_worker!()`** / **`reset_worker!()`** - Worker lifecycle management
- **`capture_eval_on_worker(code)`** - Evaluates code on the worker with output capture
- **`format_result(...)`** - Formats results (Result first, then Output, then Code)
- **`activate_project_on_worker!(path)`** - Switches the worker's environment
- **`run_pkg_action_on_worker(action, pkgs)`** - Package management on worker
- **`start_server()`** - Entry point that registers MCP tools and starts the server

### MCP Tools

Five tools registered via ModelContextProtocol.jl:

1. **`julia_eval`** - Evaluates Julia code with persistent state on the worker
2. **`julia_reset`** - **Hard reset**: kills worker, spawns fresh one (enables type redefinition)
3. **`julia_info`** - Returns session metadata (Julia version, project, variables, worker ID)
4. **`julia_pkg`** - Package management (add, rm, status, update, instantiate, resolve)
5. **`julia_activate`** - Switch active project/environment

### Key Design Decisions

- **Worker subprocess model**: Enables true hard reset with type redefinition
- **STDIO transport only**: No network ports for security
- **Environment persistence**: Activated environment survives reset
- **Result-first formatting**: Shows Result/Error first for better collapsed view UX

## Testing

Tests are in `test/test_eval.jl` covering:
- Code evaluation (arithmetic, variables, functions, multi-line)
- Output capture and error handling
- Result formatting
- Symbol management and protected symbol validation

**Note**: Tests may need updating for the worker subprocess model.

## Entry Point

`bin/julia-eval-server` is the executable script that loads the module and calls `start_server()`. It accepts `JULIA_EVAL_PROJECT` environment variable to activate a specific project on the worker.

## Plugin

The `claude-plugin/` directory contains a Claude Code plugin that:
- Auto-configures the MCP server (no manual `claude mcp add` needed)
- Provides commands: `/julia-reset`, `/julia-info`, `/julia-pkg`, `/julia-activate`
- Includes a skill with best practices for Julia evaluation

Install with:
```bash
claude --plugin-dir ./claude-plugin
```

## Using the MCP Tools

**Important**: When using `julia_eval`, always display the code in a readable format in your message BEFORE calling the tool. The MCP permission prompt shows code as an escaped string which is unreadable. By showing the code first, users can verify what will be executed before approving.

Example:
```
Running this Julia code:
```julia
x = 1 + 1
println("Hello!")
```

[then call julia_eval]
```

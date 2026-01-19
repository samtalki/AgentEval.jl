# julia-eval Plugin for Claude Code

This plugin provides persistent Julia code evaluation for Claude Code, eliminating the "Time to First X" (TTFX) startup penalty that normally occurs with each Julia invocation.

## Prerequisites

- Julia 1.10+ installed and available in PATH
- AgentEval.jl package (this repository)

## Installation

Add the plugin directory to Claude Code:

```bash
claude --plugin-dir /path/to/AgentEval.jl/claude-plugin
```

## What's Included

### MCP Server

The plugin automatically configures the `julia` MCP server which provides:

- `julia_eval` - Evaluate Julia code with persistent state
- `julia_reset` - **Hard reset**: kills worker, spawns fresh one
- `julia_info` - Get session information (including worker ID)
- `julia_pkg` - Manage packages
- `julia_activate` - Switch project/environment

### Commands

- `/julia-reset` - Kill and respawn the Julia worker (true reset)
- `/julia-info` - Show session information
- `/julia-pkg <action> [packages]` - Package management
- `/julia-activate <path>` - Activate a project/environment

### Skill

The `julia-evaluation` skill provides best practices guidance for:
- Showing code before evaluation (for readable permission prompts)
- Understanding TTFX behavior
- Working with session persistence
- Environment management
- When to use hard reset vs continuing

## Architecture

AgentEval uses a **worker subprocess model**:
- The MCP server runs in the main Julia process
- Code evaluation happens in a spawned worker (via Distributed.jl)
- `julia_reset` kills the worker and spawns a fresh one
- This enables true reset including type redefinitions

## Usage

Once installed, simply ask Claude to run Julia code:

> "Calculate the first 20 Fibonacci numbers in Julia"

On first use, Claude will ask about your environment preference:
1. Current directory (activate Project.toml if present)
2. Specific project path
3. Default/global environment

## Session Behavior

- Variables, functions, and packages persist across evaluations
- `julia_reset` provides a true hard reset (kills worker process)
- Type definitions CAN be changed after reset (unlike soft resets)
- Activated environment persists even across reset
- First evaluation is slow (TTFX), subsequent ones are fast

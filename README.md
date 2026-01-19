# AgentEval.jl

Persistent Julia code evaluation for AI agents via MCP (Model Context Protocol).

**The Problem:** Julia's "Time to First X" (TTFX) problem severely impacts AI agent workflows. Each `julia -e "..."` call incurs 1-2s startup + package loading + JIT compilation. AI agents like Claude Code spawn fresh Julia processes per command, wasting minutes of compute time.

**The Solution:** AgentEval provides a persistent Julia session via MCP STDIO transport. The Julia process stays alive, so you only pay the TTFX cost once.

## Why AgentEval?

| Feature | AgentEval | MCPRepl.jl | REPLicant.jl |
|---------|-----------|------------|--------------|
| Transport | **STDIO** | HTTP :3000 | TCP :8000+ |
| Auto-start | **Yes** | No (manual) | No (manual) |
| Network port | **None** | Yes | Yes |
| True hard reset | **Yes** | No | No |
| Type redefinition | **Yes** | No | No |
| Registry eligible | **Yes** | No (security) | No |
| Persistent | Yes | Yes | Yes |
| Solves TTFX | Yes | Yes | Yes |

### Key Advantages

- **STDIO Transport**: No network port opened, more secure, can be registered in Julia General
- **Auto-spawns**: Claude Code starts AgentEval automatically when needed
- **Persistent State**: Variables, functions, and loaded packages survive across calls
- **True Hard Reset**: Worker subprocess model allows type redefinitions without restarting Claude Code
- **Simple Setup**: One command to configure, or use the included plugin for zero-config

## Installation

```julia
using Pkg
Pkg.add(url="https://github.com/samtalki/AgentEval.jl")
```

Or for development:

```julia
Pkg.dev("https://github.com/samtalki/AgentEval.jl")
```

## Quick Start

### Option A: Use the Plugin (Recommended)

The easiest way to use AgentEval is via the included Claude Code plugin:

```bash
claude --plugin-dir /path/to/AgentEval.jl/claude-plugin
```

This provides:
- Auto-configured MCP server (no manual setup)
- Slash commands: `/julia-reset`, `/julia-info`, `/julia-pkg`, `/julia-activate`
- Best practices skill for Julia evaluation

### Option B: Manual MCP Configuration

```bash
claude mcp add julia -- julia --project=/path/to/AgentEval.jl -e "using AgentEval; AgentEval.start_server()"
```

Or using the provided script:

```bash
claude mcp add julia -- julia --project=/path/to/AgentEval.jl /path/to/AgentEval.jl/bin/julia-eval-server
```

### Using AgentEval

Start a new Claude Code session. The Julia MCP server will auto-start when Claude needs it.

Ask Claude to run Julia code:
> "Calculate the first 10 Fibonacci numbers in Julia"

Claude will use the `julia_eval` tool, and the result will appear instantly after the first call (which may take a few seconds for JIT compilation).

## Architecture

AgentEval uses a **worker subprocess model** via Distributed.jl:

```
┌─────────────────────────────────────────────────────┐
│ Claude Code                                          │
│   ↕ STDIO (MCP)                                      │
│ ┌─────────────────────────────────────────────────┐ │
│ │ AgentEval MCP Server (Main Process)             │ │
│ │   ↕ Distributed.jl                              │ │
│ │ ┌─────────────────────────────────────────────┐ │ │
│ │ │ Worker Process (code evaluation happens here)│ │ │
│ │ └─────────────────────────────────────────────┘ │ │
│ └─────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

**Why a worker subprocess?**
- `julia_reset` can kill and respawn the worker for a **true hard reset**
- Type/struct redefinitions work (impossible with in-process reset)
- The activated environment persists across resets
- Worker is spawned lazily on first use to avoid STDIO conflicts

## Tools Provided

### `julia_eval`

Evaluate Julia code in a persistent session.

```
# Example usage by Claude:
julia_eval(code = "x = 1 + 1")
# Result: 2

julia_eval(code = "x + 10")
# Result: 12 (x persists!)
```

Features:
- Variables and functions persist across calls
- Packages loaded once stay loaded
- Both return values and printed output are captured
- Errors are caught and reported with backtraces

### `julia_reset`

**Hard reset**: Kills the worker process and spawns a fresh one.

```
julia_reset()
# Worker killed and respawned - complete clean slate
```

This enables:
- Clearing all variables
- Unloading all packages
- **Redefining types/structs** (impossible with soft resets)
- Starting with a completely fresh Julia state

The activated environment persists across resets.

### `julia_info`

Get session information including worker process ID.

```
julia_info()
# Julia Version: 1.10.x
# Active Project: /path/to/project
# User Variables: x, y, my_function
# Loaded Modules: 42
# Worker ID: 2
```

### `julia_activate`

Switch the active Julia project/environment.

```
# Activate current directory
julia_activate(path=".")

# Activate a specific project
julia_activate(path="/path/to/MyProject")

# Activate a named shared environment
julia_activate(path="@v1.10")
```

After activation, install dependencies with:
```
julia_pkg(action="instantiate")
```

### `julia_pkg`

Manage Julia packages in the current environment.

```
# Add packages
julia_pkg(action="add", packages="JSON")
julia_pkg(action="add", packages="JSON, DataFrames, CSV")

# Remove packages
julia_pkg(action="rm", packages="OldPackage")

# Show package status
julia_pkg(action="status")

# Update all packages
julia_pkg(action="update")

# Update specific packages
julia_pkg(action="update", packages="JSON")

# Install dependencies from Project.toml/Manifest.toml
julia_pkg(action="instantiate")

# Resolve dependency graph
julia_pkg(action="resolve")
```

Actions:
- `add`: Install packages (packages parameter required)
- `rm`: Remove packages (packages parameter required)
- `status`: Show installed packages
- `update`: Update packages (all if packages not specified)
- `instantiate`: Download and precompile all dependencies from Project.toml/Manifest.toml
- `resolve`: Resolve dependency graph and update Manifest.toml

The `packages` parameter accepts space or comma-separated package names.

## Configuration

### Environment/Project Management

There are three ways to set the Julia environment:

**1. At runtime (recommended)**: Use `julia_activate` to switch environments dynamically:
```
julia_activate(path="/path/to/your/project")
julia_pkg(action="instantiate")
```

**2. Via environment variable**: Set `JULIA_EVAL_PROJECT` before starting:
```bash
JULIA_EVAL_PROJECT=/path/to/your/project claude mcp add julia -- julia --project=/path/to/AgentEval.jl -e "using AgentEval; AgentEval.start_server()"
```

**3. In code**: Pass directly to the server:
```julia
AgentEval.start_server(project_dir="/path/to/your/project")
```

The activated environment persists across `julia_reset` calls.

## Comparison with Alternatives

### vs Auton.jl

[Auton.jl](https://github.com/AntonOresten/Auton.jl) provides LLM-augmented REPL modes for human-in-the-loop workflows.

| Aspect | AgentEval | Auton.jl |
|--------|-----------|----------|
| Use case | AI agent automation | Human + LLM collaboration |
| Operator | AI agent (autonomous) | Human at keyboard |
| Interface | MCP tools (headless) | REPL modes (interactive) |
| LLM integration | Claude Code built-in | PromptingTools.jl (any model) |
| Setup | Plugin or one command | Startup.jl config |

**When to use Auton.jl**: You want LLM assistance while *you* work in the Julia REPL—context-aware suggestions, code generation, and iterative refinement with you in control.

**When to use AgentEval**: You want Claude Code to execute Julia code autonomously as part of a larger AI agent workflow, without requiring human presence at the REPL.

### vs ClaudeCodeSDK.jl

[ClaudeCodeSDK.jl](https://github.com/AtelierArith/ClaudeCodeSDK.jl) is an SDK for calling Claude Code **from** Julia. It's the opposite direction of AgentEval.

| Aspect | AgentEval | ClaudeCodeSDK.jl |
|--------|-----------|------------------|
| Direction | Claude → Julia | Julia → Claude |
| Purpose | Claude runs Julia code | Julia calls Claude |
| Use case | AI agent development | Automating Claude workflows |

**When to use ClaudeCodeSDK.jl**: You want to call Claude programmatically from Julia scripts or applications.

**When to use AgentEval**: You want Claude Code to execute Julia code in a persistent session.

### vs ModelContextProtocol.jl

[ModelContextProtocol.jl](https://github.com/JuliaSMLM/ModelContextProtocol.jl) is the MCP framework that AgentEval is built on. It provides the building blocks (`MCPTool`, `MCPResource`, `mcp_server`) for creating MCP servers.

| Aspect | AgentEval | ModelContextProtocol.jl |
|--------|-----------|-------------------------|
| Type | Ready-to-use MCP server | Framework for building servers |
| Setup | One command | Write custom tools |
| Flexibility | Julia eval only | Any tools you want |

**When to use ModelContextProtocol.jl**: You want to build custom MCP tools beyond code evaluation.

**When to use AgentEval**: You want persistent Julia evaluation without writing any MCP code.

### vs MCPRepl.jl

[MCPRepl.jl](https://github.com/hexaeder/MCPRepl.jl) is an excellent package that inspired AgentEval. Key differences:

| Aspect | AgentEval | MCPRepl.jl |
|--------|-----------|------------|
| Transport | STDIO | HTTP |
| Port required | No | Yes (:3000) |
| Manual startup | No | Yes |
| Shared REPL | No | Yes |
| Registry status | Eligible | Not eligible (security) |

**When to use MCPRepl.jl**: If you want to share a REPL with the AI agent (see each other's commands).

**When to use AgentEval**: If you want auto-start, no network port, or plan to distribute via Julia registry.

### vs REPLicant.jl

[REPLicant.jl](https://github.com/MichaelHatherly/REPLicant.jl) uses TCP sockets with a custom protocol (not MCP).

| Aspect | AgentEval | REPLicant.jl |
|--------|-----------|--------------|
| Protocol | MCP (standard) | Custom |
| Integration | `claude mcp add` | `just`/`nc` commands |
| Port required | No | Yes |

**When to use REPLicant.jl**: If you're using `just` for task automation.

**When to use AgentEval**: If you want standard MCP integration with Claude Code.

### vs DaemonMode.jl

[DaemonMode.jl](https://github.com/dmolina/DaemonMode.jl) is a client-daemon system for running Julia scripts faster.

| Aspect | AgentEval | DaemonMode.jl |
|--------|-----------|---------------|
| Protocol | MCP | Custom |
| Port required | No | Yes (:3000) |
| Julia 1.10+ | Yes | Broken |
| AI integration | Native | Requires wrapper |

**When to use DaemonMode.jl**: For general script acceleration (if using older Julia).

**When to use AgentEval**: For AI agent integration with modern Julia.

## Claude Code Plugin

The `claude-plugin/` directory contains a ready-to-use Claude Code plugin that provides:

### Auto-configured MCP Server
No need to manually run `claude mcp add`. The plugin configures the Julia MCP server automatically.

### Slash Commands
| Command | Description |
|---------|-------------|
| `/julia-reset` | Kill and respawn the Julia worker (hard reset) |
| `/julia-info` | Show session information |
| `/julia-pkg <action> [packages]` | Package management |
| `/julia-activate <path>` | Activate a project/environment |

### Best Practices Skill
The included skill teaches Claude:
- Always display code before calling `julia_eval` (for readable permission prompts)
- First-time environment setup dialogue
- When to use hard reset vs. continuing
- Error handling patterns

### Installation
```bash
claude --plugin-dir /path/to/AgentEval.jl/claude-plugin
```

See [claude-plugin/README.md](claude-plugin/README.md) for details.

## Security

See [SECURITY.md](SECURITY.md) for detailed security considerations.

**TL;DR**:
- STDIO transport = no network attack surface
- Code runs with user permissions
- Process terminates when Claude session ends
- No protection against malicious code (AI decides what to run)

## Development

### Running Tests

```bash
julia --project=. -e "using Pkg; Pkg.test()"
```

### Local Testing

```julia
using AgentEval
AgentEval.start_server()  # Blocks, waiting for MCP messages on stdin
```

## License

Apache License 2.0 - See [LICENSE](LICENSE) for details.

## Contributing

Contributions welcome! Please open an issue or PR on GitHub.

## Acknowledgments

- [ModelContextProtocol.jl](https://github.com/JuliaSMLM/ModelContextProtocol.jl) - MCP framework
- [MCPRepl.jl](https://github.com/hexaeder/MCPRepl.jl) - Inspiration and prior art
- [REPLicant.jl](https://github.com/MichaelHatherly/REPLicant.jl) - Alternative approach

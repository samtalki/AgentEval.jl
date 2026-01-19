# Changelog

All notable changes to AgentREPL.jl will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.5.0] - 2026-01-19

First public release of AgentREPL.jl.

### Features
- Persistent Julia REPL via MCP STDIO transport
- Worker subprocess model (Distributed.jl) for true hard reset
- Type redefinition support after reset
- Julia syntax highlighting (JuliaSyntaxHighlighting.jl stdlib)
- 7 MCP tools: eval, reset, info, pkg, activate, log_viewer, mode
- Claude Code plugin with auto-configuration
- Slash commands: /julia-reset, /julia-info, /julia-pkg, /julia-activate
- Environment variable configuration (JULIA_REPL_HIGHLIGHT, JULIA_REPL_OUTPUT_FORMAT)
- Log viewer for visual output monitoring
- Modular source code structure

### Package Management
- Full Pkg.jl integration: add, rm, status, update, instantiate, resolve, test, develop, free
- Project activation with shared environment support (@v1.10, @myenv)

### Requirements
- Julia 1.11+ (for JuliaSyntaxHighlighting stdlib)
- ModelContextProtocol.jl 0.4+

### Note
Tmux bidirectional REPL mode is deprecated. Use distributed mode with log_viewer instead.

[Unreleased]: https://github.com/samtalki/AgentREPL.jl/compare/v0.5.0...HEAD
[0.5.0]: https://github.com/samtalki/AgentREPL.jl/releases/tag/v0.5.0

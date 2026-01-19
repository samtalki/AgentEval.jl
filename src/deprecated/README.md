# Deprecated Code

This directory contains deprecated functionality that is disabled by default.

## tmux.jl - Bidirectional REPL (Deprecated)

The tmux mode provided a visible terminal REPL but has unfixable issues:
- Completion detection markers always visible in terminal output
- Marker pollution cannot be eliminated due to tmux architecture

**Alternative:** Use distributed mode (default) with the `log_viewer` tool:
```
log_viewer(mode="auto")
```

To force-enable tmux mode (not recommended):
```bash
export JULIA_REPL_ENABLE_TMUX=true
```

This code is retained for users who may need it despite the limitations.

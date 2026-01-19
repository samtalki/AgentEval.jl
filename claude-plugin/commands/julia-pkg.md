---
name: julia-pkg
description: Manage Julia packages (add, remove, status, update, instantiate)
argument-hint: "<action> [packages]"
allowed-tools:
  - mcp__julia-eval__julia_pkg
---

# Julia Package Management Command

Manage Julia packages in the current environment.

## Arguments

- `action` - One of: add, rm, status, update, instantiate, resolve
- `packages` - Package names (required for add/rm, optional for update)

## Instructions

Parse the user's arguments to determine the action and packages:

| User Input | Action | Packages |
|------------|--------|----------|
| `add JSON` | add | JSON |
| `add JSON DataFrames CSV` | add | JSON, DataFrames, CSV |
| `rm OldPackage` | rm | OldPackage |
| `status` | status | (none) |
| `update` | update | (none - updates all) |
| `update JSON` | update | JSON |
| `instantiate` | instantiate | (none) |
| `resolve` | resolve | (none) |

Call `julia_pkg` with the appropriate action and packages parameters.

## Examples

```
/julia-pkg add Plots
/julia-pkg status
/julia-pkg update
/julia-pkg instantiate
```

## Notes

- After adding packages, remind the user to load them with `using PackageName`
- The `instantiate` action installs dependencies from Project.toml/Manifest.toml
- The `resolve` action updates the Manifest.toml dependency graph

---
name: rust_debug_mcp_dap_servers
description: MCP servers for debugging Rust and other languages via the Debug Adapter Protocol (DAP); main option is Govinda-Fichtner/debugger-mcp using CodeLLDB for Rust
tags:
- rust
- mcp
- debugging
- dap
- lldb
- codelldb
- debugger
created_at: '2026-10-07T14:33:14.560783510+02:00'
---

# Rust debugging via MCP — DAP-bridge servers

## Key finding (2026-10-07)
The ecosystem converged on bridging MCP to the **Debug Adapter Protocol (DAP)** rather than wrapping specific debuggers. One server covers many languages; for Rust the adapter is **CodeLLDB** (LLDB-based).

## Main option: Govinda-Fichtner/debugger-mcp
- Rust/Tokio MCP server bridging MCP ↔ DAP. Repo: https://github.com/Govinda-Fichtner/debugger-mcp
- Languages: Python (debugpy), Ruby (rdbg), Node.js (vscode-js-debug), Go (delve), **Rust (CodeLLDB)**
- Capabilities: start/stop sessions, source breakpoints, step over/into/out, continue, pause, expression evaluation, stack traces, variable inspection
- NOT yet: conditional breakpoints, logpoints, exception breakpoints, multi-threaded, remote debugging (roadmap "Phase 4")
- Credibility: 193 unit tests, CI + integration matrix where real agents (Claude Code, Codex) autonomously debug in all 5 languages; per-language Dockerfiles (Dockerfile.rust)
- Install: `cargo build --release` then `./target/release/debugger_mcp serve`, or per-language Docker image
- Config example (Claude Desktop style): `{"mcpServers": {"debugger": {"command": "/path/to/debugger_mcp", "args": ["serve"]}}}`

## Alternative
- "DAP-MCP" — independent project, same concept (launch sessions, breakpoints, stepping). Less public documentation.

## Rust gotchas
- **Breakpoints not verifying = missing debug symbols.** Cargo `dev` profile has `debug = true` by default (cargo run/test fine). For release builds add `debug = true` under `[profile.release]` in Cargo.toml.
- Source paths must match exactly (matters in Docker: use container paths like /workspace/...).
- Security posture: executes local compiled binaries — same trust level as `cargo run`; no web-content prompt-injection surface (unlike browser MCP servers).

## Search notes
Finding these was non-trivial: queries like "rust debug mcp" returned nothing; "debug adapter protocol MCP server" worked. Use DAP as the search term.

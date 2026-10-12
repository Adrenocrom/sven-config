---
name: sven_rs_vs_python_sven
description: 'The sven-rs Rust workspace is a rewrite of the Python project Adrenocrom/sven; documents module mapping, feature divergence, GPL-vs-MIT license analysis, review rounds 1-3 (round 3 C1 ManPageTool arbitrary-file read FIXED 2026-10-09: validate_name() rejects leading ''-'' and any ''/'', unit-tested, README updated; still open: HTTP errors swallowed, backend casing, grep exit-1/BRE, unbounded history, no timeouts) and round 4 (2026-10-06): MCP client review — see skill sven_rs_mcp_client_review for findings (404-reinit MUST unhandled, Expect:100-continue breaks >1MB HTTP calls). ASYNC MIGRATION DONE 2026-10-06: Tool::execute async (see sven_rs_async_tool_migration), MCP HTTP transport now async reqwest (no curl).'
tags:
- sven
- rust
- python
- rewrite
- gpl
- license
- code-review
- argument-injection
created_at: '2026-09-23T13:46:53.356804004+02:00'
---

# sven-rs vs Python sven

Relationship between Adrenocrom/sven (Python original, GPL) and Adrenocrom/sven-rs (Rust rewrite, MIT).

## Module mapping
- Python `sven/` → Rust `src/sven/` (agent, backend, chat_history, config, macros, security, security_error, skills, term, tool, tool_registry, tools/)
- `main.py` → `src/main.rs` (clap REPL loop)
- Tool registry: 17 tools registered in `main.rs`, all via the `tool!` macro (`src/sven/macros.rs`) which desugars async trait methods into boxed futures (dyn-compatibility).

## License analysis
Python sven is GPL; sven-rs is MIT. Rewriting from scratch (not porting code) keeps MIT valid — the REWRITE_SPEC.md documents this. Do not copy Python source verbatim.

## Review rounds
- **Round 1** (2 critical, 3 high, 5 medium, 8 low): all fixed. Key: `file://` bypass in WebFetch.
- **Round 2** (1 critical): argument injection into `find`/`ls` via dash-leading `path` — fixed via `subprocess::as_operand()` (unit-tested), `--` markers for ls/ddgr, leading-dash rejection in ManPageTool.
- **Round 3** (2026-09-30, review.md): **C1 ManPageTool arbitrary-file read — FIXED 2026-10-09.** `man` opens any slash-containing argument as a file; `validate_name()` in `manpage_tool.rs` rejects leading `-` and any `/` before spawn. Verified against man-db's actual boundary (only slash-containing args are file-opened; `README.md`, `.`, `..` are page lookups; empty name harmless). Tests: unit tests for rejection/acceptance **plus an end-to-end `#[tokio::test]` through `Tool::execute`** asserting the error message — the unit tests alone would stay green if the `validate_name()` call were removed from the tool body. README security section + review.md annotated. **Audit note:** the running agent binary predates the fix — rebuild + restart needed to pick it up.
- **Round 3 still open:** M1 HTTP errors swallowed (agent silently does nothing on 404/401), M2 backend config casing (`"openai"` fails to parse → silent Ollama fallback), M3 grep exit-1-as-error, M4 grep BRE vs model-emitted ERE, M5 unbounded history (cap quietly raised 25→250), M6 no subprocess timeouts (man/ddgr/cargo/grep/find/ls unbounded), L1 `n:0` no-op success, L2 CompileTool case-sensitive enum + build-only (agent cannot run `cargo test`), L3 ManPageTool roff overstrikes, L4 params JSON printed to terminal, L5 FindTool exclusions anchored to `./`, L6 non-interactive mode undocumented, L7 skills-store symlink writes, L11 subprocess diagnostics in user locale (set LC_ALL=C), L12 hygiene grab-bag.
- **Round 4** (2026-10-06): MCP client review — see skill `sven_rs_mcp_client_review` (404-reinit MUST unhandled, Expect:100-continue breaks >1MB HTTP calls).

## Async migration
DONE 2026-10-06: `Tool::execute` is async (boxed future, see `src/sven/tool.rs`), MCP HTTP transport uses async reqwest (no curl subprocess).

## Key security patterns (keep consistent)
- Path confinement: `security::is_inside_cwd` in every path-taking tool (ReadTool, edit tools, ListFiles, GrepTool, FindTool), symlink-resolving.
- No shell anywhere: explicit argv vectors only.
- Every model-controlled operand guarded against being read as an option or file path: `as_operand()` (`./` prefix) for find, `--` for ls/ddgr, `validate_name()` for man (rejects `-` and `/`).
- WebFetch: `http://`/`https://` only.
- MCP HTTP transport: scheme allowlist, header-name validation (no `:`/control chars).

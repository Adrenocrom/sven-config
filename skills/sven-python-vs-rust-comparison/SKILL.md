---
name: sven_python_vs_rust_comparison
description: Feature comparison between the Python sven agent and the Rust sven-rs rewrite; corrected missing-feature list lives in note2.md in the sven-rs repo root (note.md is the earlier, partly outdated version)
tags:
- sven
- sven-rs
- feature-comparison
- rust
- python
- note2-md
created_at: '2026-10-07T10:07:34.857581069+02:00'
---

# sven vs sven-rs feature comparison

The workspace is `sven-rs` (Rust rewrite, github.com/Adrenocrom/sven-rs) of the
Python agent `sven` (github.com/Adrenocrom/sven). Comparisons of features the
Python version has but sven-rs lacks:

- `note.md` (repo root, written 2026-10-07) — first comparison.
- `note2.md` (repo root, written 2026-10-07) — **corrected, source-verified**
  comparison; supersedes note.md where they disagree.

Key missing features (details in note2.md):
1. FIFO task queue persisted to `tasks.json` (`task.py`, `tools/task.py`)
2. Conversation summarization when history exceeds `max_messages` (`core.py`)
3. Token usage tracking persisted to `tokens.json` — sven-rs prints per-turn
   counts (Ollama only) but has no persistence/lifetime totals
4. Cross-session chat history (`history.py` → `chat_history.json`)
5. Git `diff` tool (`tools/git.py`)
6. `touch` tool (`tools/touch.py`)
7. `replaceline` tool (line-number edit)
8. Per-file compile (sven-rs compiles whole projects only)
9. Config profiles (`SVEN_PROFILE`) — env-var overrides exist in Python but
   are never called (inactive)
10. Ollama `keep_alive` + `repeat_penalty` options
11. Multi-agent orchestrator (keyword/LLM routing) — inactive in Python
12. Prompt optimizer agent "Greg" (`chat.py`) — inactive in Python
13. Per-agent name/color identity — inactive in Python

Corrections vs note.md (verified 2026-10-07):
- Multi-language compile is NO LONGER missing: Rust `CompileTool` supports
  Rust, UefiRust, Java, Dotnet, Python.
- `SVEN_WORKDIR` is documented in the Python README but never implemented in
  Python code (config.py doesn't read it; security.py pins PROJECT_ROOT to
  cwd) — not a real gap.
- Python features not wired into its entry point are flagged [inactive]:
  orchestrator, Greg, env overrides, move/rename/append tools, git
  add/commit/push, model/server introspection helpers, old skills.py.

sven-rs features absent from Python: OpenAI/vLLM backends, MCP client, CLI
flags, non-interactive mode, strict path confinement (no y/N bypass).

The Python repo has `REWRITE_SPEC.md` at its root — a complete technical
specification of the Python implementation intended for the rewrite; consult
it for exact data models and algorithms when porting a feature.

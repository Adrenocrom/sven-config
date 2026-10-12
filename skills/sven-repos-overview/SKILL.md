---
name: sven_repos_overview
description: 'Relationship between Adrenocrom/sven (Python original) and Adrenocrom/sven-rs (Rust rewrite), plus key docs like REWRITE_SPEC.md and review.md round-6 findings (2026-10-12): round-5 M1 fix REGRESSED (content.is_empty() guard drops assistant tool-call echo → 400 on OpenAI/vLLM happy path), new M2 MAX_TOOL_OUTPUT typo 10_0000=100K, new M3 uncommitted max_tokens=Some(32000) default breaks vLLM/OpenAI, round-4 M1 serde aliases STILL OPEN 3rd round, num_predict exposed (good), README doc-sync done, debris 3rd round (output.txt, review.html, test file), carried long tail'
tags:
- sven
- sven-rs
- rust
- python
- agent
- code-review
- mcp
- regression
created_at: '2026-10-06T14:38:28.420045181+02:00'
---

# Sven repos: sven (Python) and sven-rs (Rust)

The user (Adrenocrom) maintains two related repos:

- **`Adrenocrom/sven`** — the original Python agent (Poetry, `ollama` client, prompt_toolkit). Autonomous: FIFO task queue, multi-agent orchestrator, conversation summarization, token tracking. Ollama-only. GPL-3.0. Contains `REWRITE_SPEC.md`, a full port spec.
- **`Adrenocrom/sven-rs`** — the Rust rewrite (reqwest + serde_json, hand-rolled MCP JSON-RPC 2.0). Leaner REPL loop, no task system/summarization. Backends: ollama, openai, vllm. MCP client (stdio + Streamable HTTP). MIT. This is the codebase the running agent is built from.

## Key files
- `sven/REWRITE_SPEC.md` — complete spec used to build sven-rs; documents the Python architecture (config, core loop, summarizer prompt, tools).
- `sven-rs/review.md` — **round-6 code review (2026-10-12), the current one** (supersedes round 5). Round-6 findings:
  - **M1 (new, top priority): the round-5 M1 "fix" REGRESSED.** The `if !message.content.is_empty()` guard before `history.assistant` in `run_rounds` skips the assistant echo for *every* tool round (tool-call rounds normally have empty content), so `role: "tool"` results dangle without their assistant message → OpenAI/vLLM reject with 400 on the happy path; Ollama silently loses the call record. Fix: guard on `!tool_calls.is_empty() || !content.is_empty()`, validate calls with `parse_tool_call` before history, synthesize missing ids, drop via `assistant_note`, add the interaction test.
  - **M2 (new):** `MAX_TOOL_OUTPUT` raised 10_000 → `10_0000` (100 000 chars, a typo) — guts the truncation feature; ~25–30K tokens from one tool result.
  - **M3 (new):** uncommitted config diff defaults `max_tokens: Some(32000)` (= full `num_ctx`) → vLLM/OpenAI reject requests out of the box. Fix before committing; default back to `None`. `num_predict` exposure (the other half of the diff) is good.
  - **L1 (new):** dead `MAX_TOOL_PARAMS_OUTPUT = 80` constant + new `unused import: PathBuf` warning (config.rs:3) — tree no longer warning-clean.
  - **L2 (new):** repo-root debris 3rd round running — `output.txt`, `review.html` still present, plus new file `test` (captured cargo build output). `.gitignore` still only `/target`.
  - **Round-4 M1 STILL OPEN (3rd round):** `Backend` doc comments promise serde aliases that don't exist; no `#[serde(alias)]`; missing serde test for `"backend": "Ollama"`.
  - **Carried:** M3 grep exit-1 (reproduced 4th round), M5 unbounded history (aggravated by M2), M6 no chat-client/subprocess timeouts, r4 L1 double newline, r4 L5 Ollama mid-stream error objects dropped, r5 L1 notifications unconditional (README half fixed this round), r5 L2 two SSE parsers, r5 L3 history fragility, L1 `n:0`, L2 CompileTool case-sensitive + no Test variant (reproduced: `"rust"` fails), L3 no `col -b`, L4 params JSON dump, L5 FindTool `./` anchoring, L6 stdout mixing, L7 skills symlink writes, L10 `max_tokens` no-op on Ollama (num_predict half now done), L11 no LC_ALL=C, L12 grab-bag, MCP L1–L6, async nits.
  - Test suite: 68 functions across 13 files (round 5's count of 60 was low). Still never executed (no Test variant, no CI).
  - §6 carries the 28-item feature list (context management still #1; token tracking done r5; num_predict done r6).
- `sven-rs/note2.md` — corrected Python-vs-Rust feature comparison (supersedes note.md): missing = task queue, summarization, token persistence, cross-session history, touch, replaceline, per-file compile, profiles, keep_alive/repeat_penalty. Git diff tool is NO LONGER missing.

## Implemented since round 5 (2026-10-12)
- **`num_predict` exposed** in `ChatOptions` (Ollama output cap) — uncommitted; flows through the `options` envelope. `ChatOptions::default` now derives all caps from one `num_ctx` variable (but `max_tokens = num_ctx` is the M3 footgun).
- **README doc sync** (done by the agent during round 6): all 19 tools in the table (GitDiffTool + NotifySendTool added), notification behavior documented, `options` semantics per backend, requirements list completed (git, notify-send, build tools), stats.rs in architecture list.
- **`if !message.content.is_empty()` guard** before `history.assistant` — the round-5 M1 fix attempt that regressed (see M1).
- `MAX_TOOL_OUTPUT` 10_000 → 10_0000 (the M2 typo).

## Conventions in sven-rs
- Config: `~/.config/sven/sven.json`; API key only via `SVEN_API_KEY` env (never in config).
- Security model: path confinement via `security::is_inside_cwd` (symlink-resolved), no shell (explicit argv), argument-injection guards (`./` prefix, `--` markers, leading-dash + slash rejection in man), http/https-only URLs.
- Skills: markdown + YAML frontmatter under `<data_dir>/skills/<name>/SKILL.md`, snake_case names, lowercase-hyphenated tags.
- All tools async (boxed Send futures via `tool!` macro); MCP stack fully async (tokio mpsc stdio, async reqwest HTTP, tokio Mutex).
- Data files under `data_dir` (default `~/.config/sven`): `skills/`, `history`, `statistics.json`.
- review.md convention: no trailing newline at EOF (matches original); README.md has one.

---
name: sven_vllm_backend
description: 'vLLM backend in sven-rs: Vllm variant shares all OpenAI wire code (endpoint, SSE, fragmented tool calls, bearer auth); vLLM needs --enable-auto-tool-choice + --tool-call-parser for tools (else 400), --reasoning-parser for reasoning_content; tool calls echoed back in assistant messages MUST carry type:"function" (litellm 400s without it — fixed in fragment merge + chat_history defaulting); extra fields allowed; Backend enum serde lowercase fix'
tags:
- sven
- vllm
- litellm
- openai-compatible
- backend
- tool-calling
- type-function
- serde
created_at: '2026-09-30T10:46:36.772761946+02:00'
---

# vLLM backend in sven-rs

Context: sven-rs (`src/sven/backend.rs`, `agent.rs`, `config.rs`) gained a `Vllm` backend variant
(2026-09-30). vLLM serves an OpenAI-compatible API, so `Vllm` shares ALL wire code with
`Backend::OpenAI` — endpoint `/v1/chat/completions`, SSE parsing, fragmented tool-call merging,
`tool_call_id` matching, bearer auth. The variant exists only so configs are self-documenting
and vLLM-specific behavior has a place to diverge later.

## Key facts about vLLM's OpenAI-compatible server

- Default port **8000** (Ollama is 11434). `host` points at the server root; the code appends
  `/v1/chat/completions`.
- **Tool calling requires server flags**: `--enable-auto-tool-choice --tool-call-parser <parser>`
  (parser must match the model: `hermes` for Qwen, `llama3_json` for Llama 3.1, `mistral` for
  Mistral). Without them vLLM answers **400** to any request carrying `tools` — this is why
  the HTTP status check in `agent.rs::run()` matters; before it, that 400 was silently swallowed
  (review.md round-3 M1) and the agent "did nothing".
- **Reasoning**: `--reasoning-parser deepseek_r1` makes vLLM emit thinking as
  `delta.reasoning_content` (DeepSeek lineage) — already handled by `process_json_openai`
  alongside `delta.reasoning` (OpenRouter lineage).
- **Auth**: vLLM only checks `Authorization: Bearer` when started with `--api-key`; sending it
  unconditionally is harmless. sven sends it whenever `SVEN_API_KEY` is set and the backend is
  OpenAI-compatible.
- **Extra fields are allowed**: vLLM's `OpenAIBaseModel` uses pydantic `extra="allow"` ("OpenAI
  API does allow extra fields"), so sven's non-standard `tool_name` field on tool messages is
  tolerated (it logs a warning server-side). Verified in vllm/entrypoints/openai/protocol.py.
- `tool_choice` defaults to `"auto"` in vLLM's `ChatCompletionRequest` — sven never sends it,
  which is correct.

## Tool calls echoed back MUST carry `type: "function"` (fixed 2026-09-30)

**Symptom** (captured in `vllm-error.txt`): 400 Bad Request from **litellm** fronting vLLM —
```
litellm.BadRequestError: OpenAIException - 3 validation errors:
  {'type': 'missing', 'loc': ('body', 0, 'ChatCompletionMessageFunctionToolCallParam', 'type'),
   'msg': 'Field required', 'input': {'function': {...}, 'id': 'chatcmpl-tool-…'}}
```
**Cause**: the *second* request of a tool round fails, not the first. The assistant message with
`tool_calls` is echoed back in the request body, and litellm (unlike raw vLLM, which allows extra
fields but still wants `type`) strictly validates each tool call against
`ChatCompletionMessageFunctionToolCallParam` — `type` is required. sven's fragment merge in
`backend.rs` only carried `id`/`name`/`arguments` and dropped `type` (vLLM sends it only on the
first streamed fragment); Ollama-shaped calls never had it at all.

**Fix (two layers, both needed)**:
1. `backend.rs` fragment merge: copy `tc["type"]` into the merged entry when present.
2. `chat_history.rs::assistant()`: default `tool_call["type"] = "function"` for any call
   missing it — this is the boundary where history becomes the next request body, so it covers
   Ollama-shaped calls too. Ollama ignores the extra field.

Tests: `openai_tool_call_fragments_are_merged_by_index` and `vllm_stream_parses_like_openai`
assert `tc["type"] == "function"`; `tool_calls_sent_back_always_carry_type_function` covers the
Ollama-shaped defaulting.

## Backend enum serde gotcha (fixed 2026-09-30, review round-3 M2)

`Backend` derives `Deserialize` with `#[serde(rename_all = "lowercase")]` so config files use
`"ollama"`/`"openai"`/`"vllm"` — the same strings `ToString` emits. The capitalized forms
(`"Ollama"`, `"OpenAI"`) are kept via `#[serde(alias = ...)]` for old configs. Before the fix the
un-annotated enum expected capitalized names, so `"backend": "openai"` failed to parse and
`SvenConfig::load()` silently fell back to the *whole* default config (Ollama + default model +
host). Unknown names are now a hard parse error, not a silent fallback. The dead
`From<String> for Backend` impl was removed.

## Testing note

`cargo build` (what CompileTool runs) does NOT compile `#[cfg(test)]` code — backend tests were
verified by reading, not by running. Run `cargo test` before assuming test changes are safe.
Also: GrepTool here uses BRE, so `a|b` matches the literal string "a|b" — use separate calls or
`-e` patterns instead.

---
name: sven_openai_backend_streaming
description: 'OpenAI-compatible backend streaming gotchas in sven-rs: max_tokens counts reasoning tokens (finish_reason "length" bail-out), chunked tool-call fragments, mid-stream error objects, SSE parsing, reasoning vs reasoning_content field names'
tags:
- sven
- openai
- streaming
- sse
- reasoning
- reasoning-content
- max-tokens
- finish-reason
created_at: '2026-09-30T09:27:22.361605622+02:00'
---

# OpenAI-compatible streaming gotchas in sven-rs

Context: sven-rs (`src/sven/backend.rs`, `agent.rs`, `config.rs`) supports both Ollama and
OpenAI-compatible backends via a `Backend` enum. Reasoning models on OpenAI-compatible servers
(vLLM, OpenRouter, DeepSeek, LM Studio) exposed a cluster of silent-failure bugs.

## The "bail out mid-reasoning" bug (2026-09-30)

Symptom: with a reasoning model, the agent returned to the prompt in the middle of the thinking
step — no answer, no tool call, no error.

Root cause chain:
1. `ChatOptions::default()` sets `max_tokens: Some(1024)`, sent on every OpenAI request.
   Reasoning models count thinking tokens against `max_tokens`, so the stream ends with
   `finish_reason: "length"` before any content is produced.
2. `finish_reason` was never inspected. The old code
   `if json["choices"][0].get("finish_reason") == None && stream_state.is_answering` was doubly
   broken: inverted vs. the Ollama `done` logic it mirrored, and OpenAI sends explicit
   `"finish_reason": null` on regular chunks, so `get(...) == None` is never true.
3. `run()` treats empty content + no tool calls as a finished turn and silently returns.

Fix: `StreamState.finish_reason: Option<String>` captured from the final chunk, carried through
`MessageResponse`, and `run()` prints a red warning on `"length"`.

## Token usage in streams (2026-10-09)

OpenAI-compatible streams do NOT report token counts by default. The request must send
`"stream_options": {"include_usage": true}` (added to `request_body` for OpenAI/Vllm); the server
then appends one final chunk after the `finish_reason` chunk, before `data: [DONE]`:

```json
{"choices":[{"index":0,"delta":{}}],"usage":{"completion_tokens":77,"prompt_tokens":2289,
 "total_tokens":2366,"completion_tokens_details":{"reasoning_tokens":20}}}
```

Facts:
- Every other chunk has `"usage": null`; only check `json["usage"]["completion_tokens"]` /
  `prompt_tokens` as u64 — a null/absent usage silently skips the line, so servers that ignore
  `stream_options` (older vLLM, some gateways) just print nothing instead of erroring.
- The usage chunk carries an empty `delta` and NO `finish_reason`, so it doesn't interfere with
  content/finish_reason handling.
- Display mirrors Ollama's `eval_count`/`prompt_eval_count` line: bold `in <prompt> out <completion>`.
- `completion_tokens_details.reasoning_tokens` (vLLM/DeepSeek) tells how many of the completion
  tokens were thinking — useful when diagnosing `finish_reason: "length"` on reasoning models.
- Ollama needs no opt-in: its final NDJSON line carries `eval_count`/`prompt_eval_count` directly.

### Accumulating tokens + timing a run (2026-10-09)

Both backends report the totals for the WHOLE response exactly once (Ollama on the done chunk,
OpenAI in the single usage chunk) — so per-chunk handling must ASSIGN into
`StreamState.usage` (`TokenUsage { prompt_tokens, completion_tokens }` in `chat_history.rs`),
never add, or a repeated chunk would double-count. `handle_chunks` copies it into
`MessageResponse.usage`; `Agent::run` wraps the round loop (`run_rounds`) to sum usage across
all tool rounds and print a bold summary with wall time on every exit path (final answer,
request error, round cap):

```
run finished in 1m 3s — in 15234 out 812 tokens (16046 total)
```

`format_duration` (agent.rs) rounds seconds BEFORE splitting into m/s — formatting the seconds
with `{:.0}` after flooring minutes prints "59m 60s" for 59m59.9s. Servers that report no
counts leave usage at 0; the summary still prints (with zeros).

## Reasoning field names: `reasoning` vs `reasoning_content`

There is NO standard: the official OpenAI Chat Completions API never streams reasoning at all,
so compatible servers invented two rival conventions:

- `delta.reasoning_content` — the **DeepSeek lineage**: DeepSeek-R1 named it that (mirroring
  `delta.content`), and vLLM's `--reasoning-parser` (built for R1) copied the name; SGLang,
  LM Studio and most self-hosted stacks follow vLLM.
- `delta.reasoning` — the **OpenRouter/Cerebras lineage**; OpenAI's Responses API also uses
  `reasoning`. vLLM RFC #27755 proposes renaming `reasoning_content` → `reasoning` to align,
  but both names coexist in the wild.

`process_json_openai` therefore tries `delta.reasoning` first, then falls back to
`delta.reasoning_content` (review.md L8). Practical notes:
- The field only appears if the server runs a reasoning parser (e.g. vLLM
  `--reasoning-parser deepseek_r1`); otherwise thinking is stripped or dumped raw into
  `content` inside `<think>` tags.
- Round-tripping: DeepSeek-style servers expect reasoning sent back under the SAME key
  (`reasoning_content`); mismatched names cause 400s in some gateways.

## Other OpenAI streaming facts

- Tool calls arrive as fragments keyed by `index`; only the first carries `id` and `name`.
  Accumulate `function.arguments` string fragments in `StreamState.tool_call_fragments`
  (BTreeMap<u64, Value>) and merge in `Backend::finalize` after the stream ends.
- Some servers report failures mid-stream as a top-level `{"error": {"message": ...}}` object
  instead of closing the connection — check `json.get("error")` before touching `choices`.
- SSE lines: skip `data: [DONE]`, strip `data: ` prefix, skip comments (`: keep-alive`).
- `tool_name` on tool messages is ignored by OpenAI; matching is via `tool_call_id` only.
- `num_ctx` is Ollama-only; OpenAI has no context-window knob, `max_tokens` caps output only.
- `cargo build` does not compile `#[cfg(test)]` code — run tests before assuming a struct change
  is safe (MessageResponse gained a field and five test constructors needed updating).

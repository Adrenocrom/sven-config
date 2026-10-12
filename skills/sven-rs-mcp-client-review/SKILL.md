---
name: sven_rs_mcp_client_review
description: 'Round-4 review (2026-10-06) of the sven-rs MCP client (src/sven/mcp/): verdict mostly spec-correct on the 2025-06-18 MUSTs; M1 (HTTP 404 session-expiry re-initialize) and M2 (curl Expect:100-continue breaking >1MB tools/call bodies) were FIXED the same day — SessionExpired error + retry-once in request(), -H "Expect:" + 1xx-skipping split_headers. Open findings — stdio Drop skips SIGTERM, no notifications/cancelled, negotiated protocolVersion echoed unvalidated, SSE stream kept open costs full --max-time, no HTTP DELETE on exit, inputSchema guard checks is_object not type=="object".'
tags:
- sven
- rust
- mcp
- code-review
- json-rpc
- curl
- streamable-http
- stdio
created_at: '2026-10-06T08:22:07.076460423+02:00'
---

# sven-rs MCP client — round-4 review (2026-10-06), M1/M2 fixed same day

Scope: `src/sven/mcp/{mod.rs,transport.rs}`, `src/sven/tools/mcp_tool.rs`, config wiring.
Method: read all code; verified against the official 2025-06-18 spec (lifecycle, transports,
tools, ping — fetched from modelcontextprotocol.io, .md endpoints); `cargo build` passes.
Tests compile-verified only (CompileTool runs cargo build, not cargo test). Findings marked
[code analysis] are reasoned, not reproduced (no way to run a live MCP server from the tools).

## Verdict
Mostly correct and spec-compliant. All client-side MUSTs of Streamable HTTP are implemented
(MCP-Protocol-Version echo, Mcp-Session-Id capture/echo, Accept dual content-type, POST per
message, 202-empty-body). The JSON-RPC core (classify, id matching, stale-response dropping,
server-request refusal with -32601, pagination repeat-cursor guard, isError→Err,
structuredContent fallback, binary-content placeholders) is solid and well tested
via MockTransport.

## FIXED (2026-10-06, same session)
- **M1 — HTTP 404 session expiry**: `HttpTransport::send` now raises a typed `SessionExpired`
  error on 404-with-session-id (clears `session_id` and `pending` first); `McpClient::request`
  catches it via `downcast_ref`, re-runs `initialize()` and retries the request once.
  `initialize` uses `request_once` directly (no recursion); the refusal-send path in
  `request_once` also propagates `SessionExpired` unwrapped. Tool registry intentionally NOT
  re-listed after re-init (frozen at startup — documented). MockTransport gained
  `session_expired_once` flag; test `a_terminated_session_is_reinitialized_and_the_call_retried`
  covers the full sequence (call → 404 → initialize → retry).
- **M2 — Expect: 100-continue**: curl argv now carries `-H "Expect:"` (curl's documented
  suppression — removes the header for >1 MB bodies, also kills the 1 s stall when a server
  ignores Expect). Defense-in-depth: `split_headers` skips interim 1xx blocks (loop: while
  parsed status is 1xx, advance past the next blank line); test `skips_interim_100_continue_responses`
  covers CRLF and LF-only dumps.

## Remaining findings (open)

### L1 (Low) — stdio Drop skips SIGTERM
Spec shutdown sequence: close stdin → wait → SIGTERM → wait → SIGKILL. Code: close stdin →
1 s wait → kill() (SIGKILL). Servers that flush/cleanup on SIGTERM lose that. std has no signal
API; pragmatic fix: spawn `kill -TERM <pid>` then wait, or document the deviation.

### L2 (Low) — no `notifications/cancelled` on timeout
Lifecycle: on timeout the sender SHOULD send a cancellation notification. sven stops waiting
but never tells the server → the server keeps computing the abandoned call. Stale replies are
harmlessly dropped by the id check. Fix: best-effort `notify("notifications/cancelled",
{"requestId": id})` in request_once()'s timeout branch.

### L3 (Low) — negotiated protocolVersion echoed into a header unvalidated
`initialize()` passes the server's protocolVersion string to `negotiated()` → becomes the
`MCP-Protocol-Version` header via format!. Control chars in the server's string could smuggle
header bytes into curl's request (low impact — same server already trusted — but the config
header validation discipline should apply). Fix: accept only printable ASCII [!-~] (the charset
the spec mandates for session ids), else fall back to PROTOCOL_VERSION.

### L4 (Low) — SSE stream kept open costs the full --max-time
`send()` blocks in wait_with_output until curl exits; a server that doesn't close the SSE
stream after the response (spec: SHOULD close) makes every call take the full 60 s even though
the result arrived earlier. Latency, not failure (exit 28 + non-empty queue → Ok). Inherent to
the one-POST-per-message design; document or drain incrementally (bigger rework).

### L5 (Low) — no HTTP DELETE on exit (spec SHOULD) → sessions leak server-side until expiry.

### L6 (Low) — inputSchema guard checks the wrong property
`tool_info` (mod.rs): `schema.is_object()` verifies a JSON map, NOT `"type": "object"`.
A spec-violating server reporting `{"type":"array"}` passes through into the chat request's
`parameters`; OpenAI-compatible backends validate parameters and can reject the WHOLE request
(one bad tool breaks every chat round). Fix: pass through only when type == "object" or absent,
else substitute the empty-object schema.

### Informational (conscious deviations, mostly SHOULD-level)
- No client ping (SHOULD); dead stdio servers surface as errors on the next call.
- `notifications/tools/list_changed` ignored — tool set frozen at startup (also after re-init).
- Progress notifications dropped → the 60 s tools/call timeout is never extended (spec MAY).
- PROTOCOL_VERSION pinned to 2025-06-18 (2025-11-25 / 2026-07-28 exist) — defensible: sven
  implements that feature set; negotiation + stderr warning handles newer servers; tools/*
  parsing is lenient (unknown content types → placeholders).
- Continuing after a version mismatch deviates from "SHOULD disconnect" — justified in a code
  comment (tools/list + tools/call identical across revisions for sven's lenient parsing).
- Unused config fields silently ignored per transport (args/env with url, headers with command).
- discover() blocks startup serially, up to 120 s per server (documented in README).

## Verified correct (client MUSTs + good engineering)
initialize/initialized handshake; MCP-Protocol-Version echo of the NEGOTIATED version;
Mcp-Session-Id capture + echo on every later request; Accept: application/json,
text/event-stream; one POST per message; 202-with-empty-body handled; JSON and SSE responses
both parsed (multi-line data joined per SSE spec, one optional space stripped, comments/id/retry
lines ignored); pagination with repeat-cursor guard; JSON-RPC classification (id+method /
id / neither) and id matching with stale-response dropping; server→client requests refused with
-32601 (prevents waiting servers from hanging) while notifications are correctly NOT answered;
isError → Err surfaced to the model via process_tool_call; structuredContent fallback; binary
content → placeholders (no base64 flood); timeouts on every exchange (120/60/30 s); curl argv
hardening: `--` before the URL, body via stdin (no argv limit, no option injection), header
name/value validation (':' + control chars), http/https scheme allowlist, no -L (configured
headers can't leak cross-origin on redirect), `Expect:` suppressed; stdio reader-thread → mpsc
channel so recv can time out; child killed on Drop; non-JSON stdout noise tolerated; unreachable
servers skipped at startup (never fatal); `mcp__<server>__<tool>` naming (collision-safe);
inputSchema passthrough; Arc<Mutex<McpClient>> shared per server (agent runs calls sequentially;
poisoned lock recovered). Tests: MockTransport covers handshake sequence, id increments, refusal,
JSON-RPC error mapping, closed connection, classify, tool_info leniency, result formatting,
SSE + header parsing, 1xx skipping, session-expiry re-init.

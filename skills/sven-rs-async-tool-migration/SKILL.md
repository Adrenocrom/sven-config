---
name: sven_rs_async_tool_migration
description: 'COMPLETED 2026-10-06 and reviewed/approved same day: sven-rs Tool::execute is now async (boxed Send futures via tool! macro), MCP stack fully async (tokio mpsc stdio, async reqwest HTTP, tokio Mutex); documents the !Send Box<dyn Error> match-scrutinee trap, tokio ChildStdout no-into_stdio workaround, as_std().get_program(), the Box::pin async-block error-type inference trap, and the review''s minor nits (missing EOF newlines, from_std runtime panic, blocking Drop)'
tags:
- sven
- rust
- async
- tokio
- reqwest
- mcp
- migration-completed
created_at: '2026-10-06T11:45:02.936287166+02:00'
---

# Making all tools async in sven-rs — migration COMPLETED (2026-10-06)

Status: **DONE, compiling, and REVIEWED (approved).** `Tool::execute` is async (boxed `Send`
futures via the `tool!` macro), the whole MCP stack is async, and the HTTP transport uses
plain async `reqwest` (no `blocking` feature, no `block_in_place`, no curl subprocess).

## Final shape (as built)
- tool.rs: `ToolFuture<'a> = Pin<Box<dyn Future<Output = Result<String,
  Box<dyn Error>>> + Send + 'a>>`; trait `Tool: Send + Sync`.
- macros.rs: both `tool!` arms `Box::pin(async move { … })`.
- tool_registry.rs: unchanged `Box<dyn Tool>` (trait itself is Send+Sync).
- agent.rs: `process_tool_call` async, `.await` on `tool.execute()`.
- tools/*: bodies unchanged; subprocess.rs uses `tokio::process::Command`
  (note: tokio's Command has NO `get_program()` — use `command.as_std().get_program()`).
- mcp/transport.rs: `Transport` trait methods return boxed `Send` futures
  (`TransportFuture<'a, T>`); StdioTransport spawns via **std::process**
  (stdout must stay a std handle for the reader thread) + `ChildStdin::from_std`
  for async writes + `tokio::sync::mpsc::unbounded_channel`; recv =
  `tokio::time::timeout(remaining, rx.recv())` against one deadline.
  HttpTransport: async `reqwest::Client` (built once, `redirect(Policy::none())`),
  `.timeout(timeout)` on the request + manual deadline around a
  `response.chunk().await` loop (`read_bounded`) preserving curl's
  partial-data-on-timeout semantics; 404+session → `SessionExpired`.
- mcp/mod.rs: all McpClient methods async; `Arc<tokio::sync::Mutex<McpClient>>`.
- tools/mcp_tool.rs: async execute, `lock().await`.
- main.rs: `build_agent` async, awaited in `#[tokio::main]` main.
- Tests: `#[tokio::test]` (current_thread flavor fine — no block_in_place).
- web_fetch.rs: curl via **std::process** (its stdout must wire into
  pandoc's stdin via `Stdio::from`; tokio's ChildStdout cannot convert
  back), pandoc via tokio, `wait_with_output().await` on pandoc, then a
  provably-instant blocking `curl.wait_with_output()` (pandoc EOF ⇒ curl
  already exited).

## Gotchas hit during the port (each cost a compile round)
1. **`!Send` future from `Box<dyn Error>` across awaits** — the retry
   wrapper in `McpClient::request` held the error (match scrutinee /
   binding) alive across `initialize().await`; `Box<dyn Error>` is !Send
   → the whole tool future became !Send. `drop(e)` inside the arm does
   NOT help (match temporaries live to the end of the match). Fix: match
   arm that only *prints* the error and falls through; the retry awaits
   happen AFTER the match, with no error value in scope.
2. **tokio `ChildStdout` has no `into_stdio`/`try_into_stdio`** — when a
   std handle is needed (reader thread, piping to another child), spawn
   with `std::process::Command` and convert only what must be async via
   `ChildStdin::from_std` (needs runtime context).
3. **tokio `Mutex` has no poisoning** — delete the
   `unwrap_or_else(|poisoned| poisoned.into_inner())` recovery pattern.
4. **tokio `Child` has `start_kill()`, std `Child` only `kill()`** — the
   stdio transport's Drop uses std's sync `kill()` + `try_wait()` loop.
5. **`Box::pin(async move { … })` error-type inference trap (E0271)** —
   inside an async block, `return Err(Box::new(SessionExpired))` infers
   the block's error type as `Box<SessionExpired>`; the expected
   `Box<dyn Error>` only exists at the outer `Box::pin` coercion site and
   does NOT propagate into the block body. Fix: annotate at the return
   site — `let err: Box<dyn Error> = Box::new(SessionExpired); return
   Err(err);`. (In a fn with an explicit `Result<_, Box<dyn Error>>`
   signature the coercion works fine — this only bites in unannotated
   async blocks / closures.) Hit in the test mock `MockTransport::send`
   (transport.rs); the same expression in `HttpTransport::post` was fine
   because that fn has the explicit signature.

## Deleted vs the old blocking design
reqwest `blocking` feature, `block_in_place`, `read_bounded` io::Read
loop, `is_timeout` unwrap dance, "must keep multi-thread runtime"
constraint, `#[tokio::test]` restriction, curl subprocess in the MCP HTTP
transport, `split_headers`/`status_code`/`header_value` (reqwest parses
headers itself; `Expect:` suppression became unnecessary — reqwest never
sends `Expect: 100-continue`).

## Review round (2026-10-06, full-diff review) — APPROVED
cargo build passes; semantics verified as faithfully preserved: curl
`--max-time`/exit-28 → `.timeout()` + `read_bounded` deadline with
partial-body-kept-on-timeout and `timed_out && pending unchanged → Err`;
404+session → SessionExpired retry-once intact (retry awaits after the
match, no !Send leak); `redirect(Policy::none())` kept (auth headers
can't leak cross-origin); 1xx/Expect code + its tests correctly deleted.
Cargo.lock additions (signal-hook-registry, errno) are tokio's unix
SIGCHLD plumbing pulled in by the `process` feature — expected.
Minor nits (open, cosmetic/robustness):
- 8 files lost their trailing newline: macros.rs, tool.rs, mcp/mod.rs,
  find_tool.rs, grep_tool.rs, list_files.rs, manpage_tool.rs, web_search.rs.
- `ChildStdin::from_std` PANICS outside a runtime; `StdioTransport::spawn`
  is a sync fn — only the async `connect` calls it today, but a sync
  caller would panic instead of erroring.
- `Drop for StdioTransport` can block its thread ~2 s (two 20×50 ms sleep
  loops) — fine at process exit (the only drop site today); would stall a
  worker if a client were dropped mid-session.
- web_fetch's "provably instant" `curl.wait_with_output()` claim is
  overstated for the pandoc-dies-early edge case: the block then lasts
  until curl's next write hits EPIPE — bounded, not instant.
- `discover`'s new doc paragraph duplicates the tokio-Mutex rationale
  already on `McpTool::client` and describes `call_tool` internals.
- subprocess.rs module doc lists `pandoc`, but pandoc is spawned directly
  in web_fetch.rs, not via `subprocess::run`.
- reqwest sends no `User-Agent` (curl sent `curl/x.y`) — only matters for
  UA-filtering servers.
- Tests are `#[tokio::test]` but were compile-verified only (CompileTool
  runs cargo build); run `cargo test` once to execute them.

## Still open (pre-existing review findings, unrelated to async)
M1 grep exit-1 as error, M2 grep BRE, M3 unbounded history, M4 no
timeouts on man/ddgr/cargo, L1 SearchAndReplace n:0, L2 CompileTool
case-sensitivity; MCP round-4 L1–L6 (SIGTERM, notifications/cancelled,
protocolVersion header validation, SSE drain latency, no DELETE,
inputSchema type check).

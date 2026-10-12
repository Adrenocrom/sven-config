---
name: reqwest_blocking_in_tokio
description: 'How to use reqwest::blocking inside a tokio runtime: the documented panic, the block_in_place escape hatch (source-verified mechanism), timeout semantics differences vs curl, and redirect Policy::none for no-follow behavior'
tags:
- rust
- reqwest
- tokio
- blocking
- block-in-place
- http-client
- timeouts
created_at: '2026-10-06T11:37:30.493157708+02:00'
---

# reqwest::blocking inside a tokio runtime — mechanics and escape hatch

## The rule
`reqwest::blocking` (Client::new, ClientBuilder::build, send(), body reads) **panics
when called from within an async runtime**. Documented in reqwest's module docs:
"If the immediate context is only synchronous, but a transitive caller is async,
consider changing that caller to use tokio::task::spawn_blocking around the calls
that need to block."

## Why it panics (source-verified, reqwest 0.13.5)
`src/blocking/wait.rs::enter()` — under `#[cfg(debug_assertions)]` — builds a
shell `current_thread` runtime and calls `.enter()`. Tokio's
`context::enter_runtime` panics with "Cannot start a runtime from within a
runtime" iff the thread-local `EnterRuntime` state is `Entered`. In release
builds the check is absent but blocking still deadlocks/starves the worker.

## The escape hatch: `tokio::task::block_in_place`
- Flips the thread-local `EnterRuntime` to `NotEntered` for the closure's
  duration (`context::exit_runtime`), so reqwest's shell-runtime check passes.
- Requires `rt-multi-thread` (panics on current_thread runtimes, incl. default
  `#[tokio::test]`).
- Works from the **main thread** inside `#[tokio::main]`'s `block_on` — the
  multi-thread scheduler hands the worker core to another thread.
- **Passthrough outside any runtime** — plain `#[test]` code calling through it
  works unchanged. This makes it strictly better than
  `Handle::current().block_on(...)` for sync-in-async bridges, because
  `Handle::current()` panics outside a runtime.
- `spawn_blocking` is the alternative but needs `Send + 'static` closures and
  an `.await` to collect the result — awkward from sync call sites.

## Timeout semantics gotcha (curl → reqwest migration)
- curl `--max-time` + exit 28: partial output is still delivered, so a server
  that answered but kept its SSE stream open still yields its messages.
- reqwest total timeout (`ClientBuilder::timeout` / `RequestBuilder::timeout`):
  covers connect→body-end, but on expiry the body is **discarded** — the
  `Response` is gone.
- To preserve partial-data-on-timeout: read the blocking `Response` (it
  implements `io::Read`) in a loop against a manual `Instant` deadline; treat a
  timeout-kind `io::Error` as end-of-stream and keep the bytes collected so far.
- Timeout detection: unwrap the inner `reqwest::Error` from the `io::Error`
  (`e.get_ref()` downcast) and call `is_timeout()` (walks the source chain).
  The concrete `io::ErrorKind` is NOT public contract — don't match on it alone.

## Redirects
reqwest follows redirects by default (up to 10) — curl without `-L` does not.
To preserve no-follow (e.g. so configured Authorization headers can't leak
cross-origin): `Client::builder().redirect(reqwest::redirect::Policy::none())`.
The blocking ClientBuilder does expose `.redirect()` (delegates to the async
builder via `with_inner`).

## Other notes
- Build the blocking `Client` once and reuse: it pools connections and keeps a
  background runtime thread alive for its lifetime.
- `ClientBuilder::build()` returns `Result` (TLS init can fail) — unlike
  `Client::new()` which panics.
- reqwest 0.13: rustls is the default TLS backend (aws-lc provider),
  `rustls-platform-verifier` for roots; custom roots via `tls_certs_only(...)`.
- hyper never sends `Expect: 100-continue`, so curl's `-H "Expect:"` workaround
  is unnecessary with reqwest.

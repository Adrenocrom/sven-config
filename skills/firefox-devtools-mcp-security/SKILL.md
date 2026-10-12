---
name: firefox_devtools_mcp_security
description: 'Security posture and safe-usage checklist for Mozilla''s firefox-devtools-mcp MCP server (browser automation via WebDriver BiDi)'
tags:
- firefox
- mcp
- security
- browser-automation
- prompt-injection
- mozilla
created_at: '2026-10-07T13:23:23.133225808+02:00'
---

# firefox-devtools-mcp security posture

Official Mozilla MCP server (github.com/mozilla/firefox-devtools-mcp, npm `@mozilla/firefox-devtools-mcp`) automating Firefox via WebDriver BiDi. Dual MIT/Apache-2.0. This is the same server Sven runs with (mcp__firefox-devtools__* tools).

## Source review (2026-10-07, main @ 5c69888) — no malicious code found
Reviewed: package.json, index.ts/index.public.ts/index.moz.ts, cli.ts, tools/{registry,index,downloads,webextension,firefox-prefs,privileged-context,firefox-launch,script,utilities,network,screencast,firefox-management}.ts, firefox/{core,profile,bidi,cache,dom,index,pref-utils,windows-binary}.ts, utils/{save-output,logger,paths,js-validation,response-helpers}.ts.
- **No install hooks** (no pre/postinstall/prepare); deps pinned & reputable (MCP SDK, geckodriver, selenium-webdriver, ws, yargs).
- **No network egress**: only local sockets — geckodriver on 127.0.0.1 + its BiDi WebSocket. No fetch/http imports in src.
- **No obfuscation, no telemetry, no credential access.**
- Only child-process use: `execFileSync` of absolute-path `reg.exe` (read-only App Paths lookup, Windows binary discovery; deliberately PATH-hijack-safe).
- File writes guarded by `assertAllowedPath` (save-output.ts): resolves symlinks (`realpathThroughMissing`), restricts to cwd/~/.firefox-devtools-mcp/output, **refuses writes into any Firefox profile** (anti-exfiltration), atomic temp+rename. Opt-out flag `--unrestricted-save-paths`.
- Profile handling is defensive: refuses to use a real profile directly, creates dedicated `firefox_devtools_mcp_profile` subfolder, warns user.
- Privileged tools fail closed: public build strips `--enable-privileged-context` and passes `allowPrivileged=false`; `prefs`/`privileged`/`launch` modules excluded from public npm package; chrome-context tools additionally require Firefox-side `MOZ_REMOTE_ALLOW_SYSTEM_ACCESS=1`.
- Pref injection safe: `generatePrefScript` uses `JSON.stringify` for both name and value.
- `upload_file_by_uid` types a local path into a file input (agent-reachable local→web file upload; inherent capability, not malice).

## Trust
- mozilla org, CI + codecov, honest SECURITY.md, Bugzilla security channel.

## Inherent risks (browser MCP = prompt injection surface)
- `evaluate_script` is in the DEFAULT `basic` preset → arbitrary JS in any page. Drop with `--tool-preset slim` or explicit `--tools`.
- `--connect-existing`: attaches to a running Firefox — if that's the daily profile, agent gets cookies/passwords/sessions.
- `--unrestricted-save-paths`: removes saveTo write restrictions → arbitrary file writes.
- `--accept-insecure-certs`: disables TLS validation.
- MCP servers run with full user privileges; Claude's sandbox does NOT cover them.

## Safe usage checklist
1. Dedicated Firefox profile (never the daily one).
2. `--tool-preset slim` unless script execution is needed.
3. Avoid risky flags; only visit trusted sites (page text/DOM can steer the agent).
4. Untrusted content → container/VM with restricted egress.
5. `saveTo` defaults are safe; keep `--unrestricted-save-paths` off.

## Presets (supersets)
slim (pages, snapshot, input, screenshot) < basic (+downloads, script, utilities, management, webextension, screencast) < developer (+debugging, network, console, profiler) < mozilla (+launch, prefs, privileged) < all.

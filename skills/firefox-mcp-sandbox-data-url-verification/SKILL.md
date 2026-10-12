---
name: firefox_mcp_sandbox_data_url_verification
description: 'Firefox MCP browser runs in a sandbox that cannot read the agent''s local files via file://; verify SVG/HTML rendering by inlining content as data: URLs'
tags:
- firefox
- mcp
- sandbox
- data-url
- svg
- verification
- screenshot
created_at: '2026-10-09T14:53:34.419156857+02:00'
---

# Verifying local files in the Firefox MCP browser

The Firefox MCP instance runs in a **separate sandbox** from the agent's file
tools — `file://` URLs to files written via ReplaceFileTool fail with
"Problem loading page" even though the file exists (confirmed via
FindTool/ReadTool).

## Workaround: inline content as a data: URL

Navigate to a `data:text/html,...` page that embeds the file content, e.g. an
`<img src="data:image/svg+xml;base64,...">` for SVGs. This bypasses the
filesystem entirely and renders exactly what the file contains.

Steps:
1. Base64-encode the file content (do this mentally/inline when composing the
   data URL).
2. `navigate_page` to `data:text/html,<html><body>...<img src="data:image/svg+xml;base64,...">...</body></html>`
   with `wait: "complete"`.
3. `screenshot_page` to visually verify; `evaluate_script` to check computed
   geometry (getBoundingClientRect, element counts) for structural checks.

## Useful checks for SVG logos

- Render at full size and at 32px (favicon range) to check legibility.
- Put a light background (`background:#ddd`) on the test page so dark logos
  are visible.
- `evaluate_script` can query the rendered DOM to confirm elements exist and
  measure their bounding boxes.

---
name: rust_readline_arrow_keys_rustyline
description: 'Why Rust''s read_line ignores arrow keys (raw escape sequences) and how to fix it with the rustyline crate (line editing, history, key bindings), plus alternatives like reedline and crossterm'
tags:
- rust
- stdin
- readline
- rustyline
- terminal
- arrow-keys
- history
created_at: '2026-09-29T09:23:01.982204344+02:00'
---

# Rust: interactive line editing (arrow keys, history) instead of read_line

## Problem
`std::io::stdin().read_line()` reads raw bytes. Arrow keys arrive as ANSI escape
sequences (`\x1b[A` etc.) and get inserted into the buffer as garbage. Cursor
movement and history are line-editor features, not stdin features — so
`read_line` cannot be "fixed", it must be replaced.

## Solution: rustyline (standard choice)
```toml
[dependencies]
rustyline = "15"
```

```rust
use rustyline::{DefaultEditor, Result};

fn main() -> Result<()> {
    let mut rl = DefaultEditor::new()?;
    loop {
        let line = rl.readline(">> ")?;
        rl.add_history_entry(line.as_str())?; // enables Up/Down history
        println!("You typed: {line}");
    }
}
```

Current API (verified on docs.rs, rustyline 15.x):
- `DefaultEditor::new() -> Result<DefaultEditor>` (Editor with no helper + DefaultHistory)
- `readline(prompt) -> Result<String>`
- `add_history_entry(&str) -> Result<bool>`
- Errors: `ReadlineError::Interrupted` (Ctrl-C), `Eof` (Ctrl-D) — handle these to exit cleanly.
- Extensibility via `Helper` trait: completion, hints, highlighting, validation (multi-line).
- Custom key bindings behind the `custom-bindings` feature (`bind_sequence`, `KeyEvent`, `Cmd`).

## Features out of the box
- Left/Right cursor movement, Up/Down history
- Home/End, Ctrl+A/E, Ctrl+W (delete word), Ctrl+U/K (delete to start/end)
- Alt+B/F word jumps, undo (Changeset)

## Alternatives
- `reedline` — nushell's editor; menus, snippets, more modern
- `dialoguer` — prompts/confirmations, not a REPL
- `crossterm` / `termion` — DIY raw key events; only for TUIs

## Caveat
rustyline requires an interactive TTY. For piped stdin support, gate it:
```rust
use std::io::IsTerminal;
if std::io::stdin().is_terminal() { /* rustyline path */ } else { /* read_line path */ }
```

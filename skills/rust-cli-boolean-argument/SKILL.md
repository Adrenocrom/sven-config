---
name: rust_cli_boolean_argument
description: 'Three ways to read a boolean command-line argument in Rust: str::parse for "true"/"false" values, flag-presence check, or clap derive with ArgAction::SetTrue'
tags:
- rust
- cli
- boolean
- clap
- std-env
- arguments
- parsing
created_at: '2026-09-29T09:41:02.416702253+02:00'
---

# Reading a boolean CLI argument in Rust

CLI args arrive as `String`s via `std::env::args()`; convert to `bool` one of three ways.

## 1. Parse "true"/"false" (value-style arg: `myapp true`)

```rust
let arg = std::env::args().nth(1).unwrap_or_else(|| {
    eprintln!("usage: myapp <true|false>");
    std::process::exit(1);
});
let value: bool = arg.parse().unwrap_or_else(|_| {
    eprintln!("invalid boolean: {arg}");
    std::process::exit(1);
});
```

Caveat: `bool::from_str` accepts ONLY exact `"true"` / `"false"` (case-sensitive). For "1"/"yes"/"TRUE" etc., match manually:

```rust
let value = match arg.to_lowercase().as_str() {
    "true" | "1" | "yes" => true,
    "false" | "0" | "no" => false,
    _ => panic!("invalid boolean: {arg}"),
};
```

## 2. Flag presence = true (idiomatic for flags: `myapp --verbose`)

```rust
let verbose = std::env::args().any(|a| a == "--verbose");
```

## 3. clap derive (best for real CLIs)

```toml
clap = { version = "4", features = ["derive"] }
```

```rust
use clap::Parser;

#[derive(Parser)]
struct Args {
    #[arg(long)]
    verbose: bool, // ArgAction::SetTrue automatically: true iff --verbose passed
}
```

## Rule of thumb
- User types a value (`myapp true`) → option 1
- User passes a flag (`myapp --verbose`) → option 2 or 3
- Anything non-trivial → clap (free help text, error messages, `--help`)

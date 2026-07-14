---
name: docs_rs_lookup
description: How to find and use Rust crate documentation from docs.rs
tags:
- rust
- documentation
- docs.rs
- crates
created_at: '2026-07-13T19:33:03.444016+00:00'
---

## docs.rs URL Pattern

```
https://docs.rs/{crate-name}/{version}/
```

- **Latest version**: `https://docs.rs/{crate-name}/latest/`
- **Specific version**: `https://docs.rs/{crate-name}/1.2.3/`

## Common Crates for BDPT Project

| Crate | URL |
|-------|-----|
| nalgebra (math) | https://docs.rs/nalgebra/latest/nalgebra/ |
| rand (RNG) | https://docs.rs/rand/latest/rand/ |
| image (I/O) | https://docs.rs/image/latest/image/ |
| rayon (parallelism) | https://docs.rs/rayon/latest/rayon/ |

## When to Use docs.rs

- **Primary source** for API documentation
- Always shows latest published version
- Includes dependency tree and cross-links
- Links to source code on GitHub

## Alternatives

1. **crates.io** — Package registry, version history
2. **cargo doc --open** — Local docs (offline)
3. **GitHub** — Source code, examples, issues


---
name: polar_primes_generator
description: 'How the polar_primes Rust tool works: CLI params, Sacks spiral math,
  --fill relative scaling independent of --n, --color support, --offset-x/--offset-y
  as fraction of half-image, PNG output naming'
tags:
- rust
- primes
- polar
- spiral
- png
- image
- sacks
created_at: '2026-08-31T10:34:23.727170+00:00'
---

# Polar prime spiral image generator (Rust)

Project: `polar_primes` — CLI tool that renders primes as polar coordinates (Sacks spiral) into a PNG.

## Math
- Prime p is plotted at angle = p * angle_step (radians), radius = p * scale.
- angle_step = 1.0 rad gives the classic Sacks spiral (Archimedean spiral r = scale * theta).
- Spiral center = (width/2 + offset_x * width/2, height/2 + offset_y * height/2); offsets are FRACTIONS OF THE HALF-IMAGE (0.5 = half of the half-width), NOT pixels and NOT spiral units.

## Layout
- `Cargo.toml`: deps `png = "0.17"`, `chrono = "0.4"`.
- `src/main.rs`: single file — Params::parse, parse_color(), named_color(), sieve(), render(), draw_dot(), write_png(), main().
- `type Rgba = (u8, u8, u8, u8)`.

## CLI parameters (all optional, defaults shown)
- `--n 1000`            primes up to n (sieve of Eratosthenes)
- `--width 1000` / `--height 1000`  image size in px
- `--scale 0.0`         absolute px per integer; > 0 overrides --fill
- `--fill 1.0`          relative image scaling, INDEPENDENT of --n: largest prime lands at fill * image_radius. fill > 1 crops outer primes. Used when scale <= 0.
- `--dot-radius 1.5`    dot radius in px
- `--angle-step 1.0`    radians per integer (1.0 = Sacks spiral)
- `--color white`       #RRGGBB, #RRGGBBAA or name (white black red green lime blue yellow cyan magenta orange purple pink gray)
- `--offset-x 0.0`      shift spiral center horizontally as a FRACTION OF THE HALF-IMAGE WIDTH (0.5 = half of the half-width to the right, -1.0 = one half-width to the left)
- `--offset-y 0.0`      shift spiral center vertically as a fraction of the half-image height (positive = down, negative = up)
- `--output ""`         empty = auto `image_YYYYMMDD.png` (chrono::Local::now().format("%Y%m%d"))

## Implementation notes
- Sieve uses `vec![false; n+1]`, marks multiples from i*i — O(n log log n).
- Relative sizing: scale = max_r * fill / largest_prime, so changing --n only changes point density, never the spiral extent. Absolute --scale wins over --fill.
- max_r = min(w,h)/2 - dot_radius - 1, clamped >= 0. Computed from image size only, NOT affected by offsets — offsets shift the drawn spiral, they do not rescale it.
- Offsets are fractions of the half-image: cx = w/2 + offset_x * w/2. This is proportional across image sizes and --n values, and fractional values are meaningful. HISTORY: an earlier version multiplied offsets by the effective `scale` (spiral units) — with defaults that made 1 unit ≈ 0.5 px, so small offsets looked like "not working". Do NOT reintroduce scale-based offsets.
- draw_dot clips to image bounds; i64 casts avoid usize underflow for negative coords. Offsets can push dots fully outside; clipping handles that.
- Color alpha is blended source-over onto the opaque black background (buffer stays fully opaque).
- parse_color: is_ascii() guard before byte slicing; named colors use CSS values.
- PNG via png crate: ColorType::Rgba, BitDepth::Eight, BufWriter.
- Validation: width/height >= 1, dot_radius > 0, angle_step != 0, fill > 0; unknown args rejected with usage text.
- Console prints the effective pixel shift: "Offset: (x, y) of half-image = (px, px)".

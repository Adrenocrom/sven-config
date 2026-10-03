---
name: kallirs_uefi_car_game
description: 'Kallirs UEFI car game: core gameplay + bridges/dirt/wider road + 3-stage steering implemented; remaining work is polish/tuning.'
tags:
- rust
- uefi
- no-std
- game
- kallirs
- car-game
- build-plan
created_at: '2026-10-03T19:34:01.194090446+02:00'
---

---
name: kallirs_uefi_car_game
description: 'Kallirs UEFI car game: core gameplay + bridges/dirt/wider road + 3-stage steering implemented; remaining work is polish/tuning.'
tags:
- rust
- uefi
- no-std
- game
- kallirs
- car-game
- build-plan
created_at: '2026-10-03T19:34:01.194090446+02:00'
---

# Kallirs — UEFI car game build plan

Repo: `kallirs` (Rust, `no_std`, `uefi` crate 0.41, target `x86_64-unknown-uefi`).
Spec lives in `idea.md`. No new crates allowed — everything is hand-rolled.

## Game spec (from idea.md)
- Car fixed near screen bottom; world scrolls **downwards** (car drives "up").
- Player only steers left/right; car turns **45°** left/right (smooth, "3D rotation" look).
- Track in `world.rs`, sprites in `assets.rs`.
- Obstacles: **balls rolling left↔right** across the road, small **construction sites**.
- **Meadow** left/right of the road; driving off the road = game over.
- Collect **shamrocks** for points. **One life**.

## Current state: core game + bridges/dirt + 3-stage steering (compiles clean)
- `main.rs`: UEFI scaffolding — GOP setup + min-resolution guard (640x400), 16 ms periodic timer event for frame pacing, keyboard (LEFT/RIGHT = discrete heading steps, SPACE = primary action, ESC = escape), back-buffer blit per frame, clean shutdown. No held-key emulation — steering is discrete key presses.
- `world.rs`: `Track` = centerline `center_x(d)` over driven distance, built from 240px segments with cosine-eased bends in an 8-slot ring buffer (endless road, O(1) per scanline); `Rng` (xorshift32, `next`/`below`/`in_range`); `render()` draws per-scanline road (row y → d = scroll + height−y) with edge stripes, dashed center line, mown meadow bands + 16x16 grass texture (`grass_line`/`meadow_line`, tile rows locked to world distance so grass scrolls with the road) + hashed flower dots; **`ROAD_HALF_W=200`** (widened from 160), `EDGE_W=6`.
- `assets.rs`: string-map `Sprite` (char → palette, '.' transparent) implementing `draw::Bitmap`; CAR 16x26 top-down drawn at 32x52 (CAR_SCALE=2); SHAMROCK 13x13 drawn at 26x26 (SHAM_SCALE=2); DIRT 12x8 drawn at 24x16 (DIRT_SCALE=2, palette 'd'/'k'); GRASS 16x16 tile; `draw_ball` = kids' rainbow ball (6 wedges rotating with roll phase, white rim, white hub dot; wedge picked per pixel by cross-product tests — no atan2 in core); `draw_barrier` (diagonal orange/white stripes + dark outline); **`draw_bridge(screen, y)`** — full-screen-width overhead bridge: deck band (BRIDGE_DECK_H=46) with dark underside, railing with posts on top (saturating_sub against top-of-screen clipping!), piers below every 160px. `BRIDGE_DECK_H` is pub (game.rs uses it for shadow placement).
- `draw.rs` (utilities): `Screen` (`fill`, `rect`, `rect_f`, `circle`, `text`, `text_centered`, `text_num`, `px`, `hline`, `width`, `height`), `Bitmap` trait, `blit`, `blit_scaled` (integer scale, nearest-neighbor), `blit_rotated` (inverse-rotation affine blit with `scale` param, vertical squash by `cos(angle)` clamped ≥0.55), font8x8, `Color`, `num_width`.
- `math.rs` (utility, keep): no_std `sin`/`cos` (Taylor, ~3e-5 err), `floor`, `ceil`. NOTE: no `sqrt`/`atan2` in core — use `max(hw, hh)` bounds, squared distances, or cross-product angle tests.
- `game.rs`: `Phase::{Title,Playing,GameOver}`; **3-stage heading steering** (`STEER_STAGES=3`, `steer: i32` ∈ {-3..+3}, persists between presses; each press steps one stage; target angle = `steer/STEER_STAGES*MAX_ANGLE` → 15/30/45° per side; `car_angle` eases to target, lateral speed = `car_angle/MAX_ANGLE*CAR_SPEED`, CAR_SPEED=6); scroll speed ramps 3→9; spawns in fixed `[Option<Spawn>; 24]` slots rolled every 130px up to 900px ahead (P_BALL=22, P_SITE=22, P_SHAMROCK=34, P_DIRT=20, rest nothing); balls roll ±1.5–3.5 px/frame, bounce at road edges; **`ensure_site_ahead()`** guarantees ≥1 construction site ahead (spawns one at car_d+350..800 if none exists beyond car_d+150); **bridges** spawn at FIRST_BRIDGE_D=1400 then every 1600–2600px (randomized), Thing::Bridge, no collision; collisions: off-road → game over, ball = circle-vs-car-rect, site = AABB, shamrock = AABB → score+1; HUD with shadow strip; game-over overlay.
- **Render order in `render_world`** (critical for the trees/bridge look): road → dirt (on asphalt) → bridge shadows (per-scanline dark band on road below deck, follows road curve via `bridge_shadow()`) → things (balls/sites/shamrocks) → **car** → **bridge decks** (drawn over the car — the bridge blocks the view of the car) → HUD. `let spawns = self.spawns;` copy keeps borrows disjoint while `bridge_shadow` needs `&mut self.track`.
- `run.sh`: builds release, packs `disk.img` (FAT, `efi/boot/bootx64.efi`), boots in QEMU with OVMF (`OVMF=...` overridable).

## Remaining polish ideas (optional)
- Difficulty selection on the title screen (speed/bend presets).
- Sound is impossible (no audio protocol in scope); visual feedback instead (screen shake on crash?).
- Trees/bushes as meadow sprites, speed display in HUD.
- Tune: MAX_BEND=100/SEG_LEN=240 gives max road slope ≈0.65 px/px; car lateral speed 6 px/frame must outpace `speed * slope` at MAX_SPEED=9 → 5.85, just barely OK — if raising MAX_SPEED, raise CAR_SPEED or lower MAX_BEND. With the staged-heading model the car only reaches full lateral speed while actually at ±45°, so if steering feels sluggish raise CAR_SPEED or ANGLE_EASE.
- Bridge tuning: BRIDGE_DECK_H=46 hides the car (52px tall) for ~5 frames at MAX_SPEED — raise BRIDGE_DECK_H or lower BRIDGE_GAP_MIN/MAX for more/longer occlusion.

## Conventions/pitfalls (from uefi_rust_development skill)
- Physics that mutates shared state: free functions with disjoint borrows, or hoist reads before `iter_mut` loops (borrow checker). For render passes needing `&mut self.track` inside iteration over spawns, copy the spawn table (`let spawns = self.spawns;` — it's `Copy`).
- UEFI key events are press-only — steering is discrete steps, so no release emulation needed.
- Frame pacing via periodic timer event + `wait_for_event`, never `stall`.
- rng: xorshift32, state never 0 (`| 1`).
- u32 subtraction on screen coordinates underflows — use `saturating_sub` for anything that can go above the screen top (bridge railing).
- Test with `./run.sh` (QEMU + OVMF); `disk.img` is pre-made, only `mcopy` runs per build.

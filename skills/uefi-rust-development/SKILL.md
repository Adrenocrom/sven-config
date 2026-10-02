---
name: uefi_rust_development
description: 'How to write UEFI apps in Rust (uefi crate): custom target, console I/O via system::with_stdin, keyboard input incl. held-key emulation via countdown arrays, timer events for frame pacing, GOP double buffering, stall/reset, boot vs runtime services.'
tags:
- rust
- uefi
- keyboard-input
- timer-events
- double-buffering
- gop
- no-std-math
created_at: '2026-10-01T21:24:26.767329059+02:00'
---

## Other boot/runtime helpers
- `uefi::boot::stall(Duration)` — busy-wait pause (avoid for frame pacing; use timer events). `Duration::new(secs, nanos)` or `from_millis`.
- `uefi::boot::close_event(event)` — takes `Event` by value (not `Copy`); only close events you created.
- `uefi::runtime::reset_system(...)` — reset/reboot; `uefi::boot::shutdown()` — power off.

## no_std game entity management (verified in a breakout game with power-ups)

### Fixed-size `Option` slot arrays instead of `Vec`
`no_std` has `alloc` (the uefi crate ships an allocator), but for a bounded entity count a fixed array of `Option<T>` is simpler and has no failure mode:
```rust
const MAX_BALLS: usize = 8;
balls: [Option<Ball>; MAX_BALLS],   // Ball: Copy
// iterate live ones: for ball in self.balls.iter().flatten() { ... }
// find a free slot: for slot in self.balls.iter_mut() { if slot.is_none() { *slot = Some(b); break; } }
// all gone: self.balls.iter().all(|b| b.is_none())
```
- Requires `T: Copy` for the take-out-by-value pattern (`let Some(mut d) = self.drops[i] else { continue };` — copies out, leaving `self` borrow-free while applying effects).
- `Option<T>` is `Copy` when `T` is, so `[None; N]` works as a reset.

### Disjoint borrows: physics as a free function, not a method
Per-entity physics that also mutates shared state (e.g. a ball destroying a brick) fights the borrow checker as a method — `self.balls[i].as_mut()` borrows `self.balls`, then `self.bricks` is a second borrow of `self`. Fix: make the step a **free function** taking disjoint borrows explicitly:
```rust
let step = {
    let Some(ball) = self.balls[i].as_mut() else { continue };
    step_ball(ball, self.width, self.height, self.paddle_x, self.paddle_w,
              paddle_top, grid_left, &mut self.bricks)   // two disjoint fields — fine
};  // borrow ends here; `self` fully usable again (spawn drops, decrement counters)
```
The block scopes the ball borrow; the function returns an event struct (`struct BallStep { lost: bool, brick: Option<usize> }`) so the caller reacts afterwards. This is the standard no-borrow-conflict pattern for multi-entity games.

### Randomness: xorshift32 (UEFI has no entropy source)
There is no RNG protocol for apps and no `std::time` clock; seed from something device-specific (e.g. screen resolution) and use xorshift32:
```rust
rng: (0x9E37_79B9 ^ (width as u32) ^ ((height as u32) << 16)) | 1,  // never 0!
fn rand(&mut self) -> u32 {
    let mut x = self.rng;
    x ^= x << 13; x ^= x >> 17; x ^= x << 5;
    self.rng = x; x
}
// percent roll: self.rand() % 100 < DROP_CHANCE
```
- State must never be 0 (xorshift gets stuck) — `| 1` guarantees it.
- Fine for gameplay variety; not for anything adversarial.

### Power-up/collectible pattern (breakout)
- Destroyed brick rolls a spawn chance; drop = colored square with a letter glyph (`G` grow / `S` shrink / `M` multiball), falls at constant speed, caught by AABB overlap with the paddle, discarded below the screen.
- Multiball: clone a live ball and rotate its velocity vector ±25° (`dx' = dx·cos a − dy·sin a`, `dy' = dx·sin a + dy·cos a`) — rotation preserves speed, so paddle bounce math stays consistent.
- Life is lost only when the **last** ball falls off (`balls.iter().all(|b| b.is_none())`), not per ball.
- Reset transient effects (paddle size, drops) on life loss and level clear; carry score/lives across levels.

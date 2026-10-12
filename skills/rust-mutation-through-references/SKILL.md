---
name: rust_mutation_through_references
description: 'How to mutate objects through references in Rust: split borrows, interior mutability (Cell/RefCell/Mutex), Rc<RefCell<T>>, and arena/ID patterns for shared or graph-like mutable state'
tags:
- rust
- borrow-checker
- mutability
- refcell
- interior-mutability
- ownership
created_at: '2026-09-28T12:41:07.397528698+02:00'
---

# Rust: mutating through references — patterns and fixes

Core rule: **aliasing XOR mutation** — many `&T` *or* one `&mut T`, never both. The compiler rejects designs where other languages allow shared mutation (source of iterator invalidation / data races).

## What works without tricks
Sequential `&mut` calls are fine — borrows end at last use (NLL). Storing a borrow in a variable/field keeps it alive; scoping fixes it.

## Fix ladder (prefer in order)
1. **Restructure**: pass `&mut` down the call chain, return values up. Don't store `&mut` in structs.
2. **Split borrows**: disjoint struct fields can be borrowed mutably at once: `let (p, e) = (&mut g.player, &mut g.enemies);`. For slices: `split_at_mut`.
3. **Free functions over `&mut self` methods** when iterating a field of `self` and calling back into `self` (classic error: `for e in &mut self.enemies { self.damage(e) }` → make `damage(e: &mut Enemy)` a free fn).
4. **Interior mutability** (mutate via `&self`):
   - `Cell<T>`: `Copy` types, no runtime check
   - `RefCell<T>`: general; borrow rules checked at runtime → panics on conflict
   - `Mutex`/`RwLock`/atomics: thread-safe version
5. **`Rc<RefCell<T>>`**: Java/Python-style shared mutable object. Runtime panics on borrow conflicts; cycles leak → use `Weak`.
6. **IDs/indices instead of references** for graphs, trees with parent pointers, doubly-linked structures: store nodes in `Vec`/`HashMap`, refer by `usize`/key. The idiomatic Rust answer for self-referential data (arenas, ECS).

## Common error signatures
- "cannot borrow `x` as mutable because it is also borrowed as mutable" → overlapping `&mut`, split or scope
- "cannot borrow `self.x` as mutable because `self` is also borrowed" → method-call-while-iterating, use free fn or collect
- "cannot borrow data in a `&` reference as mutable" → need `&mut`, or `Cell`/`RefCell` if API forces `&self`

## Rule of thumb
Restructure > split borrows > interior mutability > `Rc<RefCell>` > arena/IDs. Each step is a bigger hammer with more runtime risk.

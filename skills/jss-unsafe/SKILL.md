---
name: jss-unsafe
description: Focused scan of Rust unsafe code
argument-hint: <optional: path to scan>
allowed-tools: Read Grep Glob Bash
---

# jss-unsafe

Focused scan of `unsafe` code in Rust projects. Detailed FFI checks are handled by `/jss-ffi`. This skill only briefly checks FFI-related unsafe to avoid duplication.

## Scan Scope

If `$ARGUMENTS` is provided, scan only that path. Otherwise scan the entire project.
Always exclude `target/` and `vendor/`.

## Discovery Order

1. Read `Cargo.toml` to determine workspace structure. If workspace, identify all member crates.
2. Check each crate for `#![forbid(unsafe_code)]` / `#![deny(unsafe_code)]`:
   - `forbid`: skip that crate, report as Info: "unsafe_code is forbidden".
   - `deny`: report as Info: "unsafe_code is denied but can be locally overridden with `#[allow]`". Continue scanning.
3. Grep for `unsafe` (file names only) — identify target files
4. Grep for `transmute|transmute_copy`
5. Grep for `ptr::read|ptr::write|ptr::copy|ptr::drop_in_place`
6. Grep for `MaybeUninit`
7. Grep for `Pin::new_unchecked|Pin::get_unchecked_mut`
8. Grep for `union\s+\w+` — union type usage

## Checks

### 1. Unnecessary unsafe (Warning)

- Safe alternatives exist but unsafe is used
- Example: `unsafe { std::str::from_utf8_unchecked() }` → use `from_utf8()`
- Example: `unsafe { slice::from_raw_parts() }` → use safe slice operations

### 2. Missing Bounds Checks (Critical)

- Pointer arithmetic or slice access without length/range validation before the unsafe block
- Raw pointer dereference without null check
- `slice::from_raw_parts(ptr, len)` without validating `len`

### 3. transmute Misuse (Critical)

- `std::mem::transmute` without size/alignment guarantees
- Safe alternatives: `from_bits`, `to_ne_bytes`, `bytemuck::cast`/`bytes_of`, `zerocopy`, `TryFrom`/`TryInto`

### 4. unsafe impl (Warning)

- `unsafe impl Send for T` / `unsafe impl Sync for T` without `// SAFETY:` comment
- Contains raw pointers or `UnsafeCell` but thread-safety justification is insufficient

### 5. unsafe fn (Warning)

- Unnecessarily declared `unsafe fn` — could be safe fn with internal unsafe block
- `unsafe fn` without explicit `unsafe {}` blocks inside (Rust 2024 edition enables `unsafe_op_in_unsafe_fn` lint by default)

### 6. Dangerous Function Usage (Critical ~ Warning)

- `ptr::read` / `ptr::write` / `ptr::copy` / `ptr::copy_nonoverlapping` — alignment, validity, or overlap requirements not met → UB (Critical)
- `MaybeUninit::uninit().assume_init()` — treating uninitialized memory as initialized → UB (Critical)
- `MaybeUninit::zeroed().assume_init()` — used with types where zero is not a valid value → UB (Critical)
- `Pin::new_unchecked` / `Pin::get_unchecked_mut` — Pin contract violation → UB (Warning)
- `union` field access — reading wrong variant → UB (Warning)

### 7. SAFETY Comment Review

- `// SAFETY:` comments present — review justification. If sound, do not report.
- Insufficient comments (`// SAFETY: trust me`, `// SAFETY: safe`, etc.) → Warning
- No `// SAFETY:` comment at all on unsafe block → Warning

### 8. FFI unsafe (brief)

- Panic potential inside `extern "C"` fn → Warning (process abort)
- Null pointer dereference without check → Critical
- Detailed FFI checks are in `/jss-ffi`. This skill only checks the above 2 items.

## Statistics Output

```
## unsafe Statistics (~N — Grep-based, may include false matches in comments/strings)
- unsafe blocks: ~N (in M files)
- unsafe fn: ~N
- unsafe impl: ~N
- unsafe trait: ~N
- extern "C" fn: ~N
- #![forbid(unsafe_code)] crates: N / total M
- #![deny(unsafe_code)] crates: N / total M
```

If `cargo-geiger` is installed, run `cargo geiger --output-format ascii 2>/dev/null | head -50` for more accurate statistics.

## Suppression

Follow suppression rules in CLAUDE.md.

## Output

Follow the common output format in CLAUDE.md.

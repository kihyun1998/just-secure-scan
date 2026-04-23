---
name: jss-ffi
description: Scan Rust FFI boundary security issues
argument-hint: <optional: path to scan>
allowed-tools: Read Grep Glob Bash
---

# jss-ffi

Scans FFI (Foreign Function Interface) boundary security in Rust projects. This skill provides more detailed FFI checks than `/jss-unsafe`, which only performs brief FFI checks to avoid duplication.

## Scan Scope

If `$ARGUMENTS` is provided, scan only that path. Otherwise scan the entire project.
Always exclude `target/` and `vendor/`.

## Discovery Order

1. Read `Cargo.toml` to determine workspace structure. Check `[dependencies]` for FFI-related crates: `libc`, `bindgen`, `cc`, `cxx`, `pyo3`, `napi`, `wasm-bindgen`
2. Grep for `extern\s*"C"` — FFI function definitions
3. Grep for `#\[no_mangle\]` — externally exposed functions
4. Grep for `extern\s*\{` — external function declarations (C library bindings)
5. Grep for `CStr|CString|c_char|c_void` — C type usage
6. Grep for `#\[pyfunction\]|#\[pyfn\]` — pyo3 functions
7. Grep for `#\[napi\]` — napi functions
8. Grep for `#\[wasm_bindgen\]` — wasm-bindgen functions

## Checks

### 1. Null Pointers (Critical)

- `extern "C"` functions dereferencing `*const T` / `*mut T` parameters without null check
- `CStr::from_ptr(ptr)` called without null check
- External function return values used without null check when the return type is a pointer

### 2. String Safety (Warning)

- C strings (`*const c_char`) received and converted with `to_str().unwrap()` without UTF-8 validation
- `CString::new()` called with input that may contain null bytes without validation
- C-originated strings converted to `String` without length limits

### 3. Panic Across FFI (Warning)

- Panic potential inside `#[no_mangle] pub extern "C" fn` — causes process abort
- Panic-inducing code: `unwrap()`, `expect()`, `panic!()`, index access (`[]`)
- Not wrapped in `catch_unwind`
- Recommended: `Result` return + error code conversion, or `catch_unwind` + error return

### 4. Memory Ownership (Critical ~ Warning)

Scope: only checks patterns visible within a single file. Cross-file ownership tracking is reported as "manual verification required".

- `into_raw()` called without a corresponding `from_raw()` deallocation path in the same file/module (Warning)
- `Box::from_raw` used on C-allocated memory — allocator mismatch (Critical)
- Mixing `libc::free` and `Box::drop` (Critical)
- Rust-allocated memory passed to C without exporting a deallocation function (`free_*`) (Warning)

### 5. External Function Binding Safety (Warning)

- `extern { fn ... }` blocks with manually written signatures — recommend `bindgen`
- Structs or enums passed to C without `#[repr(C)]`
- Missing error code / `errno` check after calling C functions

### 6. Callback Safety (Warning)

- Panic potential in Rust functions used as C callbacks
- Callback functions not declared with `extern "C"` ABI

### 7. Thread Safety (Warning)

- C libraries called via FFI that are not thread-safe — if the wrapper has manual `Send`/`Sync` impl, verify the justification
- C functions that modify global state called from multiple threads

### 8. High-Level FFI Bindings (Warning ~ Info)

Additional checks when pyo3, napi, or wasm-bindgen are used:

**pyo3:**
- Panic potential inside `#[pyfunction]` — crashes the process if not converted to a Python exception
- Accessing Python objects inside `Python::allow_threads` (accessing without GIL is UB)

**napi:**
- Panic potential inside `#[napi]` — crashes the process if not converted to a JS exception
- `Buffer` ownership — Rust taking ownership of a JS-passed Buffer may conflict with GC

**wasm-bindgen:**
- `JsValue` leaks — patterns using `forget()` to skip explicit deallocation
- Missing bounds checks on linear memory access

## Statistics Output

```
## FFI Statistics
- extern "C" fn (exposed): ~N
- extern { fn ... } (bindings): ~N
- #[no_mangle]: ~N
- FFI-related crates: libc, bindgen, ...
- High-level bindings: pyo3 / napi / wasm-bindgen (if applicable)
```

## Suppression

Follow suppression rules in CLAUDE.md.

## Output

Follow the common output format in CLAUDE.md, including the inline axis line (Confidence / Blast / Layer) under every finding.

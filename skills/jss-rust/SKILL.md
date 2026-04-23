---
name: jss-rust
description: Comprehensive Rust crate security scan
argument-hint: <optional: path to scan>
allowed-tools: Read Grep Glob Bash
---

# jss-rust

Scans security issues in Rust crates.
If `$ARGUMENTS` is provided, scan only that path. Otherwise scan the entire project.

## Discovery Strategy

Do not read entire files upfront. Exclude `target/` and `vendor/` from all Grep searches.

### 0. Project Structure (highest priority)

1. Read `Cargo.toml` to determine `[workspace]` structure. If workspace, identify all member crates' `Cargo.toml`, `build.rs`, `src/lib.rs`, `src/main.rs`.
2. Determine whether each crate is a library (`[lib]`) or binary (`[[bin]]`). This affects panic check scope.
3. Check for `#![forbid(unsafe_code)]`. If declared, skip unsafe checks and report as Info: "unsafe_code is forbidden".

### 1–6. Discovery Targets

1. Grep for `unsafe` (file names only) — unsafe block locations
2. Grep for `Command::new|process::Command` — command injection candidates
3. Grep for `extern\s*"C"` — FFI boundaries
4. Each crate's `build.rs` — supply chain risk
5. Grep for `unwrap\(\)|\.expect\(` — panic candidates (library crates only)
6. Grep for `Deserialize|from_reader|from_slice|from_str` — deserialization candidates
7. Grep for `Path::new|PathBuf::from` — path traversal candidates
8. Grep for `query\(|execute\(` — SQL injection candidates

## Checks

### 1. unsafe Blocks (Critical ~ Warning)

- **Unnecessary unsafe** (Warning): safe alternatives exist
- **Missing bounds checks** (Critical): pointer arithmetic or slice access without length/range validation inside unsafe blocks
- **transmute misuse** (Critical): `std::mem::transmute` without size/alignment guarantees
- **unsafe impl** (Warning): manual `Send`/`Sync` implementation without justification comment
- `// SAFETY:` comments present — review the justification. If sound, do not report. If insufficient — Warning.
- No `// SAFETY:` comment at all — Warning.
- Unnecessarily declared `unsafe fn` (could be safe fn + internal unsafe block) — Warning.

### 2. Panics in Library Crates (Warning)

Check only in library crates (`[lib]`). Skip binary crates (`[[bin]]`).

- `unwrap()`, `expect()`, `panic!()`, `unreachable!()`, `todo!()`, `unimplemented!()` in library modules.

**Exceptions (do not report):**
- Test code (`#[cfg(test)]`, `#[test]`)
- `// INVARIANT:` or similar invariant-explaining comments present
- `Mutex::lock().unwrap()` — mutex poisoning panic is usually correct behavior
- Unwrap on constant input: `Regex::new("literal").unwrap()`, `"123".parse::<i32>().unwrap()`, etc.
- `once_cell`/`LazyLock`/`lazy_static` initialization blocks
- `expect()` with a meaningful message and clearly intentional use → downgrade to Info

### 3. Command Injection (Critical ~ Info)

- `Command::new(user_input)` — user input directly as command name (Critical)
- `Command::new("sh").arg("-c").arg(format!(...))` — shell command composition with user input (Critical)
- `.arg(user_input)` — no shell expansion (`Command` calls `execvp` directly), but may allow unintended flags (argument injection) (Info)

### 4. Deserialization (Warning ~ Critical)

- `serde_yaml` deserializing external input — YAML bomb possible (Warning)
- `bincode`, `postcard`, etc. binary formats trusting length prefixes for large memory allocations (Warning)
- `serde_json::from_str(user_input)` without `#[serde(deny_unknown_fields)]` (Info)
- `Deserialize` from untrusted sources without size/depth limits (Warning)

### 5. SQL Injection (Critical)

- `sqlx::query`, `diesel::sql_query`, `rusqlite::execute`, etc. using `format!` to insert user input into queries
- String concatenation for query building instead of parameter binding (`$1`, `?`)

### 6. Path Traversal (Warning)

- `Path::new(user_input)` or `PathBuf::from(user_input)` without `../` filtering
- User-provided paths used in file I/O without `canonicalize()` or base path validation

### 7. Integer Overflow (Warning)

- Narrowing `as` casts: `u64 as u32`, `i64 as i32`, `usize as u16`, etc. Grep pattern: `as u8|as u16|as u32|as i8|as i16|as i32`
- Arithmetic in `pub fn` receiving external input without `checked_*` or `saturating_*`

### 8. Supply Chain — build.rs (Warning ~ Critical)

- `build.rs` with network calls (`reqwest`, `ureq`, `curl`, `TcpStream`) (Critical)
- `build.rs` writing files to `$HOME` or paths outside the project (Critical)
- `build.rs` with `Command::new` — verify what is executed (Warning)
- Proc macro crates with similar patterns (Warning)

### 9. FFI Boundary (Critical ~ Warning)

Detailed FFI checks are in `/jss-ffi`. This skill checks only the essentials:

- `extern "C"` function dereferencing `*const`/`*mut` without null check (Critical)
- C string (`*const c_char`) received without null check before `CStr::from_ptr` (Critical)
- Missing error check / errno check on FFI return values (Warning)
- `#[no_mangle] pub extern "C" fn` with panic potential — process abort (Warning)

### 10. ReDoS (Warning)

- `fancy-regex` or `pcre2` compiling user input as regex (Warning)
- Standard `regex` crate is RE2-based (no backtracking, ReDoS-immune). However, `Regex::new(user_input)` can cause compilation-time DoS — `RegexBuilder` `size_limit` not set (Info).

### 11. TOCTOU (Info)

- Only report `Path::exists()` followed by `File::open()`/`File::create()` patterns involving `env::temp_dir()`, `/tmp`, or `tempdir` paths. Do not report for general paths.

### 12. Crypto (Warning)

- `md5`, `sha1` crate usage (Warning if used for security purposes; ignore if used for checksums)
- Custom cryptography implementations
- Hardcoded keys/IVs (`let key = b"..."`, `let iv = [0u8; ...]`)

### 13. Async Security (Warning ~ Info)

- `unwrap()` inside `tokio::spawn` — only that task panics and the error is silently swallowed (Warning)
- External requests without `timeout` — indefinite blocking DoS (Info)
- `std::sync::Mutex` (synchronous mutex) in async code — potential deadlock (Warning)

### 14. Dependencies (Info)

- Run `cargo audit --json 2>/dev/null | head -100` if available and include summary. Otherwise check `Cargo.lock` presence and major dependency versions. Note CVE cutoff limitation.
- Overly broad feature activation in `Cargo.toml`

## Suppression

Follow suppression rules in CLAUDE.md.

## Output

Follow the common output format in CLAUDE.md, including the inline axis line (Confidence / Blast / Layer) under every finding.

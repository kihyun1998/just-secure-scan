---
name: jss-deps
description: Scan dependency CVEs and security issues (npm/cargo)
argument-hint: <optional: npm or cargo>
allowed-tools: Read Grep Glob Bash
---

# jss-deps

Scans project dependencies for known vulnerabilities (CVEs) and security configuration issues.
Currently supports npm (Node.js) and cargo (Rust) ecosystems.

## Stack Detection

If `$ARGUMENTS` is `npm` or `cargo`, scan only that ecosystem.
If no argument, auto-detect based on the presence of `package.json` and `Cargo.toml`.

## npm Ecosystem

### 1. npm audit (Critical ~ Info)

```bash
npm audit --json 2>/dev/null | head -100
```

If available, parse results and classify by severity:
- `critical`, `high` → Critical
- `moderate` → Warning
- `low` → Info

If npm is not installed or the command fails, read `package-lock.json` directly to check major package versions. Note that CVEs published after Claude's training data cutoff cannot be detected. Do not rely on hardcoded CVE lists — prioritize external tool results.

### 2. Security Configuration (Info)

- `package.json` missing `engines` field — Node.js version unconstrained
- Check for `overrides`/`resolutions` pinning vulnerable packages

## Cargo Ecosystem

### 1. cargo audit (Critical ~ Info)

```bash
cargo audit --json 2>/dev/null | head -100
```

If available, parse results and classify by severity. If unavailable, read `Cargo.lock` directly to check major package versions. Note CVE cutoff limitation. Do not rely on hardcoded CVE lists — prioritize external tool results.

### 2. Feature Flag Security (Info)

- Dependencies in `Cargo.toml` not using `default-features = false` with unnecessary features enabled
- Overly broad feature sets like `full`

### 3. build.rs Dependencies (Warning)

- `[build-dependencies]` containing crates with network capabilities (`reqwest`, `ureq`, `curl`)

## Output

Follow the common output format in CLAUDE.md.
Additionally output a dependency summary:

```
## Dependency Summary
- Total dependencies: N
- Vulnerabilities: Critical N / Warning N / Info N
- Scan method: npm audit / cargo audit (or direct lock file analysis)
- CVE data as of: (latest if external tool used, training data cutoff if direct analysis)
```

## Suppression

Follow suppression rules in CLAUDE.md.

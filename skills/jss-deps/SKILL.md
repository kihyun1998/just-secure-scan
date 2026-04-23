---
name: jss-deps
description: Scan dependency CVEs and security issues (npm/cargo/pub)
argument-hint: <optional: npm, cargo, or pub>
allowed-tools: Read Grep Glob Bash
---

# jss-deps

Scans project dependencies for known vulnerabilities (CVEs) and security configuration issues.
Currently supports npm (Node.js), cargo (Rust), and pub (Dart/Flutter) ecosystems.

## Stack Detection

If `$ARGUMENTS` is `npm`, `cargo`, or `pub`, scan only that ecosystem.
If no argument, auto-detect based on the presence of `package.json`, `Cargo.toml`, and `pubspec.yaml`.

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

## Pub (Dart/Flutter) Ecosystem

### 1. dart pub audit (Critical ~ Info)

```bash
dart pub audit --json 2>/dev/null | head -100
```

If available, parse results and classify by severity. If unavailable, read `pubspec.lock` directly to check major package versions. Note CVE cutoff limitation. Do not rely on hardcoded CVE lists — prioritize external tool results.

### 2. Dependency Overrides (Warning)

- `pubspec.yaml` containing `dependency_overrides` — may mask known vulnerabilities or pin insecure versions. Acceptable in development but not in production.

### 3. SDK Constraint (Info)

- `pubspec.yaml` `environment.sdk` using outdated constraint (e.g., `>=2.x.x` when Dart 3+ is available)
- Missing `environment.sdk` constraint entirely

### 4. Git/Path Dependencies (Warning)

- Dependencies sourced from `git:` or `path:` instead of pub.dev — not audited by `dart pub audit`, may contain unreviewed code.
- `git:` dependencies without a pinned `ref:` (commit hash or tag) — vulnerable to upstream changes.

## Output

Follow the common output format in CLAUDE.md, including the inline axis line (Confidence / Blast / Layer) under every finding.
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

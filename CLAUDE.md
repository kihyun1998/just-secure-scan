## just-secure-scan — Common Principles

> The rules below apply only when executing `/secure-scan-*` skills. They do not affect other tasks.

### Severity Levels

| Level | Criteria |
|-------|----------|
| **Critical** | Exploitable by an external attacker without authentication. Directly leads to data leak/modification/deletion |
| **Warning** | Exploitable under specific conditions, or missing a defense layer |
| **Info** | Low direct threat but deviates from security best practices |

All skills must apply the same severity for the same issue. Severity must not differ between detailed skills (`/secure-scan-rls`, etc.) and comprehensive skills (`/secure-scan-nextjs`, etc.).

### Duplicate Prevention

When running multiple skills sequentially, avoid reporting the same issue twice:
- If an issue at the same `file:line` was already reported in the current session, skip it.
- Detailed skills are more thorough than comprehensive skills for overlapping checks. Running a detailed skill after a comprehensive one should only report additional findings.

### Output Format

All skills follow the format below. Omit severity sections with no findings.

```
# secure-scan-<skill-name> Results

## Critical — Immediate action required
- `file:line` — one-line summary
  **Impact:** how an attacker can exploit this
  **Fix:** specific remediation steps

## Warning — Review required
- `file:line` — one-line summary
  **Impact:** under what conditions this becomes a problem
  **Fix:** specific remediation steps

## Info — Recommendations
- `file:line` — one-line summary
  **Fix:** specific remediation steps

## Suppressed (N items)

---
Scope: N files analyzed
Not checked: (if applicable, e.g., "RLS — no SQL migration files found")
These results are based on static code analysis and do not reflect runtime or infrastructure configuration.
```

- If no issues are found, output "No issues found".
- If only suppressed items exist, output "No issues found (N items suppressed)".
- **Secret masking**: mask discovered secrets as first 4 chars + `****` + last 4 chars. Never output the full value.

### False Positive Suppression

**Suppression** (not reported) and **severity downgrade** (lowered severity) are distinct behaviors. Each skill must distinguish them explicitly.

#### Inline Comments

If a line ends with a `secure-scan-ignore: reason` comment, do not report it. Recognize language-specific comment syntax:

- JS/TS/Rust/Go: `// secure-scan-ignore: reason`
- Python/Ruby/YAML: `# secure-scan-ignore: reason`
- SQL: `-- secure-scan-ignore: reason`
- HTML/JSX: `{/* secure-scan-ignore: reason */}` or `<!-- secure-scan-ignore: reason -->`

#### Project Suppression File

Items listed in `.secure-scan-ignore` at the project root are not reported.

```
# .secure-scan-ignore
# Format: file path or glob pattern  # reason
# Suppresses all findings for the matched files.
src/lib/supabase-admin.ts  # server-only module, confirmed not in client bundle
tests/fixtures/**          # test fixtures, intentionally vulnerable code
```

- Supports glob patterns (`*`, `**`)
- Paths are relative to the project root
- Lines starting with `#` are comments; blank lines are ignored

Suppressed items are shown only as a count in the Suppressed section.

### Discovery Principles

> These principles apply only to `/secure-scan-*` skills.

- Do not read entire files blindly. Execute the discovery patterns specified in each skill first.
- If `$ARGUMENTS` is provided, limit scope to that path. However, config files (`.env*`, `Cargo.toml`, `next.config.*`, etc.) are always checked regardless of path restrictions.
- Exclude build artifacts from all grep/search operations: `node_modules/`, `.next/`, `target/`, `dist/`, `build/`, `vendor/`
- Include `Scope: N files analyzed` at the bottom of every report.

### External Tool Fallback

> These rules apply only to `/secure-scan-*` skills.

If external tools (`npm audit`, `cargo audit`, `supabase` CLI, etc.) are installed, use their output:
- `npm audit`: `npm audit --json 2>/dev/null | head -100` — fetch summary only.
- `cargo audit`: `cargo audit --json 2>/dev/null | head -100` — fetch summary only.
- If not installed, read lock files directly for analysis. Note in the report that CVEs published after Claude's training data cutoff cannot be detected.
- If external tool execution fails, ignore the error and proceed with fallback.

### Analysis Scope Limitations

Each skill performs static analysis by reading code and matching patterns. Limitations include:
- Cross-file data flow tracing is limited to 1–2 call chain levels. Beyond that, report as "cannot trace, manual verification required".
- Grep-based statistics (e.g., number of unsafe blocks) may include false matches from comments/strings. Express as "approximately N".

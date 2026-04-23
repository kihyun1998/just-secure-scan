# just-agents Roadmap

This repository aggregates Claude Code agents and skills across multiple domains. Each domain is independent and has its own roadmap section below.

## Shared design principles

- **Skill independence**: slash-command skills cannot inherit from each other, so each skill is self-contained.
- **`CLAUDE.md` = per-domain common principles**: output format, severity, suppression rules live here, scoped by prefix.
- **Token efficiency**: prompts are checklists/directives, not prose.
- **Scoped arguments**: every skill accepts `$ARGUMENTS` to narrow scope (e.g. `/jss-nextjs src/app/`).
- **Embedded discovery strategy**: each skill specifies *which files to look at first*, not just what to check.

---

# Security Scan (`/jss-*`)

Claude Code security review skills. Cover Next.js + Supabase, Rust crates, Flutter, plus stack-agnostic checks (secrets, dependency CVEs).

## Checks by stack

### Next.js + Supabase

| Category | Checks |
|---|---|
| Supabase RLS | Missing policies, overly permissive rules, RLS bypass via `service_role`. Fallback to `supabase db dump` when no SQL migration files found |
| Key exposure | `service_role` assigned to `NEXT_PUBLIC_*`, hardcoded secrets in client code |
| Auth / authz | Missing auth check in Server Actions, bypassable middleware paths, unauthenticated API routes |
| Server Action input validation | Server Actions are public endpoints — check `zod`-style validation |
| XSS | `dangerouslySetInnerHTML`, `javascript:` URL scheme injection, unsanitized markdown rendering |
| Data access | Supabase admin client used from CSR, over-exposed columns |
| API security | Missing rate limit, wildcard CORS, missing input validation in API routes |
| Security headers | `next.config.js` missing CSP, X-Frame-Options, Strict-Transport-Security |
| Open redirect | User input passed directly to `redirect()` / `router.push()` |
| SSR/CSR boundary | Sensitive server-only data flowing into Client Components |
| Storage | Public Supabase Storage buckets, missing upload type/size validation |
| Dependencies | `npm audit` CVEs; fallback to `package-lock.json` direct analysis |

### Rust package

| Category | Checks |
|---|---|
| unsafe | Unnecessary `unsafe` blocks, missing bounds checks inside `unsafe`, `transmute` misuse |
| Memory safety | Raw pointer bounds, lifetime tracking inside `unsafe` |
| FFI | Missing null checks at C FFI boundary, missing string encoding validation |
| Command injection | User input reaching `std::process::Command` |
| Panic | `unwrap()` / `panic!()` in library code (unwinding propagates to callers) |
| Integer overflow | Wraparound in release builds, missing `checked_*` ops |
| ReDoS | Regex compiled from user input, vulnerable patterns |
| TOCTOU | Filesystem race: `Path::exists()` then `File::open()` |
| Supply chain | Network calls or filesystem mutation in `build.rs`, suspicious proc macros |
| Dependencies | `cargo audit` CVEs; fallback to `Cargo.lock` direct analysis; least-privilege feature flags |
| Crypto | Custom crypto, weak hashes (MD5, SHA1) |

### Flutter / Dart

| Category | Checks |
|---|---|
| Secrets | Hardcoded API keys, tokens in source / asset files |
| Platform channels | Unvalidated input crossing Dart ↔ native boundary |
| Deep links | Unvalidated URI parameters from `uni_links` / `go_router` |
| Storage | Sensitive data in `SharedPreferences`, unencrypted local DB |
| Network | Missing cert pinning for sensitive traffic, HTTP for sensitive endpoints |
| Dependencies | `pub.dev` packages with known CVEs |

### Shared

| Category | Checks |
|---|---|
| Secrets | `.env` committed, hardcoded API keys / tokens / passwords |
| Dependencies | Known CVEs (lock-file fallback when external tools unavailable) |
| Git | `.gitignore` missing sensitive files |

## False-positive suppression

Every skill recognizes the following and does not re-report suppressed items:

```
# Inline suppression
const key = process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY; // secure-scan-ignore: anon key is public by design

# Project-root suppression file (.secure-scan-ignore)
src/lib/supabase-admin.ts  # server-only module, confirmed not in client bundle
tests/fixtures/**          # intentionally vulnerable test code
```

## Output format (defined in `CLAUDE.md`)

```
# jss-<skill-name> Results

## Critical — Immediate action required
- `src/app/api/route.ts:42` — service_role key exposed to client
  Impact: attacker bypasses RLS and reads the entire DB
  Fix: remove NEXT_PUBLIC_ prefix, move to a server-only module

## Warning — Review required
- `src/middleware.ts:15` — /api/admin/* not included in middleware matcher
  Impact: admin API reachable without auth
  Fix: add pattern to matcher

## Info — Recommendations
- `package.json` — lodash 4.17.20 has CVE-2021-23337
  Fix: upgrade to >= 4.17.21

## Suppressed (N items)
```

Each finding must include **file:line**, **Impact**, and **Fix**. If nothing is found, emit "No issues found".

## File structure

```
just-agents/
├── skills/
│   ├── jss-secrets/
│   ├── jss-deps/
│   ├── jss-nextjs/
│   ├── jss-rls/
│   ├── jss-auth/
│   ├── jss-rust/
│   ├── jss-unsafe/
│   ├── jss-ffi/
│   └── jss-flutter/
├── agents/
│   └── security-scanner.md   # orchestrator
├── CLAUDE.md                 # Security Scan common principles
└── README.md
```

## Phases

### Phase 1 — MVP ✅
- `CLAUDE.md` common principles
- `jss-secrets` (highest-value, stack-agnostic)
- `jss-nextjs` (RLS, key exposure, auth, Server Actions)
- `jss-rust` (unsafe, panic, command injection, supply chain)

### Phase 2 — Detail skills ✅
- `jss-rls`, `jss-auth`, `jss-unsafe`, `jss-ffi`, `jss-deps`, `jss-flutter`

### Phase 3 — Validation and tuning
- Test fixture projects (`test-fixtures/vulnerable-nextjs/`, `test-fixtures/vulnerable-rust/`)
- Detection-rate / precision measurement against fixtures
- Prompt tuning to reduce false positives
- Auto-fix suggestions based on findings

## Limitations

- **Static analysis only**: runtime config (Vercel CORS, Supabase Dashboard settings) cannot be inferred from code alone. Reports note this limitation.
- **External tool optional**: `npm audit`, `cargo audit`, `supabase` CLI may be missing. Each skill falls back to direct lock-file analysis.
- **Large repos**: when hundreds of files are in scope, the skill follows its discovery order and notes "N of M files scanned".

---

# Code Review (`/jcr-*`)

Language- and framework-agnostic code quality review skills. Focus on quality, not security or bug detection.

## Skills

### `/jcr-review` — quality review

Reviews the target comprehensively across 8 perspectives. Each perspective has its own reference file under `skills/jcr-review/references/` so the criteria stay consistent.

**Targets**
- `git diff` (default, unstaged)
- `git diff --staged`
- `git diff main...HEAD` (full branch diff)
- File or folder path
- Arg-based: `/jcr-review src/utils.ts`, `/jcr-review --staged`

**Review perspectives**
- Unused variables, unused imports, dead code
- Comment quality (inaccurate, unnecessary, missing)
- Naming (variables, functions, classes)
- Duplication / reusable patterns
- Function complexity / responsibility split
- Magic numbers / hardcoded values
- Error handling (empty catch, swallowed errors, over-broad exceptions)
- Style consistency

**Excluded targets**
- Generated code (`*.generated.*`, `*.g.*`)
- vendor / node_modules / lock files
- Test fixtures / snapshots

**Project config**: reads `.jcr.md` at the project root for team conventions (base branch, naming rules, ignore patterns, style).

### `/jcr-refactor` — refactoring proposals

Goes beyond review to propose concrete refactors with before/after code. Shares the `references/` files with `jcr-review` for consistent criteria.

- Extractable functions / components
- Logic that can be simplified
- Pattern-application opportunities
- Includes actual refactored code snippets

## File structure

```
skills/
├── jcr-review/
│   ├── SKILL.md
│   └── references/
│       ├── dead-code.md
│       ├── comments.md
│       ├── naming.md
│       ├── duplication.md
│       ├── complexity.md
│       ├── magic-values.md
│       ├── error-handling.md
│       ├── style.md
│       └── flutter-dart.md
└── jcr-refactor/
    └── SKILL.md
```

## Reference file shape

Each reference file follows:

```markdown
# [Perspective]

## Why it matters
## Principles
## Checklist
## Good / bad examples
## Anti-patterns
```

## `.jcr.md` example

```markdown
# Project review config

## base-branch
- main

## Project
- Next.js 15 (App Router) + TypeScript (strict)
- Tailwind + shadcn/ui
- pnpm workspace monorepo

## Naming
- variables/functions: camelCase
- components: PascalCase
- constants: UPPER_SNAKE_CASE

## Ignore
- `src/components/ui/` (shadcn generated)
- `src/generated/`
- magic numbers in `*.test.ts`

## Style
- double quotes, semicolons
- import order: external → @/ → relative

## Extra rules
- JSDoc required on all public functions
```

## Output format

```
## Review summary
- Total 7 (error: 2, warning: 3, info: 2)
- Focus areas: naming (3), dead code (2), error handling (2)

## Findings

### Dead code
- [error] `src/utils.ts:42` — `tempVar` declared but never used
  → remove

### Naming
- [warning] `src/api/handler.ts:15` — `d` is unclear
  → rename to `duration` or `delay`

### Duplication
- [info] `src/components/Card.tsx:30-45` and `src/components/Panel.tsx:20-35`
  → extract a shared layout component
```

When nothing is found:

```
## Review summary

No notable quality issues found in the reviewed code.
```

Output language matches the user's language.

## Phases

### Phase 1 — core skill + references ✅
- `jcr-review/SKILL.md` with meta rules + references guide
- All 8 reference files
- `jcr-refactor/SKILL.md`

### Phase 2 — field application and refinement ✅
- Applied to real project (just-apps-homepage); 8 reference files reinforced from actual cases
- Anti-pattern catalog built from real experience
- `.jcr.md` validation: canonical section list, lookup locations, review procedure ordering
- Description tuning (`jcr-refactor`: "코드 개선" → "코드 구조 개선" to avoid trigger collision)

### Phase 3 — distribution ✅
- README written
- Install guide (direct copy / global / symlink)

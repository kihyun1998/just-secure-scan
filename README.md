# just-agents

Claude Code agents and skills across multiple domains — security scanning and code review. Each skill is self-contained and runs as a `/slash-command`; each agent orchestrates related skills or performs its own focused analysis.

## Contents

### Agents (`agents/`)

| Agent | Purpose |
|---|---|
| `security-scanner` | Detects project stack and runs the right `/jss-*` skills. Produces a unified, deduplicated security report. |
| `code-reviewer` | Code quality reviewer. Uses `/jcr-review` and `/jcr-refactor`. Reviews git diff, files, or folders. |
| `dart-logging-audit` | Dart/Flutter try-catch coverage and logging auditor. Flags missing logs, Korean log messages, and isolate-context misuse. |

### Skills (`skills/`)

**Security Scan — `/jss-*`**

| Skill | Scope |
|---|---|
| `jss-secrets` | Hardcoded secrets, API keys, credentials (stack-agnostic) |
| `jss-deps` | Dependency CVEs (npm / cargo / pub) |
| `jss-nextjs` | Comprehensive Next.js + Supabase |
| `jss-rls` | Supabase Row Level Security policies |
| `jss-auth` | Next.js + Supabase auth / authorization |
| `jss-rust` | Comprehensive Rust crate |
| `jss-unsafe` | Focused Rust `unsafe` block scan |
| `jss-ffi` | Rust FFI boundary |
| `jss-flutter` | Comprehensive Flutter / Dart |

**Code Review — `/jcr-*`**

| Skill | Scope |
|---|---|
| `jcr-review` | Quality review: dead code, naming, complexity, duplication, magic values, error handling, style |
| `jcr-refactor` | Refactoring proposals with before/after code |

## Install

### Per-project

```bash
mkdir -p .claude/skills .claude/agents
cp -r path/to/just-agents/skills/* .claude/skills/
cp -r path/to/just-agents/agents/* .claude/agents/
```

Merge `CLAUDE.md` into your project's `CLAUDE.md` (do not overwrite):

```bash
cat path/to/just-agents/CLAUDE.md >> CLAUDE.md
```

### Global

```bash
mkdir -p ~/.claude/skills ~/.claude/agents
cp -r path/to/just-agents/skills/* ~/.claude/skills/
cp -r path/to/just-agents/agents/* ~/.claude/agents/
```

### Symlink (recommended for development)

```bash
git clone https://github.com/<your-username>/just-agents.git ~/.just-agents

mkdir -p ~/.claude/skills ~/.claude/agents
ln -s ~/.just-agents/skills/*     ~/.claude/skills/
ln -s ~/.just-agents/agents/*.md  ~/.claude/agents/
```

## Project-level config

| File | Purpose |
|---|---|
| `.secure-scan-ignore` | Security scan suppression (glob patterns, relative to project root) |
| `.jcr.md` | Code review conventions (base branch, naming, ignore patterns, style) |

Inline suppression for `/jss-*` findings:

```ts
const key = process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY; // secure-scan-ignore: anon key is public by design
```

See `CLAUDE.md` for the full suppression spec.

## Usage examples

```
# Security
/jss-secrets              # stack-agnostic secret scan
/jss-nextjs               # full Next.js + Supabase scan
/jss-rust src/core/       # scope a Rust scan
/jss-rls                  # RLS-only check

# Orchestrated (auto-detects stack)
@security-scanner

# Review
/jcr-review               # git diff
/jcr-review --staged      # staged changes
/jcr-review --branch      # diff vs base branch
/jcr-refactor src/views/  # refactor proposals for a folder

# Dart/Flutter logging audit
@dart-logging-audit lib/features/ssh/
```

## Design principles

- **Self-contained skills.** Slash-command skills can't inherit from each other, so each skill embeds its own discovery strategy, checklist, and output format.
- **Common principles in `CLAUDE.md`.** Output format, severity levels, and suppression rules are defined once per domain and referenced by all skills in that domain.
- **Token efficiency.** Prompts are checklists and directives, not prose.
- **Scoped arguments.** Every skill accepts `$ARGUMENTS` to narrow scope (e.g. `/jss-nextjs src/app/api/`).

## License

MIT

---
name: jss-secrets
description: Scan for hardcoded secrets, API keys, and credential exposure (stack-agnostic)
argument-hint: <optional: path to scan>
allowed-tools: Read Grep Glob Bash
---

# jss-secrets

Scans for hardcoded secrets, API keys, tokens, and passwords exposed in project source code.

## Scan Scope

If `$ARGUMENTS` is provided, scan only that path. Otherwise scan the entire project.
Always exclude build artifacts: `node_modules/`, `target/`, `.next/`, `dist/`, `build/`, `vendor/`.

## Discovery Order

1. Read `.gitignore` to check if `.env*` files are ignored. If not — Critical.
2. If git is available, run `git ls-files` to find committed `.env` files. If git is unavailable, fallback to `find . -name '.env*' -not -path '*/node_modules/*'`.
3. If committed `.env` files exist, read their contents to check for actual secret values (report masked).
4. Search source code for the patterns below.
5. If git is available, run `git log -p --all -S 'AKIA' -S 'sk_live' -S 'sk-ant' -S 'ghp_' --diff-filter=D -- '*.ts' '*.js' '*.py' '*.rs' | head -50` to check for deleted secrets in git history.

## Detection Patterns

Each pattern is a regex that can be directly used with the Grep tool. Multiple patterns are combined with `|` to reduce the number of searches.

### Service-Specific Key Patterns → Critical

Grep-ready regexes:

- AWS Access Key: `AKIA[0-9A-Z]{16}`
- AWS Secret Key: `aws_secret_access_key\s*[:=]\s*["'][0-9a-zA-Z/+=]{40}["']`
- GitHub PAT: `ghp_[a-zA-Z0-9]{36}|github_pat_[a-zA-Z0-9]{22}_[a-zA-Z0-9]{59}`
- GitHub OAuth: `gho_[a-zA-Z0-9]{36}`
- Google/GCP API Key: `AIza[0-9A-Za-z\-_]{35}`
- Google OAuth Secret: `GOCSPX-[a-zA-Z0-9_\-]{28}`
- Stripe Live: `sk_live_[0-9a-zA-Z]{24,}|rk_live_[0-9a-zA-Z]{24,}`
- OpenAI: `sk-[a-zA-Z0-9]{48}|sk-proj-[a-zA-Z0-9\-_]{80,}`
- Anthropic: `sk-ant-[a-zA-Z0-9\-_]{80,}`
- Slack: `xoxb-[0-9]{10,}-[0-9]{10,}-[a-zA-Z0-9]{24}|xoxp-[0-9]{10,}-`
- SendGrid: `SG\.[a-zA-Z0-9_\-]{22}\.[a-zA-Z0-9_\-]{43}`
- Twilio API Key: `SK[0-9a-fA-F]{32}`
- npm token: `npm_[a-zA-Z0-9]{36}`
- Vercel: `vercel_[a-zA-Z0-9]{24}`
- Private key header: `-----BEGIN (RSA |EC |OPENSSH )?PRIVATE KEY-----`

### General Secret Patterns → Warning (needs verification)

These patterns have false positive potential — report as Warning. Skip if the value is a placeholder (`example`, `test`, `TODO`, `xxx`, `changeme`).

- `(client_secret|auth_secret|jwt_secret|database_url|connection_string)\s*[:=]\s*["'][^"']{8,}["']`
- `(password|passwd|pwd)\s*[:=]\s*["'][^"']{8,}["']` (exclude masked patterns like `password.*=.*["']\*+["']`)
- `(api[_-]?key|apikey)\s*[:=]\s*["'][a-zA-Z0-9_\-]{20,}["']` (ignore values under 20 chars to reduce false positives)
- `(secret_?key|access_?token|auth_?token)\s*[:=]\s*["'][^"']{8,}["']`

Do not match `token` alone — too many non-secret contexts (CSRF token, pagination token, etc.).

### Environment Variable Exposure → Warning

- Next.js: `NEXT_PUBLIC_` prefixed env vars containing `SECRET`, `SERVICE_ROLE`, `PRIVATE`, `PASSWORD`
- `.env.local`, `.env.production`, etc. not in `.gitignore`
- `process.env.SECRET_*` referenced directly in client components (`"use client"`)

### Config/Auth File Exposure → Warning

Check if tracked by git (`git ls-files`):

- `firebase-adminsdk*.json`
- `credentials.json`, `service-account.json`
- `*.pem`, `*.key` files
- `.npmrc`, `.pypirc`, `.docker/config.json` (package manager auth)
- `*.tfstate` (Terraform state — may contain plaintext secrets)
- `docker-compose*.yml` with hardcoded `*_PASSWORD`, `*_SECRET` values

### CI/CD Config Files → Warning

- `.github/workflows/*.yml`, `.gitlab-ci.yml`, `Jenkinsfile` with secrets hardcoded in `env:` blocks
- Plaintext values used instead of `${{ secrets.* }}`

## Suppression

- Lines with inline `secure-scan-ignore:` comments are skipped (see CLAUDE.md).
- Paths listed in `.secure-scan-ignore` are skipped.
- Secret patterns in test files (`*.test.*`, `*.spec.*`, `__tests__/`, `tests/`) are **downgraded to Info** (not suppressed). However, production key patterns (`sk_live_`, `AKIA`) remain Critical even in test files.

## Output

Follow the common output format in CLAUDE.md, including the inline axis line (Confidence / Blast / Layer) under every finding.
All discovered secret values MUST be masked (first 4 chars + `****` + last 4 chars).

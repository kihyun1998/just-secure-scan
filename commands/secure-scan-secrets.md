---
description: 시크릿/크리덴셜 노출 점검 (스택 무관)
argument-hint: <선택: 스캔 대상 경로>
---

# secure-scan-secrets

프로젝트에서 시크릿, API 키, 토큰, 비밀번호가 코드에 하드코딩되거나 노출된 부분을 찾는다.

## 스캔 범위

`$ARGUMENTS`가 있으면 해당 경로만, 없으면 프로젝트 전체를 점검한다.
빌드 아티팩트(`node_modules/`, `target/`, `.next/`, `dist/`, `build/`, `vendor/`)는 항상 제외한다.

## 탐색 순서

1. `.gitignore`를 읽어 `.env*` 파일이 ignore 대상인지 확인. 누락 시 Critical.
2. git이 있으면 `git ls-files`로 커밋된 `.env` 파일 확인. git이 없으면 `find . -name '.env*' -not -path '*/node_modules/*'`로 fallback.
3. git에 커밋된 `.env` 파일이 있으면, 해당 파일 내용도 읽어서 실제 시크릿 값을 확인한다 (마스킹하여 보고).
4. 아래 패턴을 소스 코드에서 탐색한다.
5. git이 있으면 `git log -p --all -S 'AKIA' -S 'sk_live' -S 'sk-ant' -S 'ghp_' --diff-filter=D -- '*.ts' '*.js' '*.py' '*.rs' | head -50`으로 삭제된 시크릿 히스토리도 확인한다.

## 탐지 패턴

각 패턴은 Grep 도구로 직접 실행 가능한 정규식이다. 여러 패턴은 `|`로 OR 결합하여 실행 횟수를 줄인다.

### 서비스별 키 패턴 → Critical

Grep에 직접 넣을 수 있는 정규식:

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

### 일반 시크릿 패턴 → Warning (확인 필요)

아래 패턴은 오탐 가능성이 있으므로 Warning으로 보고. 값이 placeholder(`example`, `test`, `TODO`, `xxx`, `changeme`)인 경우 건너뛴다.

- `(client_secret|auth_secret|jwt_secret|database_url|connection_string)\s*[:=]\s*["'][^"']{8,}["']`
- `(password|passwd|pwd)\s*[:=]\s*["'][^"']{8,}["']` (단, `password.*=.*["']\*+["']` 마스킹 패턴 제외)
- `(api[_-]?key|apikey)\s*[:=]\s*["'][a-zA-Z0-9_\-]{20,}["']` (20자 미만은 무시하여 오탐 감소)
- `(secret_?key|access_?token|auth_?token)\s*[:=]\s*["'][^"']{8,}["']`

`token`만 단독으로 매칭하지 않는다 — CSRF token, pagination token 등 비시크릿 맥락이 너무 많다.

### 환경변수 노출 → Warning

- Next.js: `NEXT_PUBLIC_` 접두사 환경변수 중 `SECRET`, `SERVICE_ROLE`, `PRIVATE`, `PASSWORD` 키워드 포함
- `.env.local`, `.env.production` 등이 `.gitignore`에 없음
- `process.env.SECRET_*`을 클라이언트 컴포넌트(`"use client"`)에서 직접 참조

### 설정/인증 파일 노출 → Warning

git 추적 중인지 확인 (`git ls-files`):

- `firebase-adminsdk*.json`
- `credentials.json`, `service-account.json`
- `*.pem`, `*.key` 파일
- `.npmrc`, `.pypirc`, `.docker/config.json` (패키지 매니저 인증)
- `*.tfstate` (Terraform state — 평문 시크릿 포함 가능)
- `docker-compose*.yml` 내 하드코딩된 `*_PASSWORD`, `*_SECRET` 값

### CI/CD 설정 파일 → Warning

- `.github/workflows/*.yml`, `.gitlab-ci.yml`, `Jenkinsfile`에서 `env:` 블록에 시크릿 값이 직접 하드코딩된 경우
- `${{ secrets.* }}`를 사용하지 않고 평문 값을 쓴 경우

## 억제

- 인라인 `secure-scan-ignore:` 주석이 달린 줄은 건너뛴다 (CLAUDE.md 참조).
- `.secure-scan-ignore`에 등록된 경로는 건너뛴다.
- 테스트 파일(`*.test.*`, `*.spec.*`, `__tests__/`, `tests/`)의 시크릿 패턴은 **등급을 Info로 하향**한다 (억제가 아님). 단, `sk_live_`, `AKIA` 등 프로덕션 키 패턴은 테스트 파일이라도 Critical을 유지한다.

## 출력

CLAUDE.md의 공통 출력 포맷을 따른다.
발견된 시크릿 값은 반드시 마스킹한다 (앞 4자 + `****` + 뒤 4자).

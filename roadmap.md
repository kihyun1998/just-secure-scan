# just-secure-scan Roadmap

## Overview

Claude Code 전용 코드 보안 점검 skill 모음.
`.claude/commands/` markdown 파일로 구성하며, 각 커맨드는 자기 완결적(self-contained)으로 동작한다.
공통 원칙(출력 포맷, 심각도 기준)은 `CLAUDE.md`에 정의하여 모든 커맨드가 자동으로 참조하게 한다.

## 설계 원칙

- **커맨드 독립성**: 커맨드 간 참조/상속이 불가하므로, 각 커맨드는 단독 실행 가능해야 한다.
- **CLAUDE.md = 공통 원칙**: 출력 포맷, 심각도 기준, false positive 억제 규칙 등 모든 커맨드에 적용되는 원칙은 `CLAUDE.md`에 둔다.
- **토큰 효율**: 프롬프트는 서술형이 아닌 체크리스트/지시형으로 작성하여 context 소비를 최소화한다.
- **인자 지원**: `$ARGUMENTS`로 스캔 범위를 지정할 수 있다. (예: `/secure-scan-nextjs src/app/`)
- **탐색 전략 내장**: 각 커맨드에 "어떤 파일을 우선 탐색할지" 구체적 패턴을 명시한다.

## 스택별 점검 항목

### Next.js + Supabase

| 카테고리 | 점검 항목 |
|----------|-----------|
| Supabase RLS | RLS 정책 누락, 과도한 permissive 정책, `service_role` key로 RLS 우회. SQL 마이그레이션 파일 없으면 `supabase db dump` fallback |
| 키 노출 | `NEXT_PUBLIC_`에 `service_role` key 할당, 클라이언트 코드에 시크릿 하드코딩 |
| 인증/인가 | Server Action에서 인증 검증 누락, Middleware 우회 가능 경로, API Route 인증 누락 |
| Server Action 입력 검증 | Server Action은 공개 API endpoint — `zod` 등으로 입력 검증 여부 확인 |
| XSS | `dangerouslySetInnerHTML` 사용, URL scheme injection (`javascript:`), 마크다운 렌더링 시 sanitize 누락 |
| 데이터 접근 | CSR에서 Supabase admin client 직접 사용, 불필요한 컬럼 노출 |
| API 보안 | Rate limit 미적용, CORS 와일드카드, API Route에서 입력 검증 누락 |
| 보안 헤더 | `next.config.js`에 CSP, X-Frame-Options, Strict-Transport-Security 미설정 |
| Open Redirect | `redirect()` 또는 `router.push()`에 사용자 입력 직접 전달 |
| SSR/CSR 경계 | Server Component에서 민감 데이터가 Client Component로 전달 |
| Storage | Supabase Storage 공개 버킷, 파일 업로드 타입/사이즈 검증 미비 |
| 의존성 | `npm audit`으로 알려진 CVE 점검. 미설치 시 `package-lock.json` 직접 분석 |

### Rust Package

| 카테고리 | 점검 항목 |
|----------|-----------|
| unsafe | 불필요한 `unsafe` 블록, unsafe 내 경계 검증 누락, `transmute` 오용 |
| 메모리 안전 | raw pointer 경계 검증, unsafe 블록 내 lifetime 추적 |
| FFI | C FFI 경계에서 null 체크 누락, 문자열 인코딩 검증 미비 |
| Command Injection | `std::process::Command`에 사용자 입력 직접 전달 |
| 패닉 | 라이브러리에서 `unwrap()`/`panic!()` 사용 (호출자에게 unwinding 전파) |
| Integer Overflow | release 빌드에서 산술 오버플로우 wrap around, `checked_*` 미사용 |
| ReDoS | 사용자 입력으로 정규식 컴파일, 취약한 정규식 패턴 |
| TOCTOU | 파일시스템 작업에서 `Path::exists()` 후 `File::open()` race condition |
| Supply Chain | `build.rs`에서 네트워크 호출/파일시스템 조작, 의심스러운 proc macro |
| 의존성 | `cargo audit`으로 알려진 CVE 점검. 미설치 시 `Cargo.lock` 직접 분석, 최소 권한 feature flag |
| Crypto | 자체 구현 암호화, 약한 해시 알고리즘(MD5, SHA1) 사용 |

### 공통

| 카테고리 | 점검 항목 |
|----------|-----------|
| 시크릿 | `.env` 파일 커밋 여부, 하드코딩된 API 키/토큰/비밀번호 |
| 의존성 | 알려진 CVE (외부 도구 미설치 시 lock 파일 직접 분석으로 fallback) |
| Git | `.gitignore`에 민감 파일 누락 |

## False Positive 억제

모든 커맨드는 다음 메커니즘을 인식하여 이미 확인된 항목을 반복 보고하지 않는다:

```
# 코드 내 인라인 억제
let key = process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY; // secure-scan-ignore: anon key는 공개용

# 프로젝트 루트 억제 파일
# .secure-scan-ignore
src/lib/supabase-admin.ts:service_role  # 서버 전용 모듈, 클라이언트 번들에 미포함 확인됨
```

## 파일 구조 (이 저장소)

이 저장소는 skill 소스를 관리하는 배포 저장소다. 파일은 루트에 배치한다.

```
just-secure-scan/
├── commands/
│   ├── secure-scan-nextjs.md     # Next.js+Supabase 전체 점검
│   ├── secure-scan-rust.md       # Rust package 전체 점검
│   ├── secure-scan-secrets.md    # 시크릿 노출 점검 (공통)
│   ├── secure-scan-rls.md        # Supabase RLS 점검 (세부)
│   ├── secure-scan-auth.md       # Next.js 인증/인가 점검 (세부)
│   ├── secure-scan-unsafe.md     # Rust unsafe 점검 (세부)
│   ├── secure-scan-deps.md       # 의존성 CVE 점검 (공통)
│   └── secure-scan-ffi.md        # Rust FFI 경계 점검 (세부)
├── CLAUDE.md                     # 공통 원칙 (출력 포맷, 심각도 기준, 억제 규칙, 탐색 전략)
├── roadmap.md                    # 이 기획서
└── README.md                     # 설치/사용 안내
```

사용자 프로젝트에 설치하면 아래 구조가 된다:

```
<사용자 프로젝트>/
├── .claude/
│   └── commands/
│       ├── secure-scan-nextjs.md
│       ├── secure-scan-rust.md
│       └── ...
├── CLAUDE.md                     # 기존 내용 + just-secure-scan 공통 원칙 병합
└── .secure-scan-ignore           # (선택) false positive 억제 파일
```

## 파일 탐색 전략

각 커맨드는 전체 파일을 무작정 읽지 않고, 우선 탐색 패턴을 따른다:

**Next.js + Supabase:**
1. `grep -r "createClient\|createServerClient\|createBrowserClient"` → Supabase 클라이언트 사용처
2. `app/**/route.ts`, `app/**/action.ts` → API Route, Server Action
3. `middleware.ts` → 인증 경로 매칭
4. `.env*`, `next.config.*` → 환경변수, 보안 헤더
5. `supabase/migrations/*.sql` → RLS 정책

**Rust:**
1. `grep -r "unsafe"` → unsafe 블록 위치
2. `grep -r "Command::new\|process::Command"` → 커맨드 인젝션
3. `build.rs` → supply chain 리스크
4. `src/lib.rs`, `src/main.rs` → 진입점에서 panic 패턴
5. `Cargo.toml`, `Cargo.lock` → 의존성, feature flag

## 출력 포맷 (모든 커맨드 공통, CLAUDE.md에 정의)

```
## Critical — 즉시 수정 필요
- `src/app/api/route.ts:42` — service_role key가 클라이언트에 노출됨
  **영향**: 공격자가 RLS를 우회하여 전체 DB 접근 가능
  **수정**: NEXT_PUBLIC_ 접두사 제거, server-only 모듈로 이동

## Warning — 검토 필요
- `src/middleware.ts:15` — /api/admin/* 경로가 middleware matcher에 미포함
  **영향**: 인증 없이 admin API 접근 가능
  **수정**: matcher 패턴에 해당 경로 추가

## Info — 권장사항
- `package.json` — lodash 4.17.20에 CVE-2021-23337 존재
  **수정**: 4.17.21 이상으로 업데이트

## Suppressed — 억제된 항목 (N건)
> .secure-scan-ignore 또는 인라인 주석으로 억제됨. 상세 내용은 생략.
```

각 항목은 반드시 **파일경로:라인**, **영향**, **수정 방안**을 포함한다.
발견 항목이 없으면 "발견된 이슈 없음"을 명시한다.

## 사용 예시

```bash
# Next.js+Supabase 프로젝트 전체 점검
/secure-scan-nextjs

# 특정 디렉토리만 점검
/secure-scan-nextjs src/app/api/

# RLS 정책만 빠르게 확인
/secure-scan-rls

# Rust crate 전체 점검
/secure-scan-rust

# 시크릿 노출만 점검 (스택 무관)
/secure-scan-secrets

# 의존성 CVE만 점검 (스택 무관)
/secure-scan-deps
```

## 배포 방법

이 저장소의 `commands/` 파일을 대상 프로젝트의 `.claude/commands/`에 복사하여 사용한다.

```bash
# 수동 설치
mkdir -p <target-project>/.claude/commands
cp commands/secure-scan-*.md <target-project>/.claude/commands/

# CLAUDE.md는 기존 파일에 병합 (덮어쓰기 금지)
cat CLAUDE.md >> <target-project>/CLAUDE.md
```

> 향후 `npx just-secure-scan init` 같은 설치 스크립트를 고려할 수 있으나, MVP에서는 수동 복사로 충분하다.

## 구현 순서

### Phase 1 — MVP

핵심 점검 항목만 포함한 최소 동작 버전.

1. `CLAUDE.md` — 공통 원칙 (출력 포맷, 심각도 기준, 억제 규칙)
2. `secure-scan-secrets.md` — 시크릿 노출 점검 (가장 쉽고 가치 높음)
3. `secure-scan-nextjs.md` — Next.js+Supabase 점검 (핵심 항목: RLS, 키 노출, 인증, Server Action)
4. `secure-scan-rust.md` — Rust package 점검 (핵심 항목: unsafe, 패닉, Command Injection, supply chain)

### Phase 2 — 세부 커맨드

전체 점검에서 자주 단독으로 쓰이는 항목을 분리.

5. `secure-scan-rls.md`
6. `secure-scan-auth.md`
7. `secure-scan-unsafe.md`
8. `secure-scan-deps.md`
9. `secure-scan-ffi.md`

### Phase 3 — 검증 및 개선

10. 테스트 fixture 프로젝트 작성 (`test-fixtures/vulnerable-nextjs/`, `test-fixtures/vulnerable-rust/`)
11. fixture 기반 탐지율/정확도 측정
12. false positive 줄이기 위한 프롬프트 튜닝
13. 점검 결과 기반 자동 수정 제안 기능

## 한계 및 주의사항

- **정적 분석 한계**: 런타임 동작(Vercel CORS, Supabase Dashboard 설정 등)은 코드만으로 판단 불가. 출력에 "코드 기반 점검" 한계를 명시한다.
- **외부 도구 의존**: `npm audit`, `cargo audit`, `supabase` CLI가 없을 수 있다. 각 커맨드는 미설치 시 lock 파일 직접 분석으로 fallback한다.
- **대형 프로젝트**: 파일이 수백 개일 경우 탐색 전략에 따라 우선순위 파일만 점검하고, 출력에 "전체 파일 중 N개 점검됨"을 명시한다.

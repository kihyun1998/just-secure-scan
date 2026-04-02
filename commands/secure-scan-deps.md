---
description: 의존성 CVE 및 보안 점검 (npm/cargo)
argument-hint: <선택: npm 또는 cargo>
---

# secure-scan-deps

프로젝트 의존성의 알려진 취약점(CVE)과 보안 설정을 점검한다.
현재 npm(Node.js)과 cargo(Rust) 생태계를 지원한다.

## 스택 감지

`$ARGUMENTS`가 `npm` 또는 `cargo`이면 해당 생태계만 점검한다.
인자가 없으면 `package.json`과 `Cargo.toml` 존재 여부로 자동 감지한다.

## npm 생태계

### 1. npm audit (Critical ~ Info)

```bash
npm audit --json 2>/dev/null | head -100
```

실행 가능하면 결과를 파싱하여 심각도별로 분류:
- `critical`, `high` → Critical
- `moderate` → Warning
- `low` → Info

미설치 또는 실행 실패 시 `package-lock.json`을 직접 읽어 주요 패키지 버전을 확인한다. Claude의 학습 데이터 이후 공개된 CVE는 탐지 불가함을 명시한다. 하드코딩된 CVE 목록에 의존하지 않고, 외부 도구 결과를 우선한다.

### 2. 보안 관련 설정 (Info)

- `package.json`에 `engines` 필드 없음 — Node.js 버전 미제한
- `overrides`/`resolutions`로 취약 패키지를 고정한 흔적 확인

## Cargo 생태계

### 1. cargo audit (Critical ~ Info)

```bash
cargo audit --json 2>/dev/null | head -100
```

실행 가능하면 결과를 파싱하여 심각도별 분류. 미설치 시 `Cargo.lock`을 직접 읽어 주요 패키지 버전 확인. CVE 커트오프 한계 명시. 하드코딩된 CVE 목록에 의존하지 않고, 외부 도구 결과를 우선한다.

### 2. Feature flag 보안 (Info)

- `Cargo.toml`에서 `default-features = false`를 쓰지 않고 불필요한 feature가 활성화된 의존성
- `features`에 `full`이나 과도하게 넓은 feature set 사용

### 3. build.rs 의존성 (Warning)

- `[build-dependencies]`에 네트워크 기능이 있는 crate (`reqwest`, `ureq`, `curl`)

## 출력

CLAUDE.md의 공통 출력 포맷을 따른다.
의존성 요약표를 추가로 출력한다:

```
## 의존성 요약
- 총 의존성: N개
- 취약점: Critical N개 / Warning N개 / Info N개
- 점검 방법: npm audit / cargo audit (또는 lock 파일 직접 분석)
- CVE 데이터 기준: (외부 도구 사용 시 최신, 직접 분석 시 학습 데이터 커트오프 명시)
```

## 억제

CLAUDE.md의 억제 규칙을 따른다.

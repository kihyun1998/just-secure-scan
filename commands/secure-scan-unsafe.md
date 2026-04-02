---
description: Rust unsafe 코드 점검
argument-hint: <선택: 스캔 대상 경로>
---

# secure-scan-unsafe

Rust 프로젝트의 `unsafe` 코드를 집중 점검한다. FFI 관련 상세 점검은 `/secure-scan-ffi`에서 다룬다. 이 커맨드는 FFI 내 unsafe를 간략히만 확인하고, 중복 보고를 피한다.

## 스캔 범위

`$ARGUMENTS`가 있으면 해당 경로만, 없으면 프로젝트 전체를 점검한다.
`target/`, `vendor/`는 항상 제외한다.

## 탐색 순서

1. `Cargo.toml`을 읽어 workspace 여부를 판단. workspace이면 각 member crate를 식별한다.
2. 각 crate에서 `#![forbid(unsafe_code)]` / `#![deny(unsafe_code)]` 선언 여부를 확인.
   - `forbid`: 해당 crate는 건너뛰고 Info로 "unsafe_code 금지됨" 보고.
   - `deny`: Info로 "unsafe_code가 deny이나 `#[allow]`로 국소 해제 가능" 보고. 점검은 계속.
3. Grep으로 `unsafe` 검색 (파일명만) → 대상 파일 식별
4. Grep으로 `transmute|transmute_copy` 검색
5. Grep으로 `ptr::read|ptr::write|ptr::copy|ptr::drop_in_place` 검색
6. Grep으로 `MaybeUninit` 검색
7. Grep으로 `Pin::new_unchecked|Pin::get_unchecked_mut` 검색
8. Grep으로 `union\s+\w+` 검색 → union 타입 사용

## 점검 항목

### 1. 불필요한 unsafe (Warning)

- safe 대안이 있는데 unsafe를 쓴 경우
- 예: `unsafe { std::str::from_utf8_unchecked() }` → `from_utf8()`
- 예: `unsafe { slice::from_raw_parts() }` → safe slice 연산

### 2. 경계 검증 누락 (Critical)

- unsafe 블록 내에서 포인터 연산, 슬라이스 접근 시 길이/범위 검증이 블록 진입 전에 없음
- raw pointer dereference에서 null 체크 없음
- `slice::from_raw_parts(ptr, len)`에서 `len`의 유효성 미검증

### 3. transmute 오용 (Critical)

- `std::mem::transmute`으로 타입 변환 시 크기/정렬 보장 없음
- 안전한 대안: `from_bits`, `to_ne_bytes`, `bytemuck::cast`/`bytes_of`, `zerocopy`, `TryFrom`/`TryInto`

### 4. unsafe impl (Warning)

- `unsafe impl Send for T` / `unsafe impl Sync for T` — 수동 구현 시 `// SAFETY:` 주석 없음
- 내부에 raw pointer나 `UnsafeCell`이 있는데 thread-safety 근거가 불충분

### 5. unsafe fn (Warning)

- 불필요하게 `unsafe fn`으로 선언된 함수 — safe fn + 내부 unsafe 블록으로 변경 가능
- `unsafe fn` 내부에서 `unsafe {}` 블록 없이 unsafe 연산 수행 (Rust 2024 edition에서는 `unsafe_op_in_unsafe_fn` lint 기본 활성화)

### 6. 위험 함수 사용 (Critical ~ Warning)

- `ptr::read` / `ptr::write` / `ptr::copy` / `ptr::copy_nonoverlapping` — 정렬, 유효성, overlap 요구사항 미충족 시 UB (Critical)
- `MaybeUninit::uninit().assume_init()` — 초기화되지 않은 메모리를 초기화된 것으로 취급, UB (Critical)
- `MaybeUninit::zeroed().assume_init()` — 0이 유효한 값이 아닌 타입에서 사용 시 UB (Critical)
- `Pin::new_unchecked` / `Pin::get_unchecked_mut` — Pin 계약 위반 시 UB (Warning)
- `union` 필드 접근 — 잘못된 variant 읽기는 UB (Warning)

### 7. SAFETY 주석 검토

- `// SAFETY:` 주석이 있는 unsafe 블록은 주석 내용의 타당성을 검토. 타당하면 보고하지 않음.
- 불충분한 주석 (`// SAFETY: trust me`, `// SAFETY: safe` 등) → Warning
- `// SAFETY:` 주석이 전혀 없는 unsafe 블록 → Warning

### 8. FFI 내 unsafe (간략)

- `extern "C"` fn 내 패닉 가능성 → Warning (프로세스 abort)
- null 포인터 체크 없는 역참조 → Critical
- 상세 FFI 점검은 `/secure-scan-ffi`에서 수행. 이 커맨드에서는 위 2가지만 확인.

## 통계 출력

```
## unsafe 통계 (약 N개 — Grep 기반, 주석/문자열 내 false match 포함 가능)
- unsafe 블록: ~N개 (M개 파일)
- unsafe fn: ~N개
- unsafe impl: ~N개
- unsafe trait: ~N개
- extern "C" fn: ~N개
- #![forbid(unsafe_code)] crate: N개 / 전체 M개
- #![deny(unsafe_code)] crate: N개 / 전체 M개
```

`cargo-geiger`가 설치되어 있으면 `cargo geiger --output-format ascii 2>/dev/null | head -50`으로 더 정확한 통계를 가져온다.

## 억제

CLAUDE.md의 억제 규칙을 따른다.

## 출력

CLAUDE.md의 공통 출력 포맷을 따른다.

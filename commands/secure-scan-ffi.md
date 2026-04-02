---
description: Rust FFI 경계 보안 점검
argument-hint: <선택: 스캔 대상 경로>
---

# secure-scan-ffi

Rust 프로젝트의 FFI(Foreign Function Interface) 경계 보안을 점검한다. `/secure-scan-unsafe`와 중복되는 부분(null 체크, FFI 내 패닉)은 이 커맨드에서 더 상세하게 다루며, unsafe 쪽에서는 간략 확인만 한다.

## 스캔 범위

`$ARGUMENTS`가 있으면 해당 경로만, 없으면 프로젝트 전체를 점검한다.
`target/`, `vendor/`는 항상 제외한다.

## 탐색 순서

1. `Cargo.toml`을 읽어 workspace 여부 판단. `[dependencies]`에서 FFI 관련 crate 확인 (`libc`, `bindgen`, `cc`, `cxx`, `pyo3`, `napi`, `wasm-bindgen`)
2. Grep으로 `extern\s*"C"` 검색 → FFI 함수 정의
3. Grep으로 `#\[no_mangle\]` 검색 → 외부 노출 함수
4. Grep으로 `extern\s*\{` 검색 → 외부 함수 선언 (C 라이브러리 바인딩)
5. Grep으로 `CStr|CString|c_char|c_void` 검색 → C 타입 사용처
6. Grep으로 `#\[pyfunction\]|#\[pyfn\]` 검색 → pyo3 함수
7. Grep으로 `#\[napi\]` 검색 → napi 함수
8. Grep으로 `#\[wasm_bindgen\]` 검색 → wasm-bindgen 함수

## 점검 항목

### 1. Null 포인터 (Critical)

- `extern "C"` 함수에서 `*const T` / `*mut T` 매개변수를 null 체크 없이 역참조
- `CStr::from_ptr(ptr)`을 null 체크 없이 호출
- 외부 함수 반환값이 포인터인 경우 null 체크 없이 사용

### 2. 문자열 안전성 (Warning)

- C 문자열(`*const c_char`)을 수신할 때 UTF-8 유효성 검증 없이 `to_str().unwrap()` 사용
- `CString::new()`에 null 바이트가 포함될 수 있는 입력을 검증 없이 전달
- 문자열 길이 제한 없이 C에서 전달된 문자열을 `String`으로 변환

### 3. 패닉 across FFI (Warning)

- `#[no_mangle] pub extern "C" fn` 내에서 패닉 가능성 → 프로세스 abort로 서비스 중단
- `unwrap()`, `expect()`, `panic!()`, index 접근(`[]`) 등 패닉 유발 코드
- `catch_unwind`로 감싸지 않은 경우
- 권장: `Result` 반환 + 에러 코드 변환, 또는 `catch_unwind` + 에러 반환

### 4. 메모리 소유권 (Critical ~ Warning)

점검 가능 범위: 단일 파일 내에서 확인 가능한 패턴만 점검. 다중 파일에 걸친 소유권 추적은 "수동 확인 필요"로 보고.

- `into_raw()` 호출 후 같은 파일/모듈에서 대응하는 `from_raw()` 해제 경로가 없는 경우 (Warning)
- `Box::from_raw`로 C 할당 메모리를 가져오는 패턴 — allocator 불일치 (Critical)
- `libc::free` vs `Box::drop` 혼용 (Critical)
- Rust에서 할당한 메모리를 C에 넘기면서 해제 함수(`free_*`)를 export하지 않는 경우 (Warning)

### 5. 외부 함수 바인딩 안전성 (Warning)

- `extern { fn ... }` 블록에서 선언한 함수의 시그니처가 수동 작성인 경우 — `bindgen` 사용 권장
- `#[repr(C)]` 없이 C에 전달하는 struct 또는 enum
- C 함수 호출 후 에러 코드/`errno` 확인 누락

### 6. Callback 안전성 (Warning)

- C에서 Rust 함수 포인터를 callback으로 호출할 때, 해당 함수 내에서 패닉 가능성
- callback 함수가 `extern "C"` ABI로 선언되지 않은 경우

### 7. Thread safety (Warning)

- FFI를 통해 호출하는 C 라이브러리가 thread-safe가 아닌 경우 — 래퍼에 `Send`/`Sync` 수동 구현이 있으면 근거 확인
- 전역 상태를 변경하는 C 함수를 멀티스레드에서 호출하는 패턴

### 8. 고수준 FFI 바인딩 (Warning ~ Info)

pyo3, napi, wasm-bindgen이 사용되는 경우 추가 점검:

**pyo3:**
- `#[pyfunction]` 내에서 패닉 가능성 — Python exception으로 변환되지 않으면 프로세스 crash
- `Python::allow_threads` 내에서 Python 객체 접근 (GIL 없이 접근은 UB)

**napi:**
- `#[napi]` 함수에서 패닉 가능성 — JS exception으로 변환되지 않으면 프로세스 crash
- `Buffer` 소유권 — JS에서 전달된 Buffer를 Rust에서 소유하면 GC와 충돌 가능

**wasm-bindgen:**
- `JsValue` 누수 — `forget()`으로 명시적 해제를 건너뛰는 패턴
- linear memory 접근 시 bounds check 누락

## 통계 출력

```
## FFI 통계
- extern "C" fn (노출): ~N개
- extern { fn ... } (바인딩): ~N개
- #[no_mangle]: ~N개
- FFI 관련 crate: libc, bindgen, ...
- 고수준 바인딩: pyo3 / napi / wasm-bindgen (해당 시)
```

## 억제

CLAUDE.md의 억제 규칙을 따른다.

## 출력

CLAUDE.md의 공통 출력 포맷을 따른다.

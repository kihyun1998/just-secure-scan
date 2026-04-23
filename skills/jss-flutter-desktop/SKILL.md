---
name: jss-flutter-desktop
description: Scan Flutter desktop (Windows/macOS/Linux) security issues — file, process, FFI, registry, updater
argument-hint: <optional: path to scan>
allowed-tools: Read Grep Glob Bash
---

# jss-flutter-desktop

Scans desktop-specific security issues in Flutter apps targeting Windows, macOS, or Linux. Unlike mobile Flutter, desktop has **no OS sandbox** — filesystem, processes, FFI, and registry become direct attack surface.

If `$ARGUMENTS` is provided, scan only that path. Otherwise scan the entire project.

This skill complements `/jss-flutter`, which covers mobile-oriented checks (permissions, deep links, WebView, certificate pinning). Run both on multi-platform projects; `file:line`-based dedup prevents double reporting.

## Discovery Strategy

Do not read entire files upfront. Identify target files first.
Exclude `build/`, `.dart_tool/`, `.pub-cache/` from all Grep searches.

### 0. Project Detection

- Verify `pubspec.yaml` exists with `flutter` SDK dependency.
- Verify at least one of `windows/`, `macos/`, `linux/` directories exists. If none — report "Not a Flutter desktop project" and exit.
- Note which desktop targets are present — Windows-specific checks only apply when `windows/` exists, etc.

### 1–7. Discovery Targets

1. Grep for `Process\.run|Process\.start|Process\.runSync` (file names) — process spawning
2. Grep for `DynamicLibrary\.open|DynamicLibrary\.process` (file names) — FFI native library loading
3. Grep for `MethodChannel|EventChannel|BasicMessageChannel` (file names) — platform channels
4. Glob `windows/runner/**/*.cpp`, `windows/runner/**/*.h`, `macos/Runner/**/*.swift`, `linux/**/*.cc` — native platform channel handlers
5. Grep `win32_registry|RegistryKey` in `pubspec.yaml` and source — Windows registry usage
6. Grep for `auto_updater|flutter_autoupdate|sparkle_updater|simple_updater|msix|flutter_distributor` in `pubspec.yaml` — updater packages
7. Grep for `path_provider|getApplicationSupportDirectory|getTemporaryDirectory|getApplicationDocumentsDirectory` — storage location choices

## Checks

### 1. File System — Path Traversal (Critical ~ Warning)

- **User input to `File()` / `Directory()`** (Critical): a path derived from CLI arg, IPC payload, env var, or deep-link reaches `File(path)` / `Directory(path)` without `path.normalize` + `..` rejection. Desktop has no sandbox — traversal writes outside app directory.
- **`File.rename` / `File.copy` cross-directory** (Warning): destination path computed from user input.
- **Symlink tolerance** (Warning): `File.readAsString` / `File.openRead` on user-provided path without `Link.resolveSymbolicLinks` check — symlink escape possible.
- **Path join without validation** (Warning): `path.join(basePath, userSegment)` where `userSegment` can contain `..` or absolute paths (`path.join` does NOT constrain).

### 2. Process Execution (Critical ~ Warning)

- **`Process.run` with `runInShell: true`** (Critical): `runInShell: true` + any user input in command or args → shell injection. Desktop has no sandbox; this is RCE.
- **String concatenation in command** (Critical): `Process.run('cmd $input')` or `'$input'` interpolated into executable name or single-string args.
- **PATH-based executable resolution** (Warning): `Process.run('ffmpeg', ...)` relying on `PATH` lookup. Resolves to attacker-controlled binary if `PATH` is manipulated or a local binary is planted. Prefer absolute paths for bundled binaries.
- **Windows `.bat` / `.cmd` invocation** (Warning): spawning batch files bypasses the usual arg-splitting guarantees — quoting is very different for `cmd.exe`.
- **`Process.start` fire-and-forget** (Info): spawned process never awaited — resource leak, not a security issue on its own.

### 3. Dart FFI — DLL / dylib Hijacking (Critical ~ Warning)

- **`DynamicLibrary.open('name.dll')`** (Critical on Windows): loading a library by bare name. Windows search order resolves to the executable's directory first — an attacker planting a DLL next to the exe wins. Prefer an absolute path inside the bundled app dir.
- **`DynamicLibrary.open(userInput)`** (Critical): library path derived from user input — arbitrary code execution.
- **`DynamicLibrary.process()`** (Info): loads symbols from the current process; platform-dependent behavior, generally safe but worth noting.
- **Pointer cast without bounds** (Warning): `.cast<T>()` or `.asTypedList(n)` on a pointer returned from native without verifying `n` against the actual allocation size.

### 4. Platform Channel — Native Side (Warning)

- **Native-side method handler skips arg validation** (Warning): `MethodChannel` handler in `windows/runner/*.cpp`, `macos/Runner/*.swift`, or `linux/**/*.cc` passes Dart-supplied arguments directly to native system APIs (`CreateFileA`, `ShellExecuteW`, `posix_spawn`, etc.) without validation. Cross-file / cross-language inference — mark Confidence: Medium/Low.
- **Native exceptions leaked to Dart** (Info): native code returning `error.toString()`-style detail (paths, handles) in the `FlutterError` payload — information disclosure.
- **Method name not whitelisted** (Warning): native method dispatcher using `call.method` in an open switch/chain without a "default: return NotImplemented" guard; unknown method names may hit unintended handlers.

### 5. Auto-Updater (Critical ~ Warning)

If any updater package is declared in `pubspec.yaml`:
- **HTTP endpoint** (Critical): updater URL configured with `http://`.
- **No signature verification** (Critical): updater initialized without a public-key / signature parameter. `auto_updater`, `sparkle_updater`, `simple_updater` each expose a verification hook — if it's not wired up, endpoint or MITM compromise → RCE.
- **Endpoint from runtime env** (Warning): updater URL read from `Platform.environment` or similar — attacker who can set env can redirect updates.
- **`allowPrerelease: true` in release** (Info): opt-in pre-release channel in production config — not security per se but increases attack surface.

### 6. Local Storage Location (Warning ~ Info)

- **Sensitive data in `getTemporaryDirectory()`** (Warning): tokens, keys, or session data written to `%TEMP%` / `/tmp`. On desktop, other processes (and sometimes other users on shared systems) can read these.
- **Sensitive data in `getApplicationDocumentsDirectory()` unencrypted** (Warning): user-visible path with token/key written as plain text. Backup tooling and cloud sync can propagate these off-device.
- **Correct choice missed** (Info): sensitive data should use `flutter_secure_storage` (wraps Credential Manager / Keychain / libsecret) or explicit encryption before write. Flag if the project stores credentials but does not depend on `flutter_secure_storage` or equivalent.

### 7. Windows Registry (Warning)

If `win32_registry` is in dependencies:
- **HKLM write** (Warning): `Registry.localMachine.createKey(...)` or write under `HKEY_LOCAL_MACHINE\*`. Requires admin — legitimate for installers, suspicious in a normal app runtime.
- **Unvalidated registry key path** (Warning): key name constructed from user input.
- **Credential stored as plain registry value** (Warning): token or key written as a string value; prefer Credential Manager.

### 8. Shell Integration & Protocol Handlers (Warning)

- **Protocol handler without validation** (Warning): URI scheme registered in `windows/runner/main.cpp`, macOS `Info.plist`, or Linux `.desktop` file, consumed by a Dart handler that reads the URL without structural validation. On desktop, any app can invoke `yourapp://...` URLs.
- **File association without type check** (Info): file extension associated with the app, and the Dart handler opens the file without validating extension or magic bytes (could be anything).
- **Single-instance IPC without origin check** (Warning): single-instance apps passing startup args to the running instance via named pipe / mutex / unix socket without validating the sender — any local process can inject args.

### 9. Window / UX Spoofing (Info)

- **Always-on-top + borderless** (Info): `window_manager.setAlwaysOnTop(true)` combined with `titleBarStyle: TitleBarStyle.hidden` or transparent flags — building blocks for credential phishing overlays. Low priority unless the app handles auth prompts.
- **No content protection** (Info): apps handling sensitive data with no `window_manager.setContentProtection` (Windows) / equivalent. Screen recording is trivial on desktop.

### 10. Logging on Desktop (Warning ~ Info)

- **Log files in world-readable path** (Warning): file sinks writing to `getApplicationSupportDirectory` or user home without masking secrets. Logs persist and are easy to copy.
- **`print` / `debugPrint` / `log` with secrets** (Warning): if not already reported by `/jss-flutter`, flag patterns matching `token|password|secret|key|credential` in call arguments inside non-test files.

## Suppression

Follow suppression rules in CLAUDE.md.

## Output

Follow the common output format in CLAUDE.md, including the inline axis line (Confidence / Blast / Layer) under every finding.

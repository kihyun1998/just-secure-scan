---
name: jss-flutter
description: Comprehensive Flutter/Dart security scan
argument-hint: <optional: path to scan>
allowed-tools: Read Grep Glob Bash
---

# jss-flutter

Scans security issues in Flutter/Dart projects.
If `$ARGUMENTS` is provided, scan only that path. Otherwise scan the entire project.

## Discovery Strategy

Do not read entire files upfront. Identify target files first using the patterns below.
Exclude `build/`, `.dart_tool/`, `.pub-cache/` from all Grep searches.

### 0. Project Detection

- Verify `pubspec.yaml` exists. If not, report "Not a Flutter/Dart project" and exit.
- Read `pubspec.yaml` to determine: Flutter vs pure Dart, dependencies, SDK constraints.
- Check for `analysis_options.yaml` — linter configuration.

### 1–8. Discovery Targets

1. Grep for `SharedPreferences|FlutterSecureStorage|Hive|sqflite|drift|isar` (file names only) — local storage usage
2. Grep for `http\.get|http\.post|HttpClient|Dio|chopper` (file names only) — network calls
3. Grep for `WebView|InAppWebView|WebViewWidget` (file names only) — WebView usage
4. Grep for `MethodChannel|EventChannel|BasicMessageChannel` (file names only) — platform channels
5. Grep for `dart:ffi|DynamicLibrary|Pointer<` (file names only) — FFI usage
6. Glob for `**/AndroidManifest.xml`, `**/Info.plist`, `**/build.gradle*`, `**/Podfile` — platform configs
7. Grep for `firebase|supabase|appwrite` in `pubspec.yaml` — backend service detection
8. Glob for `**/*.dart` in `lib/` — application source files

## Checks

### 1. Insecure Local Storage (Critical ~ Warning)

- **SharedPreferences for secrets** (Critical): `SharedPreferences` storing tokens, passwords, keys, secrets. Pattern: `SharedPreferences` usage near `token|password|secret|key|credential|session|jwt|auth`.
- **Unencrypted database** (Warning): `sqflite`/`drift`/`isar` without encryption. Check for `password:` or `encryptionKey:` parameter in database open calls.
- **flutter_secure_storage absence** (Warning): project handles auth tokens but `flutter_secure_storage` is not in `pubspec.yaml` dependencies.
- **Hive without encryption** (Warning): `Hive.openBox` without `encryptionCipher` parameter.

### 2. Network Security (Critical ~ Warning)

- **HTTP URLs** (Critical): hardcoded `http://` URLs (non-localhost) in source code. Pattern: `['"]http://(?!localhost|127\.0\.0\.1|10\.|192\.168\.)`.
- **Certificate pinning absence** (Info): no certificate pinning library (`dio_http2_adapter`, `http_certificate_pinning`, custom `SecurityContext`) in a project that makes API calls.
- **Disabled certificate validation** (Critical): `badCertificateCallback` returning `true` unconditionally, or `HttpOverrides` with permissive `createHttpClient`.
- **Sensitive data in URL parameters** (Warning): API calls with `token=`, `key=`, `password=` in query strings.

### 3. WebView Security (Critical ~ Warning)

- **JavaScript enabled without restriction** (Warning): `WebView` with `javascriptMode: JavascriptMode.unrestricted` or `InAppWebView` with `javaScriptEnabled: true` without URL validation.
- **Arbitrary URL loading** (Critical): WebView loading URLs from user input or deep links without validation.
- **JavaScript bridge exposure** (Warning): `addJavaScriptHandler` or `JavascriptChannel` exposing sensitive native functions.
- **File access enabled** (Warning): `allowFileAccess: true` or `allowFileAccessFromFileURLs: true`.

### 4. Authentication & Authorization (Critical ~ Warning)

- **Token in plain storage** (Critical): auth tokens stored via `SharedPreferences` instead of `FlutterSecureStorage`.
- **Missing token refresh** (Warning): API client with auth token but no refresh token logic (no `401` or `refresh` handling pattern found).
- **Biometric auth bypass** (Warning): `local_auth` usage without server-side verification. Check if biometric result only gates UI access vs actual API auth.
- **No session timeout** (Info): no idle timeout or token expiry handling detected.

### 5. Platform Channel Security (Warning)

- **Unvalidated method channel arguments** (Warning): `MethodChannel` handler that processes arguments without type checking or validation.
- **Sensitive data over channels** (Warning): platform channel calls transmitting tokens, passwords, or keys.
- **Missing error handling** (Info): `invokeMethod` calls without `PlatformException` catch blocks.

### 6. Deep Link & URL Scheme (Warning ~ Info)

- **Custom URL scheme hijacking** (Warning): custom scheme registered in `AndroidManifest.xml` (`<data android:scheme=.../>`) or `Info.plist` (`CFBundleURLSchemes`) without intent validation.
- **Deep link parameters used for auth** (Critical): deep link handlers that extract tokens or auth codes from URL parameters and use them without validation.
- **Universal links misconfiguration** (Info): `android:autoVerify="true"` without corresponding `assetlinks.json` check, or missing `apple-app-site-association`.

### 7. Build & Release Security (Warning ~ Info)

- **Debug mode indicators** (Warning): `kDebugMode` checks that leak debug functionality into release builds. Pattern: conditional logic that may not be stripped.
- **Obfuscation not configured** (Info): check `build.gradle` or CI scripts for `--obfuscate --split-debug-info` flags. If not found — Info.
- **ProGuard/R8 not configured** (Info): Android `build.gradle` missing `minifyEnabled true` for release builds.
- **Sensitive logging** (Warning): `print()`, `debugPrint()`, `log()`, `Logger` calls with sensitive patterns (`token`, `password`, `secret`, `key`, `credential`) in non-test files.
- **iOS ATS exceptions** (Warning): `Info.plist` containing `NSAppTransportSecurity` with `NSAllowsArbitraryLoads: true`.

### 8. Dart FFI Security (Warning)

If FFI usage is detected:
- **Unsafe pointer operations** (Warning): `Pointer.fromAddress`, `cast()` without bounds checking.
- **Memory leaks** (Warning): `malloc`/`calloc` allocations without corresponding `free`/`calloc.free`.
- **DynamicLibrary.open with user input** (Critical): library path derived from user-controlled input.

### 9. SQL Injection (Critical)

- **Raw SQL with string interpolation** (Critical): `rawQuery`, `rawInsert`, `rawUpdate`, `rawDelete` with string interpolation (`$variable` or `${expression}`) or concatenation.
- **Parameterized queries** (safe — do not report): `rawQuery('SELECT * FROM table WHERE id = ?', [id])`.

### 10. Sensitive Data Exposure (Warning ~ Info)

- **Clipboard exposure** (Warning): sensitive data copied to clipboard via `Clipboard.setData` with patterns matching tokens/passwords.
- **Screenshot protection absent** (Info): no `FlutterWindowManager` or `FLAG_SECURE` usage in an app handling sensitive data.
- **Error details exposed** (Warning): `catch` blocks that display raw error messages to users (`showDialog` or `SnackBar` with `error.toString()` or `stackTrace`).

### 11. Firebase Security (Warning)

If Firebase is detected in dependencies:
- **google-services.json / GoogleService-Info.plist committed** (Warning): check `git ls-files` for these files. Note: Firebase keys are designed to be public, but the files may contain project IDs that aid targeting.
- **Firebase options hardcoded** (Info): `FirebaseOptions` with hardcoded values in source (acceptable but should use `firebase_options.dart` generated by FlutterFire CLI).
- **Firestore rules not in repo** (Info): if using Firestore, check for `firestore.rules` file. If missing, note as "Firestore rules not found in repo — verify in Firebase Console".

### 12. Dependency Security (Info)

- Run `dart pub audit --json 2>/dev/null | head -100` if available. Parse and classify by severity.
- If unavailable, read `pubspec.lock` for major version checks. Note CVE cutoff limitation.
- Check for `dependency_overrides` in `pubspec.yaml` — Warning if present in non-dev context.
- Outdated SDK constraint: `sdk: '>=2.x.x'` when Dart 3+ is available — Info.

### 13. Code Generation Security (Info)

- **Generated files committed** (Info): `*.g.dart`, `*.freezed.dart` files tracked in git. These should typically be generated at build time.
- **JSON serialization without validation** (Warning): `fromJson` factory methods that don't validate input types/ranges for security-sensitive fields.

## Suppression

Follow suppression rules in CLAUDE.md.

## Output

Follow the common output format in CLAUDE.md.

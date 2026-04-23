# Riverpod — State Management Patterns

## Why It Matters

Riverpod is Flutter's most widely used "compile-safe" state management library, but the provider/ref system has subtle rules that, when violated, cause memory leaks, unexpected rebuilds, stale state, and test-prod divergence. Most issues don't surface in the type system — they show up as performance regressions or intermittent bugs.

Apply this reference when reviewing Flutter code. If the project doesn't use Riverpod, the *principles* (narrow scope, explicit lifecycle, exhaustive async handling) still apply; map them to whatever state library the project actually uses.

## Principles

- **`ref.watch` inside `build`, `ref.read` outside.** `read` in `build` silently misses updates; `watch` in callbacks causes needless rebuilds.
- **`autoDispose` by default.** A provider that outlives its consumer is a leak waiting to happen. Drop `autoDispose` only when the lifetime is deliberately long-lived.
- **One concern per provider.** A provider that mixes unrelated state causes unrelated consumers to rebuild together.
- **Handle `AsyncValue` exhaustively.** All three branches (`data`, `loading`, `error`) or an explicit `.maybeWhen` default.
- **Prefer the new `Notifier` / `AsyncNotifier` API** (riverpod ≥ 2.0). `StateNotifierProvider` is legacy.

## Checklist

### Ref Usage

- [ ] `ref.read` inside a `build` method — updates silently missed. Use `ref.watch`.
- [ ] `ref.watch` inside a callback (`onPressed`, `onTap`, `then`) — unnecessary rebuild. Use `ref.read`.
- [ ] `ref.listen` used where `ref.watch` would suffice (or vice versa) — `listen` fires side effects, `watch` rebuilds.
- [ ] `ref` stored as an instance field on a long-lived object — becomes invalid after its owning widget is disposed.
- [ ] `ref.read(provider.notifier)` called in `build` instead of once in `initState` or inline at call sites.

### Provider Lifecycle

- [ ] Widget-local state promoted to a global provider without a clear reason — prefer `StatefulWidget` or a provider scoped to the nearest `ProviderScope`.
- [ ] `autoDispose` missing on a provider consumed by a single screen that is routed away from — the state leaks until app restart.
- [ ] `keepAlive` (`ref.keepAlive()`) called without a corresponding invalidation path — provider pinned forever.
- [ ] Provider that holds a subscription (`StreamSubscription`, `WebSocketChannel`, `Timer`) without `ref.onDispose` cleanup.

### AsyncValue Handling

- [ ] `.when(data:, loading:, error:)` with one branch missing — behavior undefined for that state.
- [ ] `.value!` force-unwrap on `AsyncValue` — throws when loading or errored.
- [ ] `.maybeWhen` with no `orElse` — won't compile, but worth checking that `orElse` isn't just `() => SizedBox.shrink()` silently hiding errors.
- [ ] `AsyncError` branch that doesn't surface the error to the user or log it.
- [ ] Using `FutureProvider` when `AsyncNotifierProvider` would let the UI retry / mutate — one-shot future can't be refreshed cleanly.

### Notifier / StateNotifier

- [ ] New code using `StateNotifierProvider` (legacy) instead of `NotifierProvider` / `AsyncNotifierProvider`.
- [ ] `StateNotifier.state = ...` called from inside `build()` of a consumer — causes "setState during build" errors.
- [ ] `Notifier.build()` performing I/O directly instead of returning the initial state and deferring I/O to explicit methods.
- [ ] Emitting the same reference as new state (`state = state`) — does nothing; Riverpod checks `==`.
- [ ] Mutating `state` in place (e.g., `state.add(x)`) instead of assigning a new value — consumers don't rebuild.

### Family & Modifier Chains

- [ ] `.family` called with unbounded arg values (e.g., every row ID) — cache grows without bound unless combined with `autoDispose`.
- [ ] Modifier chain longer than `.autoDispose.family.future` — reconsider whether the provider is doing too much.
- [ ] `.family` with a non-hashable argument (custom class missing `==`/`hashCode`) — cache miss on every call, or worse, cache collisions.

### Composition / Dependencies

- [ ] Provider A `watch`es provider B, and provider B `watch`es A — circular dependency, runtime error.
- [ ] Provider chain deeper than 3 levels — fragile; consider collapsing intermediaries.
- [ ] `select` not used when watching a large provider — consumer rebuilds on any field change, not just the one it cares about.

### Testing

- [ ] Tests don't wrap the widget tree in `ProviderScope(overrides: [...])` — providers hit real APIs.
- [ ] `overrides` list duplicates the real wiring instead of overriding specific providers — maintenance burden.
- [ ] No test for the `AsyncError` branch — common source of production crashes.
- [ ] `ProviderContainer()` used directly in tests without `.dispose()` in `tearDown` — container leak.

### Migration / Legacy

- [ ] Mix of legacy `provider` package and Riverpod in the same codebase without a migration plan.
- [ ] `StateNotifierProvider` in new features when the codebase already uses `Notifier` elsewhere — inconsistency.
- [ ] `ChangeNotifierProvider` used for anything other than interop with legacy `ChangeNotifier` widgets.

## Good / Bad Examples

Bad — `ref.read` in `build`:
```dart
@override
Widget build(BuildContext context, WidgetRef ref) {
  final user = ref.read(userProvider);   // never rebuilds when user changes
  return Text(user.name);
}
```

Good — `ref.watch` in `build`:
```dart
@override
Widget build(BuildContext context, WidgetRef ref) {
  final user = ref.watch(userProvider);
  return Text(user.name);
}
```

Bad — force-unwrapping `AsyncValue`:
```dart
final user = ref.watch(userProvider).value!;   // crashes while loading
return Text(user.name);
```

Good — exhaustive `.when`:
```dart
return ref.watch(userProvider).when(
  data: (user) => Text(user.name),
  loading: () => const CircularProgressIndicator(),
  error: (e, st) => Text('Failed: $e'),
);
```

Bad — provider without `autoDispose`, pinned forever:
```dart
final searchResultsProvider = FutureProvider.family<List<Item>, String>((ref, query) async {
  return api.search(query);
});
// Every searched query stays in memory until app restart.
```

Good — `autoDispose.family`:
```dart
final searchResultsProvider =
  FutureProvider.autoDispose.family<List<Item>, String>((ref, query) async {
    return api.search(query);
  });
```

Bad — in-place mutation:
```dart
class TodosNotifier extends Notifier<List<Todo>> {
  @override
  List<Todo> build() => [];

  void add(Todo t) {
    state.add(t);   // consumers never rebuild
  }
}
```

Good — new list assignment:
```dart
void add(Todo t) {
  state = [...state, t];
}
```

Bad — no override in test:
```dart
testWidgets('shows user name', (tester) async {
  await tester.pumpWidget(const MyApp());   // hits real /users endpoint
});
```

Good — override fake provider:
```dart
testWidgets('shows user name', (tester) async {
  await tester.pumpWidget(
    ProviderScope(
      overrides: [userProvider.overrideWith((_) => const User(name: 'Test'))],
      child: const MyApp(),
    ),
  );
});
```

## Anti-patterns

- **God Provider** — One provider holding ten unrelated pieces of state. Consumers rebuild for changes they don't care about. Split by concern.
- **Ref Smuggling** — Storing `ref` on a service class or stream controller to call providers from outside the widget tree. Ref is tied to its owner's lifecycle; this creates dangling references.
- **Provider-of-a-Provider** — Deeply nested provider chains used as a workaround for poor composition. Usually indicates a missing domain model.
- **Modifier Soup** — `.autoDispose.family.future.select(...)` inside `build` — hard to read, hard to test. Split into intermediate providers.
- **Async Swallow** — `.maybeWhen(orElse: () => const SizedBox())` with no error path. Errors vanish silently; users see a blank screen.
- **Legacy Stacking** — `ChangeNotifier` + `StateNotifier` + `Notifier` in the same project because "we'll migrate later". Pick one and finish the migration.
- **Container-in-Test-Without-Teardown** — Using `ProviderContainer()` in unit tests but forgetting `tearDown(container.dispose)`. Tests pass individually, fail in suite runs due to leaked state.
